# 第二章：Flutter 基础 — Widget 体系与生命周期

> 目标：理解 Flutter 的 UI 构建方式，从"命令式 UI"思维转向"声明式 UI"思维
> 配套源码：`lib/app_widget.dart`、`lib/pages/popular/popular_page.dart`

---

## 1. 思维转换：命令式 → 声明式

### Java/Android 的命令式 UI

```java
// Java: 找到组件 → 修改属性
TextView title = findViewById(R.id.title);
title.setText("Hello");
title.setVisibility(View.VISIBLE);
```

### Flutter 的声明式 UI

```dart
// Flutter: 描述"当前状态下 UI 应该长什么样"
Widget build(BuildContext context) {
  return Text(isVisible ? 'Hello' : '');
}
```

核心区别：
- 命令式：你告诉框架"怎么改"（步骤）
- 声明式：你告诉框架"应该是什么样"（结果），框架自己算出怎么改

类比 Spring Boot：
- 命令式 ≈ 手动管理 JDBC Connection
- 声明式 ≈ 用 `@Transactional` 注解，框架帮你管理

---

## 2. Widget 是什么

Flutter 中一切皆 Widget。Widget 是 UI 的描述（不是 UI 本身），类似于一个"配置对象"。

```dart
// 这不是一个按钮实例，而是"我想要一个按钮"的描述
ElevatedButton(
  onPressed: () => print('clicked'),
  child: Text('点击我'),
)
```

### Widget 树

Flutter UI 是一棵 Widget 树，类似 HTML DOM：

```
MaterialApp
└── Scaffold
    ├── AppBar
    │   └── Text('标题')
    └── Body
        └── ListView
            ├── ListTile
            ├── ListTile
            └── ListTile
```

项目实例 — 简化的 `lib/app_widget.dart`:
```dart
MaterialApp.router(
  title: 'Kazumi',
  theme: lightTheme,          // 浅色主题
  darkTheme: darkTheme,       // 深色主题
  themeMode: themeMode,       // 跟随系统/手动切换
  routerConfig: Modular.routerConfig,  // 路由配置
)
```

---

## 3. StatelessWidget vs StatefulWidget

### StatelessWidget — 无状态组件

数据从外部传入，自身不维护状态。类似 Java 的不可变对象。

```dart
class BangumiCard extends StatelessWidget {
  final BangumiItem item;  // 从外部传入的数据

  const BangumiCard({required this.item});

  @override
  Widget build(BuildContext context) {
    return Card(
      child: Column(
        children: [
          Image.network(item.imageUrl),
          Text(item.name),
        ],
      ),
    );
  }
}
```

### StatefulWidget — 有状态组件

自身维护可变状态，状态变化时重新构建 UI。

```dart
class CounterPage extends StatefulWidget {
  @override
  State<CounterPage> createState() => _CounterPageState();
}

class _CounterPageState extends State<CounterPage> {
  int count = 0;  // 内部状态

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        Text('Count: $count'),
        ElevatedButton(
          onPressed: () {
            setState(() {  // 通知框架重建 UI
              count++;
            });
          },
          child: Text('加一'),
        ),
      ],
    );
  }
}
```

### 本项目的选择

Kazumi 项目中，大部分页面使用 **StatefulWidget + MobX**：
- StatefulWidget 提供生命周期（initState/dispose）
- MobX 的 `Observer` 替代 `setState`，实现精准更新

```dart
// 项目中的典型模式
class PopularPage extends StatefulWidget {
  @override
  State<PopularPage> createState() => _PopularPageState();
}

class _PopularPageState extends State<PopularPage> {
  // 从 DI 容器获取 Controller（≈ @Autowired）
  final controller = Modular.get<PopularController>();

  @override
  void initState() {
    super.initState();
    controller.queryBangumiByTrend(type: 'init');  // 页面初始化时加载数据
  }

  @override
  Widget build(BuildContext context) {
    return Observer(  // MobX 的 Observer 监听状态变化
      builder: (_) {
        if (controller.isLoadingMore && controller.trendList.isEmpty) {
          return CircularProgressIndicator();
        }
        return ListView.builder(
          itemCount: controller.trendList.length,
          itemBuilder: (_, index) => BangumiCard(item: controller.trendList[index]),
        );
      },
    );
  }
}
```

---

## 4. 生命周期

### StatefulWidget 生命周期

```
createState() → initState() → build() → [setState → build]* → dispose()
```

对照 Spring Bean 生命周期：

| Flutter | Spring | 用途 |
|---------|--------|------|
| `initState()` | `@PostConstruct` | 初始化，加载数据 |
| `build()` | — | 构建 UI（每次状态变化都调用） |
| `dispose()` | `@PreDestroy` | 清理资源，取消订阅 |
| `didChangeDependencies()` | — | 依赖变化时调用 |

项目实例：
```dart
@override
void initState() {
  super.initState();
  // 页面创建时加载数据（≈ @PostConstruct）
  controller.loadData();
}

@override
void dispose() {
  // 页面销毁时清理（≈ @PreDestroy）
  controller.scrollController.dispose();
  super.dispose();
}
```

### App 生命周期

`lib/app_widget.dart` 中监听应用级生命周期：

```dart
class _AppWidgetState extends State<AppWidget> with WidgetsBindingObserver {
  @override
  void initState() {
    super.initState();
    WidgetsBinding.instance.addObserver(this);
  }

  @override
  void didChangeAppLifecycleState(AppLifecycleState state) {
    // state: resumed, inactive, paused, detached
    if (state == AppLifecycleState.paused) {
      // 应用进入后台
    }
  }
}
```

---

## 5. 常用 Widget 速查

### 布局类（≈ HTML/CSS 布局）

```dart
// Column — 垂直排列（≈ flex-direction: column）
Column(
  children: [Widget1(), Widget2(), Widget3()],
)

// Row — 水平排列（≈ flex-direction: row）
Row(
  children: [Widget1(), Widget2()],
)

// Stack — 层叠（≈ position: absolute）
Stack(
  children: [背景Widget(), 前景Widget()],
)

// Expanded — 填充剩余空间（≈ flex: 1）
Row(
  children: [
    Text('标签'),
    Expanded(child: TextField()),  // 填满剩余宽度
  ],
)

// Padding — 内边距
Padding(
  padding: EdgeInsets.all(16),
  child: Text('有边距的文本'),
)

// SizedBox — 固定尺寸/间距
SizedBox(height: 16)  // 16px 垂直间距
```

### 滚动列表

```dart
// ListView.builder — 懒加载列表（≈ RecyclerView）
ListView.builder(
  itemCount: items.length,
  itemBuilder: (context, index) {
    return ListTile(title: Text(items[index].name));
  },
)

// GridView.builder — 网格布局
GridView.builder(
  gridDelegate: SliverGridDelegateWithFixedCrossAxisCount(
    crossAxisCount: 3,  // 3 列
  ),
  itemCount: items.length,
  itemBuilder: (context, index) => Card(child: ...),
)
```

### Material 组件

```dart
// Scaffold — 页面骨架（AppBar + Body + FAB + Drawer）
Scaffold(
  appBar: AppBar(title: Text('标题')),
  body: Center(child: Text('内容')),
  floatingActionButton: FloatingActionButton(
    onPressed: () {},
    child: Icon(Icons.add),
  ),
)

// Card — 卡片
Card(
  child: Padding(
    padding: EdgeInsets.all(16),
    child: Text('卡片内容'),
  ),
)
```

---

## 6. BuildContext

`BuildContext` 是 Widget 在树中的位置标识，用于：
- 获取主题：`Theme.of(context)`
- 获取屏幕尺寸：`MediaQuery.of(context).size`
- 导航：`Navigator.of(context).push(...)`
- 显示对话框：`showDialog(context: context, ...)`

```dart
@override
Widget build(BuildContext context) {
  // 获取当前主题色
  final colorScheme = Theme.of(context).colorScheme;

  // 获取屏幕宽度
  final screenWidth = MediaQuery.of(context).size.width;

  return Container(
    color: colorScheme.surface,
    width: screenWidth * 0.8,
  );
}
```

类比 Spring：`BuildContext` ≈ `ApplicationContext`，但作用域是 Widget 树中的位置。

---

## 7. 本项目的 UI 构建模式

### 页面结构三件套

每个页面由三个文件组成：

```
lib/pages/popular/
├── popular_controller.dart   ← 状态管理（≈ @Service）
├── popular_module.dart       ← 路由配置（≈ @Controller 的 @RequestMapping）
└── popular_page.dart         ← UI 组件（≈ 视图层）
```

### 典型页面代码结构

```dart
class _PopularPageState extends State<PopularPage> {
  // 1. 获取依赖
  final controller = Modular.get<PopularController>();

  // 2. 初始化
  @override
  void initState() {
    super.initState();
    controller.loadData();
  }

  // 3. 构建 UI
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: SysAppBar(title: '推荐'),  // 项目自定义 AppBar
      body: Observer(
        builder: (_) => _buildBody(),    // MobX 响应式更新
      ),
    );
  }

  // 4. 拆分子组件
  Widget _buildBody() {
    if (controller.isLoadingMore) return _buildLoading();
    if (controller.trendList.isEmpty) return _buildEmpty();
    return _buildList();
  }
}
```

---

## 8. 热重载（Hot Reload）

Flutter 的杀手级特性：修改代码后，UI 立即更新，不需要重新编译。

```bash
# 运行应用后
# 修改代码 → 保存 → UI 自动更新（< 1 秒）

# 如果热重载失效（修改了 main.dart 或状态结构），用热重启：
# 按 R（大写）或重新运行
```

对比 Java：
- Java: 修改代码 → 重新编译 → 重启应用（30 秒 ~ 几分钟）
- Flutter: 修改代码 → 保存 → 立即看到效果（< 1 秒）

---

## 练习

1. 打开 `lib/main.dart`，找出应用启动时做了哪些初始化（对照 Spring Boot 的 `@Bean` 配置）
2. 打开 `lib/app_widget.dart`，找出 `MaterialApp` 配置了哪些全局属性
3. 在任意一个 `*_page.dart` 文件中，找出 `Observer` widget 包裹了哪些内容

<details>
<summary>答案提示</summary>

1. `main.dart` 初始化了：Flutter 引擎绑定、MediaKit（视频库）、系统 UI 样式、Hive 数据库、窗口管理器、HTTP 客户端、代理管理器
2. `MaterialApp.router` 配置了：主题（light/dark）、路由、本地化、页面过渡动画
3. `Observer` 包裹的内容会在其内部引用的 `@observable` 变量变化时自动重建

</details>

---

下一章：[项目架构与数据流](./03-PROJECT-ARCHITECTURE.md)
