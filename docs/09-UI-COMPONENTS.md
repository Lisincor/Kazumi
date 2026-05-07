# 第九章：UI 组件与主题系统

> 目标：理解 Kazumi 的主题系统和可复用 UI 组件
> 配套源码：`lib/bean/settings/theme_provider.dart`、`lib/bean/card/`、`lib/bean/appbar/`

---

## 1. Material Design 3 主题

Kazumi 使用 Flutter 的 Material Design 3 主题系统，支持：
- 浅色/深色/跟随系统
- 9 种预设主题色
- Material You 动态取色（Android 12+）
- OLED 纯黑模式

### ThemeProvider

`lib/bean/settings/theme_provider.dart` — 主题状态管理：

```dart
class ThemeProvider extends ChangeNotifier {
  ThemeMode _themeMode = ThemeMode.system;
  ThemeData _lightTheme;
  ThemeData _darkTheme;

  // 切换主题模式
  void setThemeMode(ThemeMode mode) {
    _themeMode = mode;
    GStorage.setting.put(SettingBoxKey.themeMode, mode.index);
    notifyListeners();  // 通知所有监听者重建 UI
  }

  // 切换主题色
  void setThemeColor(Color color) {
    _lightTheme = _buildTheme(Brightness.light, color);
    _darkTheme = _buildTheme(Brightness.dark, color);
    notifyListeners();
  }

  // 判断当前是否为深色模式
  bool isEffectiveDark(BuildContext context) {
    if (_themeMode == ThemeMode.system) {
      return MediaQuery.platformBrightnessOf(context) == Brightness.dark;
    }
    return _themeMode == ThemeMode.dark;
  }
}
```

### 在 App 中使用

`lib/app_widget.dart`:
```dart
// Provider 包裹整个应用
ChangeNotifierProvider(
  create: (_) => ThemeProvider(),
  child: ModularApp(module: AppModule(), child: const AppWidget()),
)

// AppWidget 中消费主题
Consumer<ThemeProvider>(
  builder: (context, themeProvider, _) {
    return MaterialApp.router(
      theme: themeProvider.lightTheme,
      darkTheme: themeProvider.darkTheme,
      themeMode: themeProvider.themeMode,
    );
  },
)
```

### 对照 Spring Boot

| Flutter 主题 | Java/Web 对照 |
|-------------|--------------|
| `ThemeProvider` | CSS 变量 / SCSS 主题文件 |
| `Theme.of(context)` | 读取当前主题变量 |
| `ColorScheme` | CSS 调色板 |
| `notifyListeners()` | 触发页面重新渲染 |

---

## 2. 颜色系统

`lib/bean/settings/color_type.dart` — 预设颜色：

```dart
// 9 种预设主题色
enum ColorType {
  green,
  teal,
  blue,
  indigo,
  purple,
  pink,
  yellow,
  orange,
  deepOrange,
}
```

在代码中使用主题色：
```dart
@override
Widget build(BuildContext context) {
  final colorScheme = Theme.of(context).colorScheme;

  return Container(
    color: colorScheme.surface,           // 背景色
    child: Text(
      '标题',
      style: TextStyle(color: colorScheme.onSurface),  // 文字色
    ),
  );
}
```

### ColorScheme 常用属性

| 属性 | 用途 |
|------|------|
| `primary` | 主色调（按钮、链接） |
| `onPrimary` | 主色调上的文字颜色 |
| `surface` | 卡片/页面背景 |
| `onSurface` | 背景上的文字颜色 |
| `secondary` | 辅助色 |
| `error` | 错误提示色 |
| `outline` | 边框色 |

---

## 3. 自定义 AppBar

`lib/bean/appbar/sys_app_bar.dart` — 跨平台 AppBar：

```dart
class SysAppBar extends StatelessWidget implements PreferredSizeWidget {
  final Widget? title;
  final List<Widget>? actions;
  final Widget? leading;

  // 处理平台差异：
  // - 桌面端：支持窗口拖拽
  // - macOS：标题居中，适配刘海屏
  // - Windows/Linux：自定义标题栏按钮
}
```

使用方式：
```dart
Scaffold(
  appBar: SysAppBar(title: Text('推荐')),
  body: ...,
)
```

---

## 4. 卡片组件

### BangumiCardV — 番剧卡片

`lib/bean/card/bangumi_card.dart` — 垂直布局的番剧卡片：

```dart
class BangumiCardV extends StatelessWidget {
  final BangumiItem bangumiItem;
  final bool canEdit;

  @override
  Widget build(BuildContext context) {
    return Card(
      child: Column(
        children: [
          // 封面图（带 Hero 动画）
          Hero(
            tag: bangumiItem.id,
            child: NetworkImgLayer(src: bangumiItem.imageUrl),
          ),
          // 标题
          Text(bangumiItem.nameCn.isNotEmpty
              ? bangumiItem.nameCn
              : bangumiItem.name),
        ],
      ),
    );
  }
}
```

### NetworkImgLayer — 网络图片

`lib/bean/card/network_img_layer.dart` — 带缓存和占位符的网络图片：

```dart
class NetworkImgLayer extends StatelessWidget {
  final String src;
  final double? width;
  final double? height;

  // 功能：
  // - 自动缓存（CachedNetworkImage）
  // - 加载中显示占位动画
  // - 加载失败显示默认图片
  // - 根据尺寸自动调整缓存大小
}
```

---

## 5. 对话框系统

`lib/bean/dialog/dialog_helper.dart` — 统一的对话框管理：

```dart
class KazumiDialog {
  // 显示 Toast 提示
  static void showToast({required String message}) {
    // 底部浮动提示
  }

  // 显示加载对话框
  static void showLoading({String? message}) {
    showDialog(
      context: context,
      builder: (_) => Center(child: CircularProgressIndicator()),
    );
  }

  // 关闭对话框
  static void dismiss() {
    Navigator.of(context).pop();
  }
}
```

使用方式：
```dart
// 显示提示
KazumiDialog.showToast(message: '收藏成功');

// 显示加载
KazumiDialog.showLoading(message: '加载中...');
// ... 异步操作
KazumiDialog.dismiss();
```

---

## 6. 响应式布局

### 断点定义

`lib/utils/constants.dart`:
```dart
// 屏幕尺寸断点
static const double compactWidth = 600;   // 手机
static const double mediumWidth = 840;    // 平板

// 判断方法
static bool isCompact(BuildContext context) =>
    MediaQuery.of(context).size.width < compactWidth;

static bool isDesktop() =>
    Platform.isWindows || Platform.isMacOS || Platform.isLinux;
```

### 响应式网格

```dart
// 根据屏幕宽度决定列数
int getCrossAxisCount(BuildContext context) {
  final width = MediaQuery.of(context).size.width;
  if (width < 600) return 3;       // 手机：3 列
  if (width < 900) return 4;       // 平板：4 列
  if (width < 1200) return 5;      // 小桌面：5 列
  return 6;                         // 大桌面：6 列
}

GridView.builder(
  gridDelegate: SliverGridDelegateWithFixedCrossAxisCount(
    crossAxisCount: getCrossAxisCount(context),
    childAspectRatio: 0.65,  // 宽高比
  ),
  itemBuilder: (_, index) => BangumiCardV(item: items[index]),
)
```

---

## 7. 平台适配

### 条件渲染

```dart
Widget build(BuildContext context) {
  if (Platform.isAndroid || Platform.isIOS) {
    return _buildMobileLayout();
  } else {
    return _buildDesktopLayout();
  }
}
```

### 平台特定功能

```dart
// 桌面端：窗口拖拽区域
if (Utils.isDesktop()) {
  GestureDetector(
    onPanStart: (_) => windowManager.startDragging(),
    child: appBarContent,
  );
}

// Android：全屏沉浸式
if (Platform.isAndroid) {
  SystemChrome.setEnabledSystemUIMode(SystemUiMode.immersiveSticky);
}

// macOS：标题栏高度适配
if (Platform.isMacOS) {
  toolbarHeight = kToolbarHeight + 22;  // 额外 22px 给系统按钮
}
```

---

## 8. 常用 Widget 模式

### 加载状态三态

```dart
Observer(
  builder: (_) {
    // 加载中
    if (controller.isLoading && controller.items.isEmpty) {
      return Center(child: CircularProgressIndicator());
    }
    // 空状态
    if (controller.items.isEmpty) {
      return GeneralErrorWidget(
        message: '暂无数据',
        onRetry: controller.refresh,
      );
    }
    // 正常内容
    return ListView.builder(...);
  },
)
```

### Hero 动画（页面间共享元素过渡）

```dart
// 列表页
Hero(
  tag: 'bangumi_${item.id}',
  child: NetworkImgLayer(src: item.imageUrl),
)

// 详情页（同一个 tag，Flutter 自动做过渡动画）
Hero(
  tag: 'bangumi_${item.id}',
  child: NetworkImgLayer(src: item.imageUrl),
)
```

### 下拉刷新

```dart
RefreshIndicator(
  onRefresh: () async {
    await controller.refresh();
  },
  child: ListView.builder(...),
)
```

---

## 练习

1. 打开 `lib/bean/settings/theme_provider.dart`，找出支持哪些主题模式
2. 打开任意一个 `*_page.dart`，找出它如何使用 `Theme.of(context)` 获取颜色
3. 如果你要创建一个新的卡片组件（比如"评论卡片"），应该放在哪个目录？需要哪些属性？

<details>
<summary>答案</summary>

1. 支持 ThemeMode.system（跟随系统）、ThemeMode.light（浅色）、ThemeMode.dark（深色）
2. 通常通过 `Theme.of(context).colorScheme.primary` 等方式获取颜色
3. 放在 `lib/bean/card/` 目录，需要评论内容、用户名、时间等属性，继承 StatelessWidget

</details>

---

下一章：[视频播放与弹幕](./10-VIDEO-PLAYER.md)
