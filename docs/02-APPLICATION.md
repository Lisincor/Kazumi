# 第二课：应用启动与初始化

> 📖 学习目标：理解 Flutter 应用是如何启动的，各个模块是如何被初始化的
> ⏱️ 预计时间：1-2 小时
> 📁 配套源码：`lib/main.dart`、`lib/app_widget.dart`、`lib/app_module.dart`

---

## 目录

1. [应用启动流程](#1-应用启动流程)
2. [main 函数解析](#2-main-函数解析)
3. [模块初始化](#3-模块初始化)
4. [应用生命周期](#4-应用生命周期)
5. [窗口与托盘管理](#5-窗口与托盘管理)
6. [主题系统](#6-主题系统)

---

## 1. 应用启动流程

### 1.1 流程概览

```
用户点击图标
    ↓
系统加载 Flutter 引擎
    ↓
执行 main() 函数
    ↓
初始化核心组件 (Hive, MediaKit, WindowManager)
    ↓
创建根 Widget
    ↓
应用主界面显示
```

### 1.2 代码流程图

```
main()
├── MediaKit.ensureInitialized()        媒体库初始化
├── SystemChrome 设置                    系统UI配置
├── Utils.checkWebViewFeatureSupport()   WebView检测
├── Hive.initFlutter()                   存储初始化
├── GStorage.init()                     数据存储初始化
├── windowManager 初始化                 窗口管理
├── Request 初始化                       网络请求
├── ProxyManager 初始化                   代理设置
└── runApp()                            启动应用
    └── ModularApp                      模块化应用
        └── AppWidget                   根组件
```

---

## 2. main 函数解析

📁 完整源码：`lib/main.dart:1-102`

### 2.1 为什么 main 要是 async 的？

```dart
void main() async {
  // async 允许我们使用 await 等待异步操作完成
  // 比如文件读取、网络请求等都是异步的
}
```

### 2.2 Flutter 绑定初始化

```dart
void main() async {
  // 必须调用！这是 Flutter 的固定写法
  WidgetsFlutterBinding.ensureInitialized();
  // ...
}
```

**为什么需要这一行？**

Flutter 的 `WidgetsFlutterBinding` 是一个全局单例，连接了 Flutter 引擎和 Dart 虚拟机。在调用任何 Flutter API 之前，必须确保这个绑定已经初始化。

### 2.3 MediaKit 初始化

```dart
MediaKit.ensureInitialized();
```

📁 这行代码初始化 media-kit 库，用于跨平台视频播放。

### 2.4 系统 UI 配置

```dart
if (Platform.isAndroid || Platform.isIOS) {
  // 设置为沉浸式模式（透明状态栏/导航栏）
  SystemChrome.setEnabledSystemUIMode(SystemUiMode.edgeToEdge);
  SystemChrome.setSystemUIOverlayStyle(const SystemUiOverlayStyle(
    systemNavigationBarColor: Colors.transparent,    // 底部导航栏透明
    systemNavigationBarDividerColor: Colors.transparent,
    statusBarColor: Colors.transparent,               // 顶部状态栏透明
  ));
}
```

📁 这让 Android/iOS 端实现全面屏效果。

### 2.5 数据存储初始化

```dart
try {
  // 获取应用数据目录
  final hivePath = '${(await getApplicationSupportDirectory()).path}/hive';

  // 初始化 Hive 数据库
  await Hive.initFlutter(hivePath);

  // 初始化我们的存储模块
  await GStorage.init();
} catch (e) {
  // 如果初始化失败，显示错误页面
  debugPrint('Storage initialization failed: $e');
  // ...
  return runApp(MaterialApp(
    // 显示存储错误页面
    builder: (context, child) => const StorageErrorPage(),
  ));
}
```

📁 源码详解：参考 [第四课：Hive 数据存储](./04-HIVE-STORAGE.md)

### 2.6 窗口管理初始化（桌面端）

```dart
if (Utils.isDesktop()) {
  // 初始化窗口管理器
  await windowManager.ensureInitialized();

  // 获取用户是否显示窗口按钮的设置
  bool showWindowButton = await GStorage.setting.get(
    SettingBoxKey.showWindowButton,
    defaultValue: false,
  );

  // 配置窗口参数
  WindowOptions windowOptions = WindowOptions(
    // 窗口尺寸：低分辨率用小窗口
    size: isLowResolution ? const Size(840, 600) : const Size(1280, 860),
    center: true,                          // 启动时居中显示
    skipTaskbar: false,                    // 不隐藏任务栏图标
    titleBarStyle: (Platform.isMacOS || !showWindowButton)
        ? TitleBarStyle.hidden
        : TitleBarStyle.normal,           // macOS 或关闭按钮设置时隐藏标题栏
    windowButtonVisibility: showWindowButton,  // 是否显示窗口按钮
    title: 'Kazumi',                       // 窗口标题
  );

  // 等待窗口准备就绪后显示
  windowManager.waitUntilReadyToShow(windowOptions, () async {
    await windowManager.show();
    await windowManager.focus();
  });
}
```

### 2.7 网络请求初始化

```dart
// 创建 Request 单例
Request();

// 设置 Cookie
await Request.setCookie();

// 应用代理设置
ProxyManager.applyProxy();
```

📁 源码详解：参考 [第五课：网络请求](./05-DIO-NETWORK.md)

### 2.8 启动应用

```dart
runApp(
  ChangeNotifierProvider(
    create: (_) => ThemeProvider(),           // 主题状态管理
    child: ModularApp(
      module: AppModule(),                    // 根模块
      child: const AppWidget(),              // 根组件
    ),
  ),
);
```

---

## 3. 模块初始化

### 3.1 AppModule - 根模块配置

📁 源码：`lib/app_module.dart`

```dart
class AppModule extends Module {
  @override
  void binds(i) {
    // 绑定依赖（这里为空，因为我们不需要全局单例）
  }

  @override
  void routes(r) {
    // 定义路由
    r.module("/", module: IndexModule());
    // 访问 / 时加载 IndexModule
  }
}
```

### 3.2 IndexModule - 主页面模块

📁 源码：`lib/pages/index_module.dart:27-95`

```dart
class IndexModule extends Module {
  @override
  List<Module> get imports => menu.moduleList;  // 引入底部导航的模块列表

  @override
  void binds(i) {
    // ===== Repository 层 =====
    i.addSingleton<ICollectRepository>(CollectRepository.new);
    i.addSingleton<ISearchHistoryRepository>(SearchHistoryRepository.new);
    i.addSingleton<ICollectCrudRepository>(CollectCrudRepository.new);
    i.addSingleton<IHistoryRepository>(HistoryRepository.new);
    i.addSingleton<IDownloadRepository>(DownloadRepository.new);
    i.addSingleton<IDownloadManager>(DownloadManager.new);

    // ===== Controller 层 =====
    i.addSingleton(PopularController.new);
    i.addSingleton(PluginsController.new);
    i.addSingleton(VideoPageController.new);
    i.addSingleton(TimelineController.new);
    i.addSingleton(CollectController.new);
    i.addSingleton(HistoryController.new);
    i.addSingleton(MyController.new);
    i.addSingleton(ShadersController.new);
    i.addSingleton(DownloadController.new);
  }

  @override
  void routes(r) {
    // 主路由
    r.child("/",
      child: (_) => const InitPage(),
      children: [
        // 子路由定义
        ChildRoute("/error", child: (_) => ...),
      ],
    );

    // Tab 页面（底部导航）
    r.child("/tab",
      child: (_) => const IndexPage(),
      children: menu.routes,       // 菜单路由
    );

    // 其他页面模块
    r.module("/video", module: VideoModule());
    r.module("/info", module: InfoModule());
    r.module("/settings", module: SettingsModule());
    r.module("/search", module: SearchModule());
  }
}
```

---

## 4. 应用生命周期

📁 源码：`lib/app_widget.dart:25-54`

### 4.1 State 的生命周期方法

```dart
class _AppWidgetState extends State<AppWidget>
    with TrayListener,           // 托盘事件监听
        WidgetsBindingObserver,   // 应用生命周期监听
        WindowListener {         // 窗口事件监听

  @override
  void initState() {
    // 组件创建时调用
    trayManager.addListener(this);
    windowManager.addListener(this);
    setPreventClose();
    WidgetsBinding.instance.addObserver(this);
    super.initState();
  }

  @override
  void dispose() {
    // 组件销毁时调用
    trayManager.removeListener(this);
    windowManager.removeListener(this);
    WidgetsBinding.instance.removeObserver(this);
    super.dispose();
  }
}
```

### 4.2 AppLifecycleState - 应用状态

📁 源码：`lib/app_widget.dart:154-165`

```dart
@override
void didChangeAppLifecycleState(AppLifecycleState state) async {
  switch (state) {
    case AppLifecycleState.paused:
      // 应用进入后台（移动端）或失去焦点（桌面端）
      break;
    case AppLifecycleState.resumed:
      // 应用回到前台
      break;
    case AppLifecycleState.inactive:
      // 应用处于非活动状态（过渡状态）
      break;
    case AppLifecycleState.detached:
      // Flutter 引擎正在分离（仅 Android）
      break;
    case AppLifecycleState.hidden:
      // 应用完全隐藏（Flutter 3.22+）
      break;
  }
}
```

### 4.3 生命周期图示

```
┌─────────────────────────────────────────┐
│           Application Lifecycle          │
├─────────────────────────────────────────┤
│                                         │
│   inactive ────── resumed               │  ← 用户可见
│       │              │                  │
│       ↓              ↓                  │
│   paused ──────► hidden                 │  ← 用户不可见
│                                         │
└─────────────────────────────────────────┘
```

---

## 5. 窗口与托盘管理

### 5.1 托盘图标设置

📁 源码：`lib/app_widget.dart:182-202`

```dart
Future<void> _handleTray() async {
  // 根据不同平台设置不同图标
  if (Platform.isWindows) {
    await trayManager.setIcon('assets/images/logo/logo_lanczos.ico');
  } else if (Platform.environment.containsKey('FLATPAK_ID') ||
             Platform.environment.containsKey('SNAP')) {
    // Linux 特殊包管理器
    await trayManager.setIcon('io.github.Predidit.Kazumi');
  } else {
    await trayManager.setIcon('assets/images/logo/logo_rounded.png');
  }

  // 设置提示文字
  await trayManager.setToolTip('Kazumi');

  // 设置右键菜单
  Menu trayMenu = Menu(items: [
    MenuItem(key: 'show_window', label: '显示窗口'),
    MenuItem.separator(),
    MenuItem(key: 'exit', label: '退出 Kazumi')
  ]);
  await trayManager.setContextMenu(trayMenu);
}
```

### 5.2 窗口关闭处理

📁 源码：`lib/app_widget.dart:78-148`

```dart
@override
void onWindowClose() {
  // 读取用户设置的退出行为
  final exitBehavior = setting.get(SettingBoxKey.exitBehavior, defaultValue: 2);

  switch (exitBehavior) {
    case 0:
      // 直接退出
      exit(0);

    case 1:
      // 最小化到托盘
      KazumiDialog.dismiss();
      windowManager.hide();
      break;

    default:
      // 弹出确认对话框
      KazumiDialog.show(builder: (context) {
        return AlertDialog(
          title: const Text('退出确认'),
          content: const Text('您想要退出 Kazumi 吗？'),
          actions: [
            TextButton(onPressed: () => exit(0), child: const Text('退出')),
            TextButton(onPressed: () {
              windowManager.hide();
            }, child: const Text('最小化')),
          ],
        );
      });
  }
}
```

---

## 6. 主题系统

📁 源码：`lib/app_widget.dart:204-331`

### 6.1 主题提供者

```dart
// 使用 Provider 提供主题状态
runApp(
  ChangeNotifierProvider(
    create: (_) => ThemeProvider(),  // 创建主题提供者
    child: ModularApp(...),
  ),
);
```

### 6.2 主题配置

```dart
var defaultDarkTheme = ThemeData(
  useMaterial3: true,           // 使用 Material Design 3
  fontFamily: 'MiSans',         // 使用自定义字体
  brightness: Brightness.dark,  // 深色模式
  colorSchemeSeed: color,       // 主题色
);

// 创建浅色主题和深色主题
themeProvider.setTheme(
  ThemeData(brightness: Brightness.light, ...),  // 浅色
  ThemeData(brightness: Brightness.dark, ...),    // 深色
);
```

### 6.3 动态颜色

```dart
var app = DynamicColorBuilder(
  builder: (ColorScheme? light, ColorScheme? dark) {
    if (themeProvider.useDynamicColor) {
      // 如果用户开启了动态颜色，使用系统主题色
      themeProvider.setTheme(
        ThemeData(colorScheme: light),
        ThemeData(colorScheme: dark),
      );
    }
    return MaterialApp.router(
      themeMode: themeProvider.themeMode,  // light / dark / system
      // ...
    );
  },
);
```

---

## 小练习

### 练习 1：追踪初始化流程

📁 阅读 `lib/main.dart`，回答以下问题：

1. 如果存储初始化失败，应用会显示什么？
2. 桌面端和移动端的初始化有什么区别？

<details>
<summary>答案</summary>

1. 会显示 `StorageErrorPage()` 错误页面（第 63 行）
2. 桌面端需要初始化 `windowManager`（窗口大小、标题栏等），移动端需要配置 `SystemChrome`（透明状态栏）

</details>

### 练习 2：理解模块注入

📁 阅读 `lib/pages/index_module.dart:32-50`，回答：

1. `i.addSingleton` 和 `i.add` 有什么区别？
2. 为什么 Repository 层使用接口类型（如 `ICollectRepository`），而 Controller 层使用具体类型？

<details>
<summary>答案</summary>

1. `addSingleton` 注册单例（全局唯一实例），`add` 每次注入创建新实例
2. Repository 使用接口是为了解耦和方便替换实现；Controller 已经是具体实现类

</details>

### 练习 3：添加新的全局状态

假设我们要添加一个新的全局设置 `showDanmaku`（是否显示弹幕），需要修改哪些地方？

<details>
<summary>提示</summary>

1. 在 `SettingBoxKey` 中添加 key
2. 在 `GStorage` 中添加访问方法（可选）
3. 在需要使用的地方读取 `GStorage.setting.get(SettingBoxKey.showDanmaku)`

</details>

---

## 下一课预告

下一课我们将学习 [MobX 状态管理](./03-MOBX-STATE.md)，了解响应式编程在本项目中的应用。