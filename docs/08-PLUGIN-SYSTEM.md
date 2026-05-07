# 第八章：插件系统与 XPath 解析

> 目标：理解 Kazumi 的核心业务逻辑 — 如何通过规则从网站采集番剧数据
> 配套源码：`lib/plugins/plugins.dart`、`assets/plugins/*.json`

---

## 1. 插件系统概述

Kazumi 的核心功能是从各种视频网站采集番剧信息。每个网站的 HTML 结构不同，插件系统通过 **XPath 选择器** 定义规则，告诉程序"在哪里找到番剧名称、链接、集数"。

### 类比理解

| 概念 | 类比 |
|------|------|
| Plugin | 爬虫规则配置文件 |
| XPath 选择器 | CSS 选择器 / jQuery 选择器 |
| searchList | "搜索结果列表在哪里" |
| searchName | "每个结果的标题在哪里" |
| searchResult | "每个结果的链接在哪里" |
| chapterRoads | "播放列表在哪里" |
| chapterResult | "每一集的链接在哪里" |

### 工作流程

```
用户搜索 "进击的巨人"
        │
        ▼
Plugin.queryBangumi("进击的巨人")
        │
        ▼
1. 构造搜索 URL: baseUrl + searchURL.replace("@keyword", "进击的巨人")
2. 发送 HTTP 请求获取 HTML
3. 用 XPath 选择器解析 HTML
4. 提取番剧名称和链接
5. 返回 List<SearchItem>
```

---

## 2. Plugin 类结构

`lib/plugins/plugins.dart`:

```dart
class Plugin {
  String name;          // 插件名称（如 "某某源"）
  String baseUrl;       // 网站根地址
  String searchURL;     // 搜索 URL 模板（@keyword 会被替换）
  String searchList;    // XPath: 搜索结果列表
  String searchName;    // XPath: 结果标题
  String searchResult;  // XPath: 结果链接
  String chapterRoads;  // XPath: 播放列表（线路）
  String chapterResult; // XPath: 集数链接
  bool useWebview;      // 是否用 WebView 解析视频地址
  bool usePost;         // 搜索是否用 POST 请求
  String userAgent;     // 自定义 UA
  String referer;       // 自定义 Referer
  // ...
}
```

### 从 JSON 创建

插件规则以 JSON 格式存储：

```json
{
  "api": "6",
  "type": "anime",
  "name": "示例源",
  "version": "1.0",
  "muliSources": true,
  "useWebview": true,
  "useNativePlayer": true,
  "usePost": false,
  "userAgent": "",
  "baseURL": "https://example.com",
  "searchURL": "https://example.com/search?q=@keyword",
  "searchList": "//div[@class='search-list']/div",
  "searchName": "//a[@class='title']/text()",
  "searchResult": "//a[@class='title']",
  "chapterRoads": "//div[@class='playlist']",
  "chapterResult": "//a[@class='episode']",
  "referer": ""
}
```

---

## 3. XPath 选择器

### 基础语法

XPath 是一种在 XML/HTML 文档中定位元素的语言。如果你用过 CSS 选择器或 jQuery，概念类似：

| XPath | CSS 选择器等价 | 含义 |
|-------|---------------|------|
| `//div` | `div` | 所有 div 元素 |
| `//div[@class='item']` | `div.item` | class 为 item 的 div |
| `//a/@href` | — | a 标签的 href 属性 |
| `//h2/text()` | — | h2 标签的文本内容 |
| `//div[@id='list']/a` | `#list a` | id 为 list 的 div 下的所有 a |

### 在插件中的使用

假设目标网站的搜索结果 HTML 是：
```html
<div class="search-results">
  <div class="item">
    <a href="/anime/123" class="title">进击的巨人</a>
    <span class="year">2013</span>
  </div>
  <div class="item">
    <a href="/anime/456" class="title">进击的巨人 第二季</a>
    <span class="year">2017</span>
  </div>
</div>
```

对应的 XPath 规则：
```
searchList:   //div[@class='search-results']/div[@class='item']
searchName:   //a[@class='title']/text()
searchResult: //a[@class='title']
```

解析过程：
1. `searchList` 找到所有搜索结果项（2 个 div.item）
2. 对每个结果项，`searchName` 提取标题文本
3. 对每个结果项，`searchResult` 提取链接（从 href 属性）

---

## 4. 核心方法解析

### queryBangumi — 搜索番剧

```dart
Future<PluginSearchResponse> queryBangumi(String keyword) async {
  // 1. 构造搜索 URL
  String queryURL = searchURL.replaceAll('@keyword', Uri.encodeQueryComponent(keyword));

  // 2. 发送 HTTP 请求
  var resp = await Request().get(queryURL, options: Options(headers: httpHeaders));

  // 3. 解析 HTML
  var htmlElement = parse(resp.data.toString()).documentElement!;

  // 4. 检测验证码（反爬）
  if (antiCrawlerConfig.enabled) {
    // 如果检测到验证码页面，抛出异常
    throw CaptchaRequiredException(name);
  }

  // 5. 用 XPath 提取搜索结果
  List<SearchItem> searchItems = [];
  htmlElement.queryXPath(searchList).nodes.forEach((element) {
    SearchItem searchItem = SearchItem(
      name: element.queryXPath(searchName).node!.text?.trim() ?? '',
      src: element.queryXPath(searchResult).node!.attributes['href'] ?? '',
    );
    searchItems.add(searchItem);
  });

  return PluginSearchResponse(pluginName: name, data: searchItems);
}
```

### querychapterRoads — 获取播放列表

```dart
Future<List<Road>> querychapterRoads(String url) async {
  // 1. 请求番剧详情页
  var resp = await Request().get(queryURL, options: Options(headers: httpHeaders));
  var htmlElement = parse(resp.data.toString()).documentElement!;

  // 2. 提取播放线路（可能有多条线路）
  htmlElement.queryXPath(chapterRoads).nodes.forEach((element) {
    List<String> chapterUrlList = [];
    List<String> chapterNameList = [];

    // 3. 提取每一集的链接和名称
    element.queryXPath(chapterResult).nodes.forEach((item) {
      chapterUrlList.add(item.node.attributes['href'] ?? '');
      chapterNameList.add(item.node.text ?? '');
    });

    // 4. 组装成 Road 对象
    Road road = Road(
      name: '播放列表$count',
      data: chapterUrlList,
      identifier: chapterNameList,
    );
    roadList.add(road);
  });

  return roadList;
}
```

---

## 5. 插件管理

### PluginsController

`lib/plugins/plugins_controller.dart` 管理所有已安装的插件：

```dart
abstract class _PluginsController with Store {
  @observable
  ObservableList<Plugin> pluginList = ObservableList.of([]);

  // 从本地存储加载插件
  void loadPlugins() { ... }

  // 从 JSON 字符串导入插件
  void importPlugin(String jsonString) { ... }

  // 删除插件
  void deletePlugin(Plugin plugin) { ... }
}
```

### 内置插件

`assets/plugins/*.json` 包含预装的插件规则文件，应用首次启动时会加载。

### 自定义插件

用户可以通过导入 JSON 文件添加自定义插件规则，存储在 Hive 中。

---

## 6. 反爬机制

### AntiCrawlerConfig

部分网站有验证码保护，插件支持配置反爬检测：

```dart
class AntiCrawlerConfig {
  bool enabled;
  String captchaImage;   // XPath: 验证码图片位置
  String captchaButton;  // XPath: 验证码提交按钮
  String captchaInput;   // XPath: 验证码输入框
}
```

当检测到验证码时，应用会弹出 WebView 让用户手动完成验证。

### Cookie 管理

`lib/plugins/plugin_cookie_manager.dart` — 每个插件独立管理 Cookie：

```dart
class PluginCookieManager {
  // 为每个插件维护独立的 Cookie
  static Map<String, String> getCookies(String pluginName) { ... }
  static void setCookies(String pluginName, String cookies) { ... }
}
```

---

## 7. 视频源解析

搜索到番剧后，获取实际视频播放地址的流程：

```
用户选择某一集
      │
      ▼
Plugin.querychapterRoads(url)  → 获取集数列表
      │
      ▼
用户点击某一集
      │
      ▼
WebView 加载该集的页面 URL
      │
      ▼
WebView 拦截视频请求（.m3u8/.mp4）
      │
      ▼
将视频 URL 传给 media_kit 播放
```

大部分视频源使用 WebView 解析，因为视频地址通常由 JavaScript 动态生成，无法通过简单的 HTTP 请求获取。

---

## 8. 数据流总结

```
Plugin JSON 规则
      │
      ▼
Plugin 对象（内存中）
      │
      ├── queryBangumi(keyword)
      │         │
      │         ▼
      │   HTTP GET/POST → HTML → XPath 解析 → List<SearchItem>
      │
      └── querychapterRoads(url)
                │
                ▼
          HTTP GET → HTML → XPath 解析 → List<Road>
                                              │
                                              ▼
                                    WebView 解析视频地址
                                              │
                                              ▼
                                    media_kit 播放视频
```

---

## 9. 编写插件规则的要点

1. **只支持 `//` 开头的 XPath** — 必须是绝对路径选择器
2. **searchList 选中的是列表项** — 每个节点代表一个搜索结果
3. **searchName/searchResult 是相对于列表项的** — 在每个列表项内部查找
4. **chapterRoads 选中的是播放线路容器** — 可能有多个线路
5. **chapterResult 是相对于线路容器的** — 在每个线路内查找集数

---

## 练习

1. 打开 `assets/plugins/` 目录下的任意一个 JSON 文件，识别各个 XPath 字段的含义
2. 如果一个网站的搜索结果结构是：
   ```html
   <ul id="results">
     <li><a href="/show/1">番剧A</a></li>
     <li><a href="/show/2">番剧B</a></li>
   </ul>
   ```
   写出对应的 searchList、searchName、searchResult

<details>
<summary>答案</summary>

2. XPath 规则：
   - searchList: `//ul[@id='results']/li`
   - searchName: `//a/text()`
   - searchResult: `//a`

</details>

---

下一章：[UI 组件与主题系统](./09-UI-COMPONENTS.md)
