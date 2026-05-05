# 第七课：Flutter Modular 路由

> 📖 学习目标：理解 Flutter Modular 路由系统，学会创建页面、配置路由、处理导航
> ⏱️ 预计时间：2-3 小时
> 📁 配套源码：`lib/pages/index_module.dart`、`lib/pages/router.dart`、`lib/pages/menu/menu.dart`

---

## 目录

1. [路由概述](#1-路由概述)
2. [Module 系统](#2-module-系统)
3. [路由配置](#3-路由配置)
4. [依赖注入](#4-依赖注入)
5. [页面导航](#5-页面导航)
6. [底部导航](#6-底部导航)
7. [路由守卫](#7-路由守卫)
8. [实战：创建新页面](#8-实战创建新页面)

---

## 1. 路由概述

### 1.1 什么是路由？

路由就像"地图"，告诉应用：
- 当用户访问 `/search` 时，显示哪个页面
- 当用户点击按钮时，跳转到哪个页面
- 页面之间如何传递数据

### 1.2 URL 结构

```
https://kazumi.app/
│
├── /                    → InitPage（初始化页）
├── /error               → 错误页
├── /tab                 → 主页面容器
│   ├── /popular         → 推荐页
│   ├── /timeline        → 时间表页
│   ├── /collect         → 收藏页
│   └── /my              → 我的页
├── /video               → 视频播放页
├── /info                → 番剧详情页
├── /settings            → 设置页
└── /search              → 搜索页
```

### 1.3 Flutter Modular 核心概念

| 概念 | 说明 |
|------|------|
| Module | 模块，定义路由和依赖 |
| Route | 路由，页面路径映射 |
| binds | 依赖注入配置 |
| routes | 路由配置 |
| RouterOutlet | 路由出口，渲染子页面 |

---

## 2. Module 系统

### 2.1 什么是 Module？

Module 就像一个"功能包"，包含：
- 页面组件（Widget）
- 状态管理（Controller）
- 路由配置（Routes）
- 依赖注入（binds）

📁 源码：`lib/pages/index_module.dart`

```dart
class IndexModule extends Module {
  // 引入其他模块
  @override
  List<Module> get imports => menu.moduleList;

  // 依赖注入配置
  @override
  void binds(i) {
    // 添加单例
    i.addSingleton(PopularController.new);
    i.addSingleton(PluginsController.new);
    // ...
  }

  // 路由配置
  @override
  void routes(r) {
    r.child("/tab", child: (_) => IndexPage(), children: menu.routes);
    r.module("/video", module: VideoModule());
    r.module("/search", module: SearchModule());
  }
}
```

### 2.2 标准 Module 结构

```
lib/pages/example/
├── example_controller.dart    ← 状态管理
├── example_controller.g.dart   ← 代码生成（不要手动修改）
├── example_module.dart         ← 路由配置
├── example_page.dart          ← 页面 UI
└── example_page_state.dart   ← 可选的状态组件
```

### 2.3 Module 继承关系

```
Module (基类)
  │
  ├── IndexModule (主模块)
  │     ├── imports: [PopularModule, TimelineModule, CollectModule, MyModule]
  │
  ├── PopularModule
  │     └── routes: /popular
  │
  ├── TimelineModule
  │     └── routes: /timeline
  │
  └── ...
```

---

## 3. 路由配置

### 3.1 路由类型

📁 源码：`lib/pages/index_module.dart:54-93`

```dart
@override
void routes(r) {
  // 1. child - 子路由（配合 children 使用）
  r.child("/",
    child: (_) => const InitPage(),
    children: [
      ChildRoute("/error", child: (_) => ...),
    ],
  );

  // 2. module - 子模块（完整模块，包含自己的路由）
  r.module("/video", module: VideoModule());
  r.module("/info", module: InfoModule());
  r.module("/settings", module: SettingsModule());
  r.module("/search", module: SearchModule());

  // 3. 自定义路由（带参数）
  r.child(
    ImageViewer.routePath,
    child: (_) {
      final args = Modular.args.data as ImageViewerRouteArgs;
      return ImageViewer(...);
    },
  );
}
```

### 3.2 ChildRoute 子路由

```dart
r.child("/path",
  child: (_) => MyWidget(),        // 页面组件
  children: [...],                  // 子路由
  transition: TransitionType.fadeIn,  // 过渡动画
  duration: Duration(milliseconds: 300),  // 动画时长
);
```

### 3.3 ModuleRoute 子模块

```dart
r.module("/path", module: MyModule());
```

子模块可以有自己的路由、子模块、依赖注入。

### 3.4 路由参数传递

**方式 1：通过 arguments**

```dart
// 跳转时传递参数
Modular.to.pushNamed('/info', arguments: bangumiItem);

// 目标页面接收参数
class InfoModule extends Module {
  @override
  void routes(r) {
    r.module("/info", module: InfoModule());

    // 方式 A：在 build 方法中获取
    r.child("/page", child: (args) {
      final bangumi = args.data as BangumiItem;
      return InfoPage(bangumi: bangumi);
    });

    // 方式 B：在 Route 中定义
    r.child("/page",
      child: (_) => InfoPage(),
      args: InfoPageArgs,  // 必须在 args 中声明
    );
  }
}
```

**方式 2：通过 url 参数**

```dart
// URL: /detail/123
r.child("/detail/:id", child: (args) {
  final id = Modular.args.params['id'];
  return DetailPage(id: id);
});
```

---

## 4. 依赖注入

### 4.1 binds 方法

📁 源码：`lib/pages/index_module.dart:32-50`

```dart
@override
void binds(i) {
  // 添加单例（全局唯一）
  i.addSingleton(PopularController.new);
  i.addSingleton(PluginsController.new);

  // 添加实例（每次获取都是新实例）
  i.add(MyService.new);

  // 添加接口绑定
  i.addSingleton<ICollectRepository>(CollectRepository.new);

  // 绑定值
  i.bind<String>(tag: 'apiKey', value: 'xxx');
}
```

### 4.2 获取依赖

```dart
// 在 Controller/Page 中获取
class _PopularController with Store {
  // 方式 1：从 Modular 获取
  final controller = Modular.get<PopularController>();

  // 方式 2：构造函数注入（在 Module 中配置）
  _XxxController(this._service);

  // 方式 3：延迟获取（避免循环依赖）
  MyService get _service {
    if (_cached == null) {
      _cached = Modular.get<MyService>();
    }
    return _cached!;
  }
}
```

### 4.3 生命周期

```
应用启动
    ↓
IndexModule.init()
    ↓
binds() 注册依赖
    ↓
routes() 配置路由
    ↓
访问 /tab → PopularModule
    ↓
PopularModule.init()
    ↓
导入的依赖可用（来自 IndexModule）
```

---

## 5. 页面导航

### 5.1 导航方法

```dart
// 跳转（压栈）
Modular.to.pushNamed('/path');

// 替换当前页面
Modular.to.pushReplacementNamed('/path');

// 返回
Modular.to.pop();

// 返回到指定页面
Modular.to.popUntil('/home');

// 跳转并清除之前所有页面
Modular.to.navigateAndRemoveUntil('/splash', ModalRoute.withName('/splash'));
```

### 5.2 带参数的跳转

```dart
// 跳转并传递参数
Modular.to.pushNamed(
  '/info',
  arguments: BangumiItem(...),
);

// 或者
Modular.to.pushNamed(
  '/detail/${bangumi.id}',
  arguments: {'id': bangumi.id},
);

// 接收参数
class InfoModule extends Module {
  @override
  void routes(r) {
    r.module("/info", module: InfoModule());
  }
}

// 在 InfoPage 中
class InfoPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final BangumiItem bangumi = Modular.args.data;
    // ...
  }
}
```

### 5.3 动画过渡

```dart
// 无动画
r.child("/page", child: (_) => ..., transition: TransitionType.noTransition);

// 淡入淡出
r.child("/page", child: (_) => ...,
  transition: TransitionType.fadeIn,
  duration: Duration(milliseconds: 300),
);

// 从右侧滑入
r.child("/page", child: (_) => ...,
  transition: TransitionType.rightToLeft,
);

// 从底部弹出
r.child("/page", child: (_) => ...,
  transition: TransitionType.bottomToTop,
);

// 缩放
r.child("/page", child: (_) => ...,
  transition: TransitionType.scale,
);
```

---

## 6. 底部导航

📁 源码：`lib/pages/menu/menu.dart`

### 6.1 路由配置

📁 源码：`lib/pages/router.dart`

```dart
// 定义菜单项
final MenuRoute menu = MenuRoute([
  MenuRouteItem(path: "/popular", module: PopularModule()),
  MenuRouteItem(path: "/timeline", module: TimelineModule()),
  MenuRouteItem(path: "/collect", module: CollectModule()),
  MenuRouteItem(path: "/my", module: MyModule()),
]);

// MenuRoute 类
class MenuRoute {
  final List<MenuRouteItem> menuList;

  List<Module> get moduleList {
    return menuList.map((e) => e.module).toList();
  }

  List<ModuleRoute> get routes {
    return menuList.map((e) => ModuleRoute(e.path, module: e.module)).toList();
  }

  getPath(int index) => menuList[index].path;
}
```

### 6.2 底部导航栏

📁 源码：`lib/pages/menu/menu.dart:77-118`

```dart
NavigationBar(
  destinations: const <Widget>[
    NavigationDestination(
      selectedIcon: Icon(Icons.home),
      icon: Icon(Icons.home_outlined),
      label: '推荐',
    ),
    NavigationDestination(
      selectedIcon: Icon(Icons.timeline),
      icon: Icon(Icons.timeline_outlined),
      label: '时间表',
    ),
    // ...
  ],
  selectedIndex: state.selectedIndex,
  onDestinationSelected: (int index) {
    // 更新索引
    state.updateSelectedIndex(index);
    // 跳转路由
    Modular.to.navigate("/tab${menu.getPath(index)}/");
  },
)
```

### 6.3 RouterOutlet

```dart
Scaffold(
  body: PageView.builder(
    physics: const NeverScrollableScrollPhysics(),  // 禁止滑动切换
    controller: _page,
    itemCount: menu.size,
    itemBuilder: (_, __) => const RouterOutlet(),  // 路由出口
  ),
  bottomNavigationBar: NavigationBar(...),
)
```

`RouterOutlet` 会根据当前路由自动渲染对应的页面。

### 6.4 横屏侧边栏

📁 源码：`lib/pages/menu/menu.dart:121-195`

```dart
Row(
  children: [
    NavigationRail(  // 侧边导航
      destinations: [...],
      selectedIndex: state.selectedIndex,
      onDestinationSelected: (int index) {
        state.updateSelectedIndex(index);
        Modular.to.navigate("/tab${menu.getPath(index)}/");
      },
    ),
    Expanded(
      child: PageView.builder(
        itemBuilder: (_, __) => const RouterOutlet(),
      ),
    ),
  ],
)
```

---

## 7. 路由守卫

### 7.1 什么是路由守卫？

路由守卫可以在跳转前检查条件：
- 用户是否登录？
- 是否有权限？
- 是否需要加载数据？

### 7.2 自定义路由守卫

```dart
// 创建守卫
class AuthGuard extends RouteGuard {
  @override
  Future<bool> canActivate(String path, ModularNode node) async {
    final auth = Modular.get<AuthService>();
    return auth.isLoggedIn;
  }
}

// 使用守卫
@override
void routes(r) {
  r.child("/profile",
    child: (_) => ProfilePage(),
    guard: AuthGuard(),
  );
}
```

### 7.3 数据预加载

```dart
class DataGuard extends RouteGuard {
  @override
  Future<bool> canActivate(String path, ModularNode node) async {
    final id = node.args.data['id'];
    await Modular.get<DataService>().loadData(id);
    return true;
  }
}

r.child("/detail",
  child: (_) => DetailPage(),
  guard: DataGuard(),
);
```

---

## 8. 实战：创建新页面

### 8.1 需求

创建一个"历史记录"页面 `/history`，包含：
- 显示观看历史列表
- 支持删除历史

### 8.2 步骤 1：创建目录和文件

```
lib/pages/history/
├── history_controller.dart
├── history_controller.g.dart    ← 运行 build_runner 生成
├── history_module.dart
└── history_page.dart
```

### 8.3 步骤 2：编写 Controller

📁 源码：`lib/pages/history/history_controller.dart`

```dart
import 'package:mobx/mobx.dart';
import 'package:kazumi/utils/storage.dart';

part 'history_controller.g.dart';

class HistoryController = _HistoryController with _$HistoryController;

abstract class _HistoryController with Store {
  Box setting = GStorage.setting;

  @observable
  ObservableList<History> historyList = ObservableList();

  @action
  void loadHistory() {
    historyList.clear();
    historyList.addAll(GStorage.histories.values.toList());
    // 按时间排序
    historyList.sort((a, b) => b.lastWatchTime.compareTo(a.lastWatchTime));
  }

  @action
  Future<void> deleteHistory(History history) async {
    await GStorage.histories.delete(history.key);
    loadHistory();
  }

  @action
  Future<void> clearAll() async {
    await GStorage.histories.clear();
    loadHistory();
  }
}
```

### 8.4 步骤 3：编写 Module

```dart
import 'package:flutter_modular/flutter_modular.dart';
import 'package:kazumi/pages/history/history_controller.dart';
import 'package:kazumi/pages/history/history_page.dart';

class HistoryModule extends Module {
  @override
  void routes(r) {
    r.child("/history", child: (_) => HistoryPage());
  }
}
```

### 8.5 步骤 4：编写 Page

```dart
import 'package:flutter/material.dart';
import 'package:flutter_modular/flutter_modular.dart';
import 'package:flutter_mobx/flutter_mobx.dart';
import 'package:kazumi/pages/history/history_controller.dart';

class HistoryPage extends StatelessWidget {
  final controller = Modular.get<HistoryController>();

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('历史记录'),
        actions: [
          IconButton(
            icon: const Icon(Icons.delete_sweep),
            onPressed: () => controller.clearAll(),
          ),
        ],
      ),
      body: Observer(
        builder: (_) {
          if (controller.historyList.isEmpty) {
            return const Center(child: Text('暂无历史记录'));
          }
          return ListView.builder(
            itemCount: controller.historyList.length,
            itemBuilder: (context, index) {
              final history = controller.historyList[index];
              return ListTile(
                title: Text(history.bangumiName),
                subtitle: Text(history.lastWatchTime.toString()),
                trailing: IconButton(
                  icon: const Icon(Icons.delete),
                  onPressed: () => controller.deleteHistory(history),
                ),
              );
            },
          );
        },
      ),
    );
  }
}
```

### 8.6 步骤 5：注册路由

在 `lib/pages/index_module.dart` 中添加：

```dart
import 'package:kazumi/pages/history/history_module.dart';

class IndexModule extends Module {
  @override
  void routes(r) {
    // ... 其他路由

    r.module("/history", module: HistoryModule());
  }
}
```

### 8.7 步骤 6：生成代码

```bash
dart run build_runner build
```

### 8.8 步骤 7：添加到底部导航

📁 修改 `lib/pages/router.dart`

```dart
final MenuRoute menu = MenuRoute([
  // ... 原有路由
  MenuRouteItem(path: "/history", module: HistoryModule()),
]);
```

---

## 小练习

### 练习 1：理解路由配置

阅读 `lib/pages/index_module.dart`，回答：

1. `/tab` 路由使用什么过渡动画？
2. 如何跳转到 `/video` 页面？
3. `ImageViewer` 页面通过什么方式接收参数？

<details>
<summary>答案</summary>

1. `TransitionType.fadeIn`，时长 70 毫秒
2. `Modular.to.pushNamed('/video')` 或 `Modular.to.navigate('/video')`
3. 通过 `Modular.args.data` 接收 `ImageViewerRouteArgs` 对象

</details>

### 练习 2：添加新路由

在 `IndexModule` 中添加一个 `/about` 路由，显示 `AboutPage`。

<details>
<summary>答案</summary>

```dart
// 1. 添加导入
import 'package:kazumi/pages/about/about_page.dart';

// 2. 在 routes 中添加
@override
void routes(r) {
  // ... 其他路由

  r.child("/about", child: (_) => const AboutPage());
}
```

</details>

### 练习 3：实现页面跳转

在 `PopularPage` 中添加按钮，点击后跳转到 `/search` 页面。

<details>
<summary>答案</summary>

```dart
// 在 build 方法中添加
IconButton(
  icon: const Icon(Icons.search),
  onPressed: () => Modular.to.pushNamed('/search/'),
);
```

</details>

---

## 下一课预告

下一课我们将学习 [数据模型](./08-DATA-MODELS.md)，了解项目中各个模型类的设计和关系。