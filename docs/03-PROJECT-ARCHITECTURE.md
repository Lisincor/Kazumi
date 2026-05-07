# 第三章：项目架构与数据流

> 目标：理解 Kazumi 的整体分层架构和数据流向，建立全局视角
> 配套源码：`lib/pages/index_module.dart`、`lib/app_module.dart`

---

## 1. 架构总览

### 对照 Spring Boot 分层

```
Spring Boot                          Kazumi (Flutter)
─────────────────────────────────────────────────────────
Controller (@RestController)    →    Page (*_page.dart)
Service (@Service)              →    Controller (*_controller.dart)
Repository (@Repository)        →    Repository (repositories/)
Entity (@Entity)                →    Module (modules/)
application.yml                 →    GStorage (Hive Box)
Spring IoC Container            →    Flutter Modular
```

### 分层图

```
┌─────────────────────────────────────────────────────────┐
│  UI 层 (lib/pages/*_page.dart)                          │
│  职责：构建界面，响应用户交互                              │
│  技术：Flutter Widget + Observer (MobX)                  │
└────────────────────────┬────────────────────────────────┘
                         │ 调用 Controller 方法
┌────────────────────────▼────────────────────────────────┐
│  状态层 (lib/pages/*_controller.dart)                    │
│  职责：管理页面状态，协调业务逻辑                          │
│  技术：MobX Store (@observable, @action)                 │
└────────────────────────┬────────────────────────────────┘
                         │ 调用 Repository / Request
┌────────────────────────▼────────────────────────────────┐
│  数据层                                                  │
│  ┌─────────────────┐  ┌─────────────────┐               │
│  │ Repository       │  │ Request (Dio)   │               │
│  │ (本地数据 CRUD)  │  │ (远程 API 调用) │               │
│  └────────┬────────┘  └────────┬────────┘               │
│           │                     │                        │
│  ┌────────▼────────┐  ┌────────▼────────┐               │
│  │ GStorage (Hive) │  │ 外部 API        │               │
│  │ 本地持久化       │  │ Bangumi/弹幕等  │               │
│  └─────────────────┘  └─────────────────┘               │
└─────────────────────────────────────────────────────────┘
```

---

## 2. 目录结构详解

```
lib/
├── main.dart                  # 应用入口（≈ Application.java）
├── app_module.dart            # 根路由模块（≈ Spring Boot 自动配置）
├── app_widget.dart            # 根 Widget（主题、生命周期）
│
├── pages/                     # 页面层（每个子目录 = 一个功能模块）
│   ├── popular/               #   推荐页
│   ├── timeline/              #   时间线/新番表
│   ├── collect/               #   收藏管理
│   ├── my/                    #   个人中心
│   ├── search/                #   搜索
│   ├── info/                  #   番剧详情
│   ├── video/                 #   视频源解析
│   ├── player/                #   视频播放器
│   ├── history/               #   观看历史
│   ├── download/              #   下载管理
│   ├── settings/              #   设置
│   ├── plugin_editor/         #   插件编辑器
│   └── webdav_editor/         #   WebDAV 配置
│
├── modules/                   # 数据模型层（≈ Entity/DTO）
│   ├── bangumi/               #   番剧数据模型
│   ├── collect/               #   收藏数据模型
│   ├── history/               #   历史记录模型
│   ├── download/              #   下载记录模型
│   ├── danmaku/               #   弹幕数据模型
│   └── search/                #   搜索结果模型
│
├── repositories/              # 数据仓库层（≈ @Repository）
│   ├── collect_repository.dart
│   ├── collect_crud_repository.dart
│   ├── history_repository.dart
│   ├── download_repository.dart
│   └── search_history_repository.dart
│
├── request/                   # 网络请求层（≈ Feign Client）
│   ├── request.dart           #   Dio 单例配置
│   ├── api.dart               #   API 地址常量
│   ├── bangumi.dart           #   Bangumi API
│   ├── damaku.dart            #   弹幕 API
│   └── interceptor.dart       #   请求拦截器
│
├── plugins/                   # 插件系统（规则解析引擎）
│   └── plugins.dart           #   XPath 选择器解析
│
├── bean/                      # 可复用 UI 组件
│   ├── appbar/                #   自定义 AppBar
│   ├── card/                  #   卡片组件
│   ├── dialog/                #   对话框
│   ├── settings/              #   主题 Provider
│   └── widget/                #   通用 Widget
│
├── utils/                     # 工具类
│   ├── storage.dart           #   Hive 存储封装
│   ├── download_manager.dart  #   下载管理器
│   ├── proxy_manager.dart     #   代理管理
│   └── logger.dart            #   日志工具
│
└── shaders/                   # GPU 着色器（Anime4K 超分）
```

---

## 3. 数据流实例：用户搜索番剧

以"用户在搜索页输入关键词并查看结果"为例，追踪完整数据流：

```
用户输入 "进击的巨人"
        │
        ▼
┌─ SearchPage (UI) ─────────────────────────────────┐
│  TextField → onSubmitted → controller.search()     │
└────────────────────────┬──────────────────────────┘
                         │
                         ▼
┌─ SearchController (状态) ─────────────────────────┐
│  @action search(keyword) {                         │
│    isLoading = true;                               │
│    results = await plugin.queryBangumi(keyword);   │
│    isLoading = false;                              │
│  }                                                 │
└────────────────────────┬──────────────────────────┘
                         │
                         ▼
┌─ Plugin (规则引擎) ───────────────────────────────┐
│  1. 用 Dio 请求目标网站                            │
│  2. 用 XPath 选择器解析 HTML                       │
│  3. 提取番剧名称、链接、封面                        │
│  4. 返回 List<SearchItem>                          │
└────────────────────────┬──────────────────────────┘
                         │
                         ▼
┌─ SearchPage (UI 更新) ────────────────────────────┐
│  Observer 检测到 results 变化                       │
│  → 自动重建 ListView 显示搜索结果                   │
└───────────────────────────────────────────────────┘
```

---

## 4. 依赖注入：IndexModule

`lib/pages/index_module.dart` 是整个应用的 DI 配置中心，类似 Spring 的 `@Configuration` 类：

```dart
class IndexModule extends Module {
  @override
  void binds(i) {
    // Repository 层（≈ @Bean ICollectRepository）
    i.addSingleton<ICollectRepository>(CollectRepository.new);
    i.addSingleton<IHistoryRepository>(HistoryRepository.new);
    i.addSingleton<IDownloadRepository>(DownloadRepository.new);

    // Controller 层（≈ @Bean PopularController）
    i.addSingleton(PopularController.new);
    i.addSingleton(VideoPageController.new);
    i.addSingleton(CollectController.new);
    i.addSingleton(HistoryController.new);
  }
}
```

对照 Spring Boot：
```java
@Configuration
public class AppConfig {
    @Bean
    public ICollectRepository collectRepository() {
        return new CollectRepository();
    }

    @Bean
    public PopularController popularController() {
        return new PopularController();
    }
}
```

---

## 5. 路由结构

```
/                          → InitPage（启动页/闪屏）
/tab                       → IndexPage（主页，底部导航栏）
  /tab/popular             →   推荐页
  /tab/timeline            →   时间线
  /tab/collect             →   收藏
  /tab/my                  →   我的
/video                     → 视频播放页
/info                      → 番剧详情页
/settings                  → 设置页
  /settings/danmaku        →   弹幕设置
  /settings/theme          →   主题设置
  /settings/player         →   播放器设置
/search                    → 搜索页
```

对照 Spring MVC：
- `/tab/popular` ≈ `@GetMapping("/popular")`
- Module ≈ `@Controller` 类
- Route ≈ `@RequestMapping` 方法

---

## 6. 启动流程

`lib/main.dart` 的启动顺序：

```dart
void main() async {
  // 1. Flutter 引擎初始化（≈ Spring Boot 启动 Tomcat）
  WidgetsFlutterBinding.ensureInitialized();

  // 2. 视频库初始化
  MediaKit.ensureInitialized();

  // 3. 系统 UI 配置（状态栏透明等）
  SystemChrome.setEnabledSystemUIMode(SystemUiMode.edgeToEdge);

  // 4. 本地数据库初始化（≈ DataSource 初始化）
  await Hive.initFlutter(hivePath);
  await GStorage.init();

  // 5. 桌面端窗口配置
  if (Utils.isDesktop()) {
    await windowManager.ensureInitialized();
  }

  // 6. HTTP 客户端初始化（≈ RestTemplate Bean）
  Request();
  await Request.setCookie();

  // 7. 代理配置
  ProxyManager.applyProxy();

  // 8. 启动应用（≈ SpringApplication.run()）
  runApp(
    ChangeNotifierProvider(
      create: (_) => ThemeProvider(),
      child: ModularApp(
        module: AppModule(),
        child: const AppWidget(),
      ),
    ),
  );
}
```

---

## 7. 模块间通信

### Controller 之间的调用

```dart
// 在 VideoController 中获取 CollectController
class _VideoPageController with Store {
  void addToCollection(BangumiItem item) {
    final collectController = Modular.get<CollectController>();
    collectController.addCollect(item);
  }
}
```

### 页面间传参

```dart
// 跳转并传参
Modular.to.pushNamed('/info', arguments: bangumiItem);

// 接收参数
final item = Modular.args.data as BangumiItem;
```

### 全局事件（通过 Observable）

MobX 的 Observable 天然支持跨组件通信：
- Controller A 修改了 `collectibles` 列表
- 任何 Observer 监听了这个列表的 Widget 都会自动更新

---

## 8. 关键设计决策

| 决策 | 选择 | 原因 |
|------|------|------|
| 状态管理 | MobX | 代码量少，响应式精准更新 |
| 路由/DI | Flutter Modular | 模块化组织，支持懒加载 |
| 本地存储 | Hive | 轻量、快速、无需 SQL |
| 网络请求 | Dio | 拦截器、Cookie、代理支持完善 |
| 主题管理 | Provider | 简单场景用 Provider 足够 |
| 视频播放 | media_kit | 跨平台，支持硬件加速 |

---

## 练习

1. 打开 `lib/pages/index_module.dart`，数一数注册了多少个 Repository 和 Controller
2. 从 `lib/pages/collect/collect_controller.dart` 出发，追踪 `addCollect` 方法的完整调用链
3. 画出从"用户点击收藏按钮"到"数据保存到本地"的数据流图

<details>
<summary>答案提示</summary>

1. 5 个 Repository + 1 个 DownloadManager，9 个 Controller
2. `addCollect` → `_collectCrudRepository.addCollectible` → Hive Box 写入
3. Page(点击) → Controller(addCollect) → Repository(addCollectible) → GStorage(Hive put)

</details>

---

下一章：[路由与依赖注入](./04-MODULAR-ROUTING.md)
