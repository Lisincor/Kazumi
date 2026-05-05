# 第六课：插件系统

> 📖 学习目标：理解插件的工作原理，学会阅读和编写插件配置，了解 XPath 选择器
> ⏱️ 预计时间：2-3 小时
> 📁 配套源码：`lib/plugins/plugins.dart`、`assets/plugins/AGE.json`

---

## 目录

1. [插件系统概述](#1-插件系统概述)
2. [插件配置结构](#2-插件配置结构)
3. [XPath 选择器](#3-xpath-选择器)
4. [Plugin 类解析](#4-plugin-类解析)
5. [搜索流程](#5-搜索流程)
6. [剧集获取流程](#6-剧集获取流程)
7. [反爬虫配置](#7-反爬虫配置)
8. [插件管理](#8-插件管理)

---

## 1. 插件系统概述

### 1.1 什么是插件？

插件是一种**规则配置文件**，定义了如何从某个网站抓取番剧信息。

```
用户搜索"进击的巨人"
       ↓
系统遍历所有启用的插件
       ↓
Plugin A 使用 XPath 抓取 A 网站
Plugin B 使用 XPath 抓取 B 网站
       ↓
合并所有网站的搜索结果
       ↓
用户选择想要观看的来源
```

### 1.2 为什么用插件？

| 优点 | 说明 |
|------|------|
| 解耦 | 不用改代码，只改配置 |
| 可扩展 | 用户可以添加自己的规则 |
| 灵活 | 每个网站用不同的选择器 |
| 共享 | 用户可以导出/导入规则 |

### 1.3 项目中的插件文件

```
assets/plugins/
├── AGE.json      → AGE 动漫
├── DM84.json     → 第 84 期
├── aafun.json   → AA 资源
└── ...
```

📁 AGE 插件配置示例：`assets/plugins/AGE.json`

```json
{
  "api": "1",
  "type": "anime",
  "name": "AGE",
  "version": "1.5",
  "muliSources": true,
  "useWebview": true,
  "useNativePlayer": true,
  "userAgent": "",
  "baseURL": "https://www.agedm.io/",
  "searchURL": "https://www.agedm.io/search?query=@keyword",
  "searchList": "//div[2]/div/section/div/div/div/div",
  "searchName": "//div/div[2]/h5/a",
  "searchResult": "//div/div[2]/h5/a",
  "chapterRoads": "//div[2]/div/section/div/div[2]/div[2]/div[2]/div",
  "chapterResult": "//ul/li/a"
}
```

---

## 2. 插件配置结构

### 2.1 字段说明

| 字段 | 类型 | 说明 | 示例 |
|------|------|------|------|
| `api` | string | 插件 API 版本 | "1" |
| `type` | string | 资源类型 | "anime" |
| `name` | string | 插件名称 | "AGE" |
| `version` | string | 插件版本 | "1.5" |
| `muliSources` | bool | 是否支持多源 | true |
| `useWebview` | bool | 是否使用 WebView 播放 | true |
| `useNativePlayer` | bool | 是否使用原生播放器 | true |
| `userAgent` | string | 自定义 User-Agent | "" |
| `baseURL` | string | 网站基础 URL | "https://..." |
| `searchURL` | string | 搜索页面 URL（含 @keyword 占位） | "https://...search?q=@keyword" |
| `searchList` | string | 搜索结果列表的 XPath | "//div[@class='item']" |
| `searchName` | string | 番剧名称的 XPath | "//div[@class='title']" |
| `searchResult` | string | 结果链接的 XPath | "//div[@class='link']" |
| `chapterRoads` | string | 剧集列表的 XPath | "//div[@class='episode']" |
| `chapterResult` | string | 剧集链接的 XPath | "//a[@class='ep']" |

### 2.2 字段详解

#### baseURL - 基础地址

用于拼接相对路径：
```json
"baseURL": "https://www.agedm.io/"
```
如果搜索结果返回 `/anime/123`，实际访问地址为 `https://www.agedm.io/anime/123`

#### searchURL - 搜索地址

`@keyword` 是搜索关键词的占位符：
```json
"searchURL": "https://www.agedm.io/search?query=@keyword"
```
搜索"进击的巨人"时，实际请求 `https://www.agedm.io/search?query=%E8%BF%9B%E5%87%BB%E7%9A%84%E5%B7%A8%E4%BA%BA`

#### XPath 选择器

```
searchList   → 包含所有搜索结果的容器元素
searchName   → 番剧标题（相对路径，从 searchList 开始）
searchResult → 番剧详情页链接（相对路径）
```

---

## 3. XPath 选择器

### 3.1 什么是 XPath？

XPath 是一种在 XML/HTML 文档中查找信息的语言。类比：

| 概念 | 类比 |
|------|------|
| HTML 文档 | 一棵树 |
| XPath | 描述如何从树根走到某个树枝的路径 |
| `//` | 从任何位置开始查找 |

### 3.2 基本语法

```xpath
// 任意位置的 div 元素
//div

// id 为 container 的元素
//*[@id='container']

// class 包含 item 的 div
//div[@class='item']

// 第一个 div
//div[1]

// div 下的直接子元素 p
//div/p

// div 下的任意后代元素 span
//div//span

// div 下 class 为 title 的元素
//div//*[@class='title']

// a 标签的 href 属性
//a/@href

// 获取文本内容
//div/text()
```

### 3.3 实际例子

假设 HTML 结构如下：

```html
<div class="search-results">
  <div class="item">
    <div class="cover">
      <img src="cover.jpg">
    </div>
    <div class="info">
      <h3><a href="/anime/1">进击的巨人</a></h3>
      <p>描述...</p>
    </div>
  </div>
  <div class="item">
    <div class="info">
      <h3><a href="/anime/2">进击的巨人 S2</a></h3>
    </div>
  </div>
</div>
```

对应的 XPath：
```json
{
  "searchList": "//div[@class='search-results']/div[@class='item']",
  "searchName": "//div[@class='info']/h3/a",
  "searchResult": "//div[@class='info']/h3/a/@href"
}
```

### 3.4 如何获取 XPath

1. **Chrome 开发者工具**
   - 右键点击元素 → Copy → Copy XPath
   - 注意：Chrome 复制的是绝对路径，可能需要简化

2. **在线 XPath 测试**
   - 使用 https://xpather.com/ 测试

3. **浏览器插件**
   - XPath Helper (Chrome 扩展)

### 3.5 简化原则

```xpath
# Chrome 复制（绝对路径，不好用）
/html/body/div[2]/div/section/div/div/div/div[1]/div[1]/div[2]/h5/a

# 简化后（推荐）
//div[@class='item']//h5/a
```

---

## 4. Plugin 类解析

📁 源码：`lib/plugins/plugins.dart`

### 4.1 类结构

```dart
class Plugin {
  String api;              // API 版本
  String type;             // 类型
  String name;             // 名称
  String version;          // 版本
  bool muliSources;        // 多源支持
  bool useWebview;         // WebView 播放
  bool useNativePlayer;   // 原生播放器
  bool usePost;            // POST 请求
  bool useLegacyParser;    // 旧解析器
  bool adBlocker;          // 广告拦截
  String userAgent;        // UA
  String baseUrl;          // 基础 URL
  String searchURL;        // 搜索 URL
  String searchList;       // 搜索列表 XPath
  String searchName;       // 名称 XPath
  String searchResult;     // 结果 XPath
  String chapterRoads;     // 剧集列表 XPath
  String chapterResult;    // 剧集 XPath
  String referer;          // Referer
  AntiCrawlerConfig antiCrawlerConfig;  // 反爬配置
}
```

### 4.2 JSON 序列化

📁 源码：`lib/plugins/plugins.dart:88-113`

```dart
factory Plugin.fromJson(Map<String, dynamic> json) {
  return Plugin(
    api: json['api'],
    type: json['type'],
    name: json['name'],
    // ...
    antiCrawlerConfig: json['antiCrawlerConfig'] != null
        ? AntiCrawlerConfig.fromJson(json['antiCrawlerConfig'])
        : AntiCrawlerConfig.empty(),
  );
}

Map<String, dynamic> toJson() {
  final Map<String, dynamic> data = {};
  data['api'] = api;
  data['baseURL'] = baseUrl;
  // ...
  return data;
}
```

### 4.3 工厂模板

📁 源码：`lib/plugins/plugins.dart:115-137`

```dart
factory Plugin.fromTemplate() {
  return Plugin(
    api: Api.apiLevel.toString(),
    type: 'anime',
    name: '',
    version: '',
    muliSources: true,
    useWebview: true,
    // ... 默认值
  );
}
```

---

## 5. 搜索流程

📁 源码：`lib/plugins/plugins.dart:164-244`

### 5.1 流程图

```
queryBangumi(keyword)
    ↓
替换 @keyword 为编码后的关键词
    ↓
构造请求头（Referer, Cookie, UA）
    ↓
发送 GET/POST 请求
    ↓
解析 HTML
    ↓
使用 XPath 获取搜索列表
    ↓
遍历每个结果，使用 XPath 获取名称和链接
    ↓
返回 PluginSearchResponse
```

### 5.2 代码解析

```dart
Future<PluginSearchResponse> queryBangumi(String keyword,
    {bool shouldRethrow = false}) async {
  try {
    // 1. 构造 URL
    String queryURL = searchURL.replaceAll(
      '@keyword',
      Uri.encodeQueryComponent(keyword),  // URL 编码
    );

    // 2. 构造请求头
    final String cookieHeader = await _cookieHeaderFor(queryURL);
    var httpHeaders = {
      'referer': '$baseUrl/',
      'Accept-Language': Utils.getRandomAcceptedLanguage(),
      'Connection': 'keep-alive',
      if (cookieHeader.isNotEmpty) 'Cookie': cookieHeader,
    };

    // 3. 发送请求
    if (usePost) {
      // POST 请求
      resp = await Request().post(
        postUri.toString(),
        options: Options(headers: httpHeaders),
        data: queryParams,
      );
    } else {
      // GET 请求
      resp = await Request().get(
        queryURL,
        options: Options(headers: httpHeaders),
      );
    }

    // 4. 解析 HTML
    var htmlString = resp.data.toString();
    var htmlElement = parse(htmlString).documentElement!;

    // 5. 使用 XPath 获取结果
    htmlElement.queryXPath(searchList).nodes.forEach((element) {
      try {
        SearchItem searchItem = SearchItem(
          name: element.queryXPath(searchName).node!.text?.trim() ?? '',
          src: element.queryXPath(searchResult).node!.attributes['href'] ?? '',
        );
        searchItems.add(searchItem);
      } catch (_) {}
    });

    // 6. 空结果处理
    if (searchItems.isEmpty) throw NoResultException(name);
    return PluginSearchResponse(pluginName: name, data: searchItems);

  } on NoResultException {
    rethrow;
  } catch (e, st) {
    KazumiLogger().w('Plugin: $name search failed', error: e, stackTrace: st);
    if (shouldRethrow) throw SearchErrorException(name, cause: e);
    return PluginSearchResponse(pluginName: name, data: []);
  }
}
```

### 5.3 XPath 执行

```dart
// 获取节点列表
var nodes = htmlElement.queryXPath(searchList).nodes;

// 遍历每个节点
nodes.forEach((element) {
  // 在当前节点下执行相对路径 XPath
  var name = element.queryXPath(searchName).node!.text;
  var href = element.queryXPath(searchResult).node!.attributes['href'];
});
```

### 5.4 相对路径

```json
{
  "searchList": "//div[@class='item']",
  "searchName": ".//h3/a",
  "searchResult": ".//h3/a/@href"
}
```

注意：`searchList` 用 `//` 表示从根开始查找，`searchName` 和 `searchResult` 用 `.//` 表示从当前节点开始查找。

---

## 6. 剧集获取流程

📁 源码：`lib/plugins/plugins.dart:247-290`

### 6.1 流程图

```
querychapterRoads(url)
    ↓
验证 URL（添加 https 或 baseUrl）
    ↓
发送 GET 请求
    ↓
解析 HTML
    ↓
使用 XPath 获取播放列表
    ↓
遍历每个列表，使用 XPath 获取剧集链接和名称
    ↓
返回 List<Road>
```

### 6.2 代码解析

```dart
Future<List<Road>> querychapterRoads(String url, {CancelToken? cancelToken}) async {
  List<Road> roadList = [];

  // 1. URL 处理
  if (!url.contains('https')) {
    url = url.replaceAll('http', 'https');
  }
  String queryURL = url.contains(baseUrl) ? url : baseUrl + url;

  // 2. 发送请求
  var httpHeaders = {
    'referer': '$baseUrl/',
    'Accept-Language': Utils.getRandomAcceptedLanguage(),
    'Connection': 'keep-alive',
  };
  var resp = await Request().get(
    queryURL,
    options: Options(headers: httpHeaders),
    cancelToken: cancelToken,
  );

  // 3. 解析 HTML
  var htmlElement = parse(resp.data.toString()).documentElement!;

  // 4. 获取播放列表
  int count = 1;
  htmlElement.queryXPath(chapterRoads).nodes.forEach((element) {
    try {
      List<String> chapterUrlList = [];
      List<String> chapterNameList = [];

      // 5. 获取每个剧集
      element.queryXPath(chapterResult).nodes.forEach((item) {
        String itemUrl = item.node.attributes['href'] ?? '';
        String itemName = item.node.text ?? '';
        chapterUrlList.add(itemUrl);
        chapterNameList.add(itemName.replaceAll(RegExp(r'\s+'), ''));  // 去除空白
      });

      // 6. 创建 Road 对象
      if (chapterUrlList.isNotEmpty && chapterNameList.isNotEmpty) {
        Road road = Road(
          name: '播放列表$count',
          data: chapterUrlList,
          identifier: chapterNameList,
        );
        roadList.add(road);
        count++;
      }
    } catch (_) {}
  });

  return roadList;
}
```

### 6.3 Road 数据结构

📁 源码：`lib/modules/roads/road_module.dart`

```dart
class Road {
  String name;              // 播放列表名称（如"第1季"）
  List<String> data;        // 剧集 URL 列表
  List<String> identifier;  // 剧集名称列表
}
```

---

## 7. 反爬虫配置

📁 源码：`lib/plugins/anti_crawler_config.dart`

### 7.1 CaptchaRequiredException

当检测到验证码时抛出：

```dart
if (antiCrawlerConfig.enabled) {
  final List<String> detectionXpaths = [
    antiCrawlerConfig.captchaImage,
    antiCrawlerConfig.captchaButton,
  ].where((x) => x.isNotEmpty).toList();

  final bool captchaDetected = detectionXpaths.any(
    (xpath) => htmlElement.queryXPath(xpath).node != null,
  );

  if (captchaDetected) {
    throw CaptchaRequiredException(name);
  }
}
```

### 7.2 Cookie 管理

📁 源码：`lib/plugins/plugin_cookie_manager.dart`

```dart
Future<String> _cookieHeaderFor(String url) async {
  if (!PluginCookieManager.instance.hasCookies(name)) return '';
  final cookies = await PluginCookieManager.instance.getJar(name).loadForRequest(uri);
  if (cookies.isEmpty) return '';
  return cookies.map((c) => '${c.name}=${c.value}').join('; ');
}
```

---

## 8. 插件管理

📁 源码：`lib/plugins/plugins_controller.dart`

### 8.1 加载插件

```dart
Future<void> loadAllPlugins() async {
  pluginList.clear();

  // 从 v2 目录读取 plugins.json
  final pluginsFile = File('${newPluginDirectory!.path}/$pluginsFileName');
  if (await pluginsFile.exists()) {
    final jsonString = await pluginsFile.readAsString();
    pluginList.addAll(getPluginListFromJson(jsonString));
  } else {
    // 从旧版分离文件迁移
    var jsonFiles = await getPluginFiles();
    for (var filePath in jsonFiles) {
      final jsonString = await File(filePath).readAsString();
      final plugin = Plugin.fromJson(jsonDecode(jsonString));
      pluginList.add(plugin);
    }
    savePlugins();
  }
}
```

### 8.2 保存插件

```dart
Future<void> savePlugins() async {
  final jsonData = jsonEncode(pluginListToJson());
  final pluginsFile = File('${newPluginDirectory!.path}/$pluginsFileName');
  await pluginsFile.writeAsString(jsonData);
}
```

### 8.3 更新插件

```dart
Future<int> tryUpdatePlugin(Plugin plugin) async {
  var pluginHTTPItem = await queryPluginHTTP(plugin.name);
  if (pluginHTTPItem != null) {
    if (int.parse(pluginHTTPItem.api) > Api.apiLevel) {
      return 1;  // API 版本不兼容
    }
    updatePlugin(pluginHTTPItem);  // 更新规则
    return 0;  // 成功
  }
  return 2;  // 未找到
}
```

---

## 小练习

### 练习 1：分析插件配置

阅读 `assets/plugins/AGE.json`，回答：

1. 这个插件用于哪个网站？
2. 搜索关键词时，会请求什么 URL？
3. 如何获取搜索结果列表？

<details>
<summary>答案</summary>

1. 网站：https://www.agedm.io/
2. 搜索 URL：`https://www.agedm.io/search?query=@keyword`
3. 搜索列表 XPath：`//div[2]/div/section/div/div/div/div`

</details>

### 练习 2：编写简单插件

假设你要为网站 `https://example.com` 编写插件，已知：
- 搜索 URL：`https://example.com/search?q=@keyword`
- 搜索结果 HTML：
```html
<div class="result">
  <a href="/anime/1">番剧名</a>
</div>
```

写出插件配置：

<details>
<summary>答案</summary>

```json
{
  "api": "1",
  "type": "anime",
  "name": "Example",
  "version": "1.0",
  "muliSources": true,
  "useWebview": true,
  "useNativePlayer": true,
  "userAgent": "",
  "baseURL": "https://example.com",
  "searchURL": "https://example.com/search?q=@keyword",
  "searchList": "//div[@class='result']",
  "searchName": ".",
  "searchResult": "./a/@href",
  "chapterRoads": "",
  "chapterResult": ""
}
```

</details>

### 练习 3：理解 XPath

给定以下 HTML，写出对应的 XPath：

```html
<div class="container">
  <div class="item">
    <a href="/1">第一集</a>
    <a href="/2">第二集</a>
  </div>
  <div class="item">
    <a href="/3">第三集</a>
  </div>
</div>
```

1. 获取所有 item
2. 获取第一个 item 的所有链接
3. 获取第三个链接的文字

<details>
<summary>答案</summary>

1. `//div[@class='item']`
2. `//div[@class='item'][1]/a`
3. `//div[@class='item']/a[3]/text()`

</details>

---

## 下一课预告

下一课我们将学习 [Flutter Modular 路由](./07-ROUTING.md)，了解页面是如何组织和跳转的。