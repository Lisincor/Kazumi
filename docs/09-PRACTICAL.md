# 第九课：实战 - 开发新页面

> 📖 学习目标：通过实际案例完整地开发一个新功能，从需求分析到代码实现
> ⏱️ 预计时间：3-4 小时
> 📁 配套源码：`lib/pages/popular/` 完整示例

---

## 目录

1. [需求分析](#1-需求分析)
2. [创建页面结构](#2-创建页面结构)
3. [编写 Controller](#3-编写-controller)
4. [编写 Module](#4-编写-module)
5. [编写 Page](#5-编写-page)
6. [注册路由](#6-注册路由)
7. [测试与调试](#7-测试与调试)
8. [完整案例：搜索历史页面](#8-完整案例搜索历史页面)

---

## 1. 需求分析

### 1.1 我们要做什么？

创建一个"设置"页面，包含：
- 主题切换（浅色/深色/跟随系统）
- 字体切换（应用字体/系统字体）
- 代理设置
- 关于页面

### 1.2 页面结构

```
设置页面 (/settings)
├── 外观设置
│   ├── 主题模式
│   ├── 主题色
│   └── 字体
├── 网络设置
│   ├── 代理开关
│   └── 代理地址
└── 关于
    ├── 版本信息
    └── 反馈
```

### 1.3 技术要点

- 使用 MobX 管理状态
- 使用 Hive 存储设置
- 使用 Flutter Modular 路由
- 使用 Provider 管理主题

---

## 2. 创建页面结构

### 2.1 目录结构

```
lib/pages/settings/
├── settings_controller.dart       ← 状态管理
├── settings_controller.g.dart    ← 代码生成
├── settings_module.dart          ← 路由配置
└── settings_page.dart           ← 页面 UI
```

### 2.2 创建步骤

```bash
# 创建目录（如果不存在）
mkdir -p lib/pages/settings
```

### 2.3 文件模板

#### settings_controller.dart 模板

```dart
import 'package:mobx/mobx.dart';
import 'package:kazumi/utils/storage.dart';

part 'settings_controller.g.dart';

class SettingsController = _SettingsController with _$SettingsController;

abstract class _SettingsController with Store {
  Box setting = GStorage.setting;

  @observable
  String themeMode = 'system';

  @observable
  String themeColor = 'default';

  @observable
  bool useSystemFont = false;

  @observable
  bool proxyEnable = false;

  @observable
  String proxyUrl = '';

  // ==================== 主题设置 ====================

  @action
  void setThemeMode(String mode) {
    themeMode = mode;
    setting.put(SettingBoxKey.themeMode, mode);
  }

  @action
  void setThemeColor(String color) {
    themeColor = color;
    setting.put(SettingBoxKey.themeColor, color);
  }

  @action
  void setUseSystemFont(bool value) {
    useSystemFont = value;
    setting.put(SettingBoxKey.useSystemFont, value);
  }

  // ==================== 代理设置 ====================

  @action
  void setProxyEnable(bool value) {
    proxyEnable = value;
    setting.put(SettingBoxKey.proxyEnable, value);
  }

  @action
  void setProxyUrl(String url) {
    proxyUrl = url;
    setting.put(SettingBoxKey.proxyUrl, url);
  }

  // ==================== 初始化 ====================

  void loadSettings() {
    themeMode = setting.get(SettingBoxKey.themeMode, defaultValue: 'system');
    themeColor = setting.get(SettingBoxKey.themeColor, defaultValue: 'default');
    useSystemFont = setting.get(SettingBoxKey.useSystemFont, defaultValue: false);
    proxyEnable = setting.get(SettingBoxKey.proxyEnable, defaultValue: false);
    proxyUrl = setting.get(SettingBoxKey.proxyUrl, defaultValue: '');
  }
}
```

#### settings_module.dart 模板

```dart
import 'package:flutter_modular/flutter_modular.dart';
import 'package:kazumi/pages/settings/settings_controller.dart';
import 'package:kazumi/pages/settings/settings_page.dart';

class SettingsModule extends Module {
  @override
  void routes(r) {
    r.child("/settings", child: (_) => SettingsPage());
  }
}
```

#### settings_page.dart 模板

```dart
import 'package:flutter/material.dart';
import 'package:flutter_modular/flutter_modular.dart';
import 'package:flutter_mobx/flutter_mobx.dart';
import 'package:kazumi/pages/settings/settings_controller.dart';
import 'package:kazumi/request/api.dart';

class SettingsPage extends StatefulWidget {
  const SettingsPage({super.key});

  @override
  State<SettingsPage> createState() => _SettingsPageState();
}

class _SettingsPageState extends State<SettingsPage> {
  final SettingsController controller = Modular.get<SettingsController>();

  @override
  void initState() {
    super.initState();
    controller.loadSettings();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('设置'),
      ),
      body: Observer(
        builder: (_) {
          return ListView(
            children: [
              // 外观设置
              _buildSectionHeader('外观设置'),
              _buildThemeModeTile(),
              _buildThemeColorTile(),
              _buildSystemFontTile(),

              const Divider(),

              // 网络设置
              _buildSectionHeader('网络设置'),
              _buildProxyEnableTile(),
              if (controller.proxyEnable) _buildProxyUrlTile(),

              const Divider(),

              // 关于
              _buildSectionHeader('关于'),
              ListTile(
                leading: const Icon(Icons.info),
                title: const Text('版本'),
                subtitle: Text(Api.version),
              ),
            ],
          );
        },
      ),
    );
  }

  Widget _buildSectionHeader(String title) {
    return Padding(
      padding: const EdgeInsets.fromLTRB(16, 16, 16, 8),
      child: Text(
        title,
        style: Theme.of(context).textTheme.titleSmall?.copyWith(
          color: Theme.of(context).colorScheme.primary,
        ),
      ),
    );
  }

  Widget _buildThemeModeTile() {
    return ListTile(
      leading: const Icon(Icons.palette),
      title: const Text('主题模式'),
      subtitle: Text(_getThemeModeText()),
      onTap: () => _showThemeModeDialog(),
    );
  }

  String _getThemeModeText() {
    switch (controller.themeMode) {
      case 'dark': return '深色';
      case 'light': return '浅色';
      default: return '跟随系统';
    }
  }

  Future<void> _showThemeModeDialog() async {
    final result = await showDialog<String>(
      context: context,
      builder: (context) => AlertDialog(
        title: const Text('选择主题模式'),
        content: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            RadioListTile<String>(
              title: const Text('跟随系统'),
              value: 'system',
              groupValue: controller.themeMode,
              onChanged: (v) => Navigator.pop(context, v),
            ),
            RadioListTile<String>(
              title: const Text('浅色'),
              value: 'light',
              groupValue: controller.themeMode,
              onChanged: (v) => Navigator.pop(context, v),
            ),
            RadioListTile<String>(
              title: const Text('深色'),
              value: 'dark',
              groupValue: controller.themeMode,
              onChanged: (v) => Navigator.pop(context, v),
            ),
          ],
        ),
      ),
    );

    if (result != null) {
      controller.setThemeMode(result);
    }
  }

  Widget _buildSystemFontTile() {
    return SwitchListTile(
      secondary: const Icon(Icons.font_download),
      title: const Text('使用系统字体'),
      value: controller.useSystemFont,
      onChanged: controller.setUseSystemFont,
    );
  }

  Widget _buildProxyEnableTile() {
    return SwitchListTile(
      secondary: const Icon(Icons.vpn_key),
      title: const Text('启用代理'),
      value: controller.proxyEnable,
      onChanged: controller.setProxyEnable,
    );
  }

  Widget _buildProxyUrlTile() {
    return ListTile(
      leading: const Icon(Icons.link),
      title: const Text('代理地址'),
      subtitle: Text(controller.proxyUrl),
      onTap: () => _showProxyUrlDialog(),
    );
  }

  Future<void> _showProxyUrlDialog() async {
    final textController = TextEditingController(text: controller.proxyUrl);

    final result = await showDialog<String>(
      context: context,
      builder: (context) => AlertDialog(
        title: const Text('代理地址'),
        content: TextField(
          controller: textController,
          decoration: const InputDecoration(
            hintText: 'http://127.0.0.1:7890',
            helperText: '支持 HTTP 代理',
          ),
        ),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context),
            child: const Text('取消'),
          ),
          FilledButton(
            onPressed: () => Navigator.pop(context, textController.text),
            child: const Text('确定'),
          ),
        ],
      ),
    );

    if (result != null) {
      controller.setProxyUrl(result);
    }
  }
}
```

---

## 3. 编写 Controller

📁 完整源码参考：`lib/pages/popular/popular_controller.dart`

### 3.1 Controller 设计原则

1. **单一职责**：一个 Controller 只管理一个页面的状态
2. **响应式**：使用 `@observable` 标记需要响应式更新的变量
3. **动作化**：使用 `@action` 标记会修改状态的方法
4. **依赖注入**：通过 `Modular.get<Controller>()` 获取实例

### 3.2 常用模式

#### 列表加载

```dart
@observable
ObservableList<Item> items = ObservableList();

@observable
bool isLoading = false;

@observable
bool hasMore = true;

@action
Future<void> loadMore() async {
  if (isLoading || !hasMore) return;

  isLoading = true;
  try {
    final result = await fetchItems();
    items.addAll(result);
    hasMore = result.length >= pageSize;
  } catch (e) {
    // 错误处理
  }
  isLoading = false;
}
```

#### 搜索

```dart
@observable
String searchKeyword = '';

@action
Future<void> search(String keyword) async {
  searchKeyword = keyword;
  if (keyword.isEmpty) {
    items.clear();
    return;
  }
  isLoading = true;
  try {
    final result = await _searchAPI(keyword);
    items.clear();
    items.addAll(result);
  } catch (e) {
    // 错误处理
  }
  isLoading = false;
}
```

#### 表单处理

```dart
@observable
String inputValue = '';

@action
void updateInput(String value) {
  inputValue = value;
}

@action
Future<void> submit() async {
  if (inputValue.isEmpty) return;

  isSubmitting = true;
  try {
    await _saveAPI(inputValue);
    inputValue = '';
  } catch (e) {
    // 错误处理
  }
  isSubmitting = false;
}
```

---

## 4. 编写 Module

📁 完整源码参考：`lib/pages/popular/popular_module.dart`

### 4.1 Module 模板

```dart
import 'package:flutter_modular/flutter_modular.dart';
import 'package:kazumi/pages/settings/settings_controller.dart';
import 'package:kazumi/pages/settings/settings_page.dart';

class SettingsModule extends Module {
  @override
  void routes(r) {
    // 子路由
    r.child("/settings", child: (_) => SettingsPage());

    // 可以嵌套更多路由
    r.child("/settings/about", child: (_) => AboutPage());
  }
}
```

### 4.2 嵌套 Module

```dart
class ParentModule extends Module {
  @override
  void routes(r) {
    r.module("/child", module: ChildModule());
  }
}

class ChildModule extends Module {
  @override
  void routes(r) {
    r.child("/page", child: (_) => ChildPage());
  }
}
```

---

## 5. 编写 Page

📁 完整源码参考：`lib/pages/popular/popular_page.dart`

### 5.1 Page 结构

```dart
class MyPage extends StatefulWidget {
  // 使用 StatefulWidget 以支持生命周期
  const MyPage({super.key});

  @override
  State<MyPage> createState() => _MyPageState();
}

class _MyPageState extends State<MyPage> {
  // 获取 Controller
  final controller = Modular.get<MyController>();

  @override
  void initState() {
    super.initState();
    // 初始化时加载数据
    controller.loadData();
  }

  @override
  void dispose() {
    // 清理资源
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('标题')),
      body: Observer(
        builder: (_) {
          // 使用 Observer 监听状态变化
          if (controller.isLoading) {
            return const Center(child: CircularProgressIndicator());
          }
          return _buildContent();
        },
      ),
    );
  }

  Widget _buildContent() {
    // 实际内容
  }
}
```

### 5.2 常用组件

#### AppBar

```dart
Scaffold(
  appBar: AppBar(
    title: const Text('标题'),
    actions: [
      IconButton(icon: Icon(Icons.refresh), onPressed: controller.refresh),
    ],
  ),
);
```

#### ListView

```dart
ListView.builder(
  itemCount: controller.items.length,
  itemBuilder: (context, index) {
    return ListTile(
      title: Text(controller.items[index].name),
      onTap: () => _onItemTap(controller.items[index]),
    );
  },
)
```

#### GridView

```dart
GridView.builder(
  gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
    crossAxisCount: 3,  // 列数
    mainAxisSpacing: 8, // 间距
    crossAxisSpacing: 8,
  ),
  itemCount: controller.items.length,
  itemBuilder: (context, index) {
    return _buildGridItem(controller.items[index]);
  },
)
```

#### 加载状态

```dart
Observer(
  builder: (_) {
    if (controller.isLoading && controller.items.isEmpty) {
      return const Center(child: CircularProgressIndicator());
    }
    if (controller.items.isEmpty) {
      return const Center(child: Text('暂无数据'));
    }
    return ListView(...);
  },
)
```

### 5.3 导航

```dart
// 跳转
Modular.to.pushNamed('/path');

// 跳转并传递参数
Modular.to.pushNamed('/path', arguments: data);

// 返回
Modular.to.pop();

// 替换当前页
Modular.to.pushReplacementNamed('/path');
```

---

## 6. 注册路由

### 6.1 在 IndexModule 中注册

📁 修改 `lib/pages/index_module.dart`

```dart
import 'package:kazumi/pages/settings/settings_module.dart';

class IndexModule extends Module {
  @override
  void routes(r) {
    // ... 其他路由

    r.module("/settings", module: SettingsModule());
  }
}
```

### 6.2 添加到菜单（可选）

📁 修改 `lib/pages/router.dart`

```dart
final MenuRoute menu = MenuRoute([
  // ... 其他菜单项
  MenuRouteItem(path: "/settings", module: SettingsModule()),
]);
```

---

## 7. 测试与调试

### 7.1 运行代码生成

```bash
# 生成 MobX 代码
dart run build_runner build

# 如果有冲突
dart run build_runner build --delete-conflicting-outputs
```

### 7.2 热重载

```bash
# 在终端按 r 键
# 或保存文件后自动热重载
```

### 7.3 断点调试

```dart
// 在 VS Code 中点击行号左侧设置断点
@action
Future<void> loadData() async {
  // 断点停在这里
  isLoading = true;
  // ...
}
```

### 7.4 日志调试

```dart
import 'package:kazumi/utils/logger.dart';

// 打印日志
KazumiLogger().i('Info: $variable');
KazumiLogger().w('Warning: $error');
KazumiLogger().e('Error: $error', error: e);
```

### 7.5 检查代码风格

```bash
flutter analyze
```

---

## 8. 完整案例：搜索历史页面

### 8.1 需求

创建一个搜索历史页面，显示用户的搜索记录，支持：
- 显示历史列表
- 点击跳转搜索
- 删除单条记录
- 清空全部记录

### 8.2 创建文件

**search_history_controller.dart**

```dart
import 'package:mobx/mobx.dart';
import 'package:kazumi/modules/search/search_history_module.dart';
import 'package:kazumi/utils/storage.dart';
import 'package:flutter_modular/flutter_modular.dart';

part 'search_history_controller.g.dart';

class SearchHistoryController = _SearchHistoryController with _$SearchHistoryController;

abstract class _SearchHistoryController with Store {
  Box setting = GStorage.setting;

  @observable
  ObservableList<SearchHistory> historyList = ObservableList();

  void loadHistory() {
    historyList.clear();
    final items = setting.get('searchHistory', defaultValue: <SearchHistory>[]);
    historyList.addAll(List<SearchHistory>.from(items));
  }

  @action
  Future<void> addHistory(String keyword) async {
    // 移除重复
    historyList.removeWhere((h) => h.keyword == keyword);

    // 添加到列表头部
    historyList.insert(0, SearchHistory(keyword: keyword));

    // 限制最大数量
    while (historyList.length > 50) {
      historyList.removeLast();
    }

    // 保存
    await _saveHistory();
  }

  @action
  Future<void> removeHistory(SearchHistory item) async {
    historyList.remove(item);
    await _saveHistory();
  }

  @action
  Future<void> clearAll() async {
    historyList.clear();
    await _saveHistory();
  }

  Future<void> _saveHistory() async {
    await setting.put('searchHistory', historyList.toList());
  }
}
```

**search_history_module.dart**

```dart
import 'package:flutter_modular/flutter_modular.dart';
import 'package:kazumi/pages/search/search_history_controller.dart';
import 'package:kazumi/pages/search/search_history_page.dart';

class SearchHistoryModule extends Module {
  @override
  void routes(r) {
    r.child("/history", child: (_) => SearchHistoryPage());
  }
}
```

**search_history_page.dart**

```dart
import 'package:flutter/material.dart';
import 'package:flutter_modular/flutter_modular.dart';
import 'package:flutter_mobx/flutter_mobx.dart';
import 'package:kazumi/pages/search/search_history_controller.dart';

class SearchHistoryPage extends StatefulWidget {
  const SearchHistoryPage({super.key});

  @override
  State<SearchHistoryPage> createState() => _SearchHistoryPageState();
}

class _SearchHistoryPageState extends State<SearchHistoryPage> {
  final controller = Modular.get<SearchHistoryController>();

  @override
  void initState() {
    super.initState();
    controller.loadHistory();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('搜索历史'),
        actions: [
          Observer(
            builder: (_) => controller.historyList.isEmpty
                ? const SizedBox()
                : IconButton(
                    icon: const Icon(Icons.delete_sweep),
                    onPressed: () => _confirmClear(),
                    tooltip: '清空全部',
                  ),
          ),
        ],
      ),
      body: Observer(
        builder: (_) {
          if (controller.historyList.isEmpty) {
            return const Center(
              child: Text('暂无搜索历史'),
            );
          }

          return ListView.builder(
            itemCount: controller.historyList.length,
            itemBuilder: (context, index) {
              final item = controller.historyList[index];
              return ListTile(
                leading: const Icon(Icons.history),
                title: Text(item.keyword),
                trailing: IconButton(
                  icon: const Icon(Icons.close),
                  onPressed: () => controller.removeHistory(item),
                ),
                onTap: () {
                  // 返回搜索页并携带关键词
                  Modular.to.pop(item.keyword);
                },
              );
            },
          );
        },
      ),
    );
  }

  Future<void> _confirmClear() async {
    final confirmed = await showDialog<bool>(
      context: context,
      builder: (context) => AlertDialog(
        title: const Text('清空历史'),
        content: const Text('确定要清空所有搜索历史吗？'),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context, false),
            child: const Text('取消'),
          ),
          FilledButton(
            onPressed: () => Navigator.pop(context, true),
            child: const Text('确定'),
          ),
        ],
      ),
    );

    if (confirmed == true) {
      controller.clearAll();
    }
  }
}
```

### 8.3 注册路由

**在 `lib/pages/search/search_module.dart` 中添加：**

```dart
import 'package:kazumi/pages/search/search_history_module.dart';

class SearchModule extends Module {
  @override
  void routes(r) {
    r.child("/search", child: (_) => SearchPage());
    r.module("/search/history", module: SearchHistoryModule());
  }
}
```

### 8.4 运行代码生成

```bash
dart run build_runner build
```

### 8.5 完整目录结构

```
lib/pages/search/
├── search_controller.dart
├── search_controller.g.dart
├── search_module.dart
├── search_page.dart
├── search_history_controller.dart
├── search_history_controller.g.dart
├── search_history_module.dart
└── search_history_page.dart
```

---

## 小练习

### 练习 1：完善设置页面

在上面的设置页面中添加以下功能：

1. 显示"关于"页面链接
2. 显示"清理缓存"按钮

<details>
<summary>提示</summary>

```dart
Widget _buildAboutTile() {
  return ListTile(
    leading: const Icon(Icons.info),
    title: const Text('关于'),
    trailing: const Icon(Icons.chevron_right),
    onTap: () => Modular.to.pushNamed('/settings/about/'),
  );
}

Widget _buildClearCacheTile() {
  return ListTile(
    leading: const Icon(Icons.cleaning_services),
    title: const Text('清理缓存'),
    subtitle: const Text('清除临时文件和缓存'),
    onTap: () async {
      // 实现清理逻辑
      await clearCache();
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          const SnackBar(content: Text('缓存已清理')),
        );
      }
    },
  );
}
```

</details>

### 练习 2：实现排序功能

为搜索历史页面添加排序功能，支持按时间排序和按字母排序。

<details>
<summary>答案</summary>

在 `SearchHistoryController` 中添加：

```dart
enum SortType { byTime, byAlphabet }

@observable
SortType sortType = SortType.byTime;

@action
void setSortType(SortType type) {
  sortType = type;
  _sortHistory();
}

void _sortHistory() {
  switch (sortType) {
    case SortType.byTime:
      // 按时间排序（已有）
      break;
    case SortType.byAlphabet:
      historyList.sort((a, b) => a.keyword.compareTo(b.keyword));
      break;
  }
}
```

</details>

---

## 总结

恭喜你完成了所有教程的学习！

### 你学到了什么？

1. **Dart 基础** - 语法、数据类型、异步编程
2. **应用启动** - 初始化流程、生命周期
3. **MobX 状态管理** - Observable、Action、Computed
4. **Hive 数据存储** - 模型定义、Box 操作
5. **Dio 网络请求** - GET/POST、拦截器、错误处理
6. **插件系统** - XPath 选择器、搜索和解析
7. **Flutter Modular 路由** - Module、Route、导航
8. **数据模型** - BangumiItem、History、Download 等
9. **页面开发实战** - 完整的功能开发流程

### 下一步

继续学习：
- 单元测试编写
- 性能优化
- 平台特定代码（原生插件）
- 动画与交互设计
- CI/CD 自动化部署

祝你开发愉快！ 🎉