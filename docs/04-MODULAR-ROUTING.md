# 第四章：路由与依赖注入（Flutter Modular）

> 目标：理解 Flutter Modular 如何组织路由和管理依赖，对照 Spring IoC 理解
> 配套源码：`lib/app_module.dart`、`lib/pages/index_module.dart`、`lib/pages/video/video_module.dart`

---

## 1. Flutter Modular 是什么

Flutter Modular = Spring IoC + Spring MVC 路由的合体。它同时解决两个问题：
- **路由管理**：页面之间如何跳转（≈ `@RequestMapping`）
- **依赖注入**：对象如何创建和获取（≈ `@Autowired`）

### 对照 Spring Boot

```java
// Spring: 路由 + DI 分开配置
@RestController
@RequestMapping("/popular")
public class PopularController {
    @Autowired
    private PopularService service;
}
```

```dart
// Flutter Modular: 路由 + DI 在 Module 中统一配置
class IndexModule extends Module {
  @override
  void binds(i) {
    i.addSingleton(PopularController.new);  // DI 注册
  }

  @override
  void routes(r) {
    r.child("/popular", child: (_) => PopularPage());  // 路由注册
  }
}
```

---

## 2. Module 结构

### 根 Module

`lib/app_module.dart` — 应用的根模块，类似 Spring Boot 的 `@SpringBootApplication`：

```dart
class AppModule extends Module {
  @override
  void routes(r) {
    r.module('/', module: IndexModule());  // 所有请求转发给 IndexModule
  }
}
```

### IndexModule — 核心配置

`lib/pages/index_module.dart` — 注册所有全局依赖和主路由：

```dart
class IndexModule extends Module {
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
    i.addSingleton(CollectController.new);
    i.addSingleton(HistoryController.new);
    i.addSingleton(MyController.new);
  }

  @override
  void routes(r) {
    r.child("/", child: (_) => const InitPage());
    r.child("/tab", child: (_) => const IndexPage(), children: menu.routes);
    r.module("/video", module: VideoModule());
    r.module("/info", module: InfoModule());
    r.module("/settings", module: SettingsModule());
    r.module("/search", module: SearchModule());
  }
}
```

### 功能 Module

每个功能模块有自己的 Module 文件，定义子路由：

```dart
// lib/pages/video/video_module.dart
class VideoModule extends Module {
  @override
  void routes(r) {
    r.child("/", child: (_) => const VideoPage());
  }
}
```

---

## 3. 依赖注入详解

### 注册方式

```dart
void binds(i) {
  // 单例（整个应用生命周期只有一个实例）≈ @Scope("singleton")
  i.addSingleton(PopularController.new);

  // 接口绑定实现（≈ @Bean IRepository → RepositoryImpl）
  i.addSingleton<ICollectRepository>(CollectRepository.new);

  // 懒加载单例（首次使用时才创建）
  i.addLazySingleton(HeavyService.new);

  // 每次获取都创建新实例 ≈ @Scope("prototype")
  i.add(SearchPageController.new);
}
```

### 获取依赖

```dart
// 方式 1：直接获取（最常用）≈ applicationContext.getBean()
final controller = Modular.get<PopularController>();

// 方式 2：在类属性中获取
class _CollectController with Store {
  final _repository = Modular.get<ICollectRepository>();
}
```

### 接口与实现分离

```dart
// 定义接口（≈ Java interface）
abstract class ICollectRepository {
  List<CollectedBangumi> getAllCollectibles();
  Future<void> addCollectible(BangumiItem item, int type);
  Future<void> deleteCollectible(int bangumiId);
}

// 实现类
class CollectRepository implements ICollectRepository {
  @override
  List<CollectedBangumi> getAllCollectibles() {
    return GStorage.collectibles.values.toList();
  }
  // ...
}

// 注册时绑定接口到实现
i.addSingleton<ICollectRepository>(CollectRepository.new);

// 使用时通过接口获取（面向接口编程）
final repo = Modular.get<ICollectRepository>();
```

---

## 4. 路由详解

### 路由定义

```dart
void routes(r) {
  // 子路由（当前 Module 内的页面）
  r.child("/", child: (_) => const InitPage());

  // 模块路由（委托给子 Module 处理）
  r.module("/video", module: VideoModule());

  // 带子路由的页面
  r.child("/tab", child: (_) => const IndexPage(), children: [
    ChildRoute("/popular", child: (_) => PopularPage()),
    ChildRoute("/timeline", child: (_) => TimelinePage()),
  ]);

  // 带过渡动画
  r.child("/tab",
    child: (_) => const IndexPage(),
    transition: TransitionType.fadeIn,
    duration: Duration(milliseconds: 70),
  );
}
```

### 导航

```dart
// 跳转到指定路由
Modular.to.pushNamed('/video');

// 带参数跳转
Modular.to.pushNamed('/info', arguments: bangumiItem);

// 在目标页面获取参数
final item = Modular.args.data as BangumiItem;

// 返回上一页
Modular.to.pop();

// 替换当前页（不可返回）
Modular.to.pushReplacementNamed('/tab');
```

### 底部导航栏路由

`lib/pages/router.dart` 定义了主页的 Tab 路由：

```dart
// 底部导航栏的四个 Tab
final menu = MenuRoute([
  MenuRouteItem(path: "/popular", module: PopularModule()),
  MenuRouteItem(path: "/timeline", module: TimelineModule()),
  MenuRouteItem(path: "/collect", module: CollectModule()),
  MenuRouteItem(path: "/my", module: MyModule()),
]);
```

---

## 5. 完整的页面创建流程

假设你要新增一个"关于"页面：

### Step 1: 创建 Controller

```dart
// lib/pages/about/about_controller.dart
import 'package:mobx/mobx.dart';
part 'about_controller.g.dart';

class AboutController = _AboutController with _$AboutController;

abstract class _AboutController with Store {
  @observable
  String version = '';

  @action
  void loadVersion() {
    version = '2.1.0';
  }
}
```

### Step 2: 创建 Page

```dart
// lib/pages/about/about_page.dart
class AboutPage extends StatefulWidget {
  @override
  State<AboutPage> createState() => _AboutPageState();
}

class _AboutPageState extends State<AboutPage> {
  final controller = Modular.get<AboutController>();

  @override
  void initState() {
    super.initState();
    controller.loadVersion();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('关于')),
      body: Observer(
        builder: (_) => Text('版本: ${controller.version}'),
      ),
    );
  }
}
```

### Step 3: 创建 Module

```dart
// lib/pages/about/about_module.dart
class AboutModule extends Module {
  @override
  void routes(r) {
    r.child('/', child: (_) => const AboutPage());
  }
}
```

### Step 4: 注册到父 Module

```dart
// 在 IndexModule 或 SettingsModule 中添加
void binds(i) {
  i.addSingleton(AboutController.new);  // 注册 Controller
}

void routes(r) {
  r.module("/about", module: AboutModule());  // 注册路由
}
```

### Step 5: 生成代码

```bash
dart run build_runner build --delete-conflicting-outputs
```

---

## 6. 对照总结

| Spring Boot | Flutter Modular | 说明 |
|-------------|-----------------|------|
| `@SpringBootApplication` | `AppModule` | 根配置 |
| `@Configuration` | `Module.binds()` | 依赖注册 |
| `@Bean` | `i.addSingleton()` | 注册单例 |
| `@Autowired` | `Modular.get<T>()` | 获取依赖 |
| `@RequestMapping` | `Module.routes()` | 路由定义 |
| `@GetMapping("/path")` | `r.child("/path", ...)` | 具体路由 |
| `@Scope("prototype")` | `i.add()` | 每次新建实例 |
| `@Scope("singleton")` | `i.addSingleton()` | 全局单例 |
| `redirect:/path` | `Modular.to.pushNamed()` | 页面跳转 |
| `@PathVariable` | `Modular.args` | 路由参数 |

---

## 练习

1. 在 `lib/pages/index_module.dart` 中，找出哪些 Controller 是单例注册的
2. 追踪从 `AppModule` → `IndexModule` → `VideoModule` 的路由嵌套关系
3. 如果你要新增一个"日志查看"页面，需要修改哪些文件？

<details>
<summary>答案</summary>

1. 所有 Controller 都是 `addSingleton` 注册的（全局单例）
2. `AppModule` 的 `/` → `IndexModule`，`IndexModule` 的 `/video` → `VideoModule`，最终路径是 `/video`
3. 需要：创建 `logs_controller.dart`、`logs_page.dart`、`logs_module.dart`，然后在 `IndexModule` 中注册 Controller 和路由

</details>

---

下一章：[MobX 响应式状态管理](./05-MOBX-STATE.md)
