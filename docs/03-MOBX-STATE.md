# 第三课：MobX 状态管理

> 📖 学习目标：理解 MobX 响应式编程，学会创建响应式状态管理类
> ⏱️ 预计时间：2-3 小时
> 📁 配套源码：`lib/pages/popular/popular_controller.dart`、`lib/pages/collect/collect_controller.dart`

---

## 目录

1. [什么是 MobX？](#1-什么是-mobx)
2. [核心概念](#2-核心概念)
3. [注解详解](#3-注解详解)
4. [Controller 编写模式](#4-controller-编写模式)
5. [在页面中使用](#5-在页面中使用)
6. [实际案例分析](#6-实际案例分析)
7. [代码生成](#7-代码生成)
8. [常见问题](#8-常见问题)

---

## 1. 什么是 MobX？

### 1.1 类比理解

想象你在厨房做饭：
- **传统方式**：你需要每隔几秒去检查锅里的水开了没有
- **MobX 方式**：你设置一个警报，水开了警报自动响

```
传统：主动轮询
    for (;;) {
      if (dataChanged) { updateUI(); }
    }

MobX：响应式通知
    when(dataChanged) → notify UI
```

### 1.2 为什么用 MobX？

| 对比项 | 原生 setState | MobX |
|--------|-------------|------|
| 代码量 | 多 | 少 |
| 性能 | 一般 | 优秀（精准更新） |
| 学习曲线 | 低 | 中 |
| 适用场景 | 简单页面 | 复杂应用 |

### 1.3 在本项目中的应用

📁 几乎所有页面控制器都使用 MobX：

- `lib/pages/popular/popular_controller.dart` - 推荐页
- `lib/pages/collect/collect_controller.dart` - 收藏页
- `lib/pages/my/my_controller.dart` - 我的页
- `lib/pages/info/info_controller.dart` - 番剧详情页

---

## 2. 核心概念

### 2.1 Observable（可观察状态）

```dart
// 被 @observable 标记的变量会"通知" UI 更新
@observable
String name = 'Kazumi';

@observable
int count = 0;

@observable
ObservableList<BangumiItem> bangumiList = ObservableList.of([]);
```

📁 源码参考：`lib/pages/popular/popular_controller.dart:14-21`

```dart
abstract class _PopularController with Store {
  @observable
  String currentTag = '';

  @observable
  ObservableList<BangumiItem> bangumiList = ObservableList.of([]);

  @observable
  ObservableList<BangumiItem> trendList = ObservableList.of([]);

  @observable
  bool isLoadingMore = false;

  @observable
  bool isTimeOut = false;
}
```

### 2.2 Action（动作/方法）

```dart
// @action 标记的方法会修改状态
@action
void setCurrentTag(String s) {
  currentTag = s;
}

@action
void clearBangumiList() {
  bangumiList.clear();
}

@action
Future<void> queryBangumi() async {
  isLoadingMore = true;
  var result = await fetchData();
  bangumiList.addAll(result);
  isLoadingMore = false;
}
```

📁 源码参考：`lib/pages/popular/popular_controller.dart:31-62`

```dart
@action
void setCurrentTag(String s) {
  currentTag = s;
}

@action
Future<void> queryBangumiByTag({String type = 'add'}) async {
  if (type == 'init') {
    bangumiList.clear();
  }
  isLoadingMore = true;
  int randomNumber = Random().nextInt(8000) + 1;
  var tag = currentTag;
  var result = await BangumiHTTP.getBangumiList(rank: randomNumber, tag: tag);
  bangumiList.addAll(result);
  isLoadingMore = false;
  isTimeOut = bangumiList.isEmpty;
}
```

### 2.3 Computed（计算属性）

```dart
// 基于已有状态计算出的新值
@computed
int get itemCount => bangumiList.length;

@computed
bool get hasData => bangumiList.isNotEmpty;

@computed
bool get isEmpty => !hasData;
```

---

## 3. 注解详解

### 3.1 @observable

**作用**：标记一个变量为"可观察的"

**使用场景**：
- 任何需要 UI 响应变化的数据
- 用户输入、服务器返回、计算结果

**示例**：

```dart
@observable
String searchKeyword = '';

@observable
bool isLoading = false;

@observable
ObservableList<VideoItem> videos = ObservableList.of([]);

@observable
int currentPage = 1;

// 不是所有变量都需要 @observable
double scrollOffset = 0.0;  // 不需要响应式的中间变量
```

📁 源码参考：`lib/pages/my/my_controller.dart:15-16`

```dart
abstract class _MyController with Store {
  Box setting = GStorage.setting;

  @observable
  ObservableList<String> shieldList = ObservableList.of([]);
  // ...
}
```

### 3.2 @action

**作用**：标记一个方法会修改状态

**规则**：
- 所有修改 `@observable` 变量的方法都应加 `@action`
- 建议所有公共方法都加 `@action`

**异步 action**：

```dart
@action
Future<void> loadData() async {
  isLoading = true;
  try {
    data = await fetchData();
  } catch (e) {
    error = e;
  }
  isLoading = false;
}
```

📁 源码参考：`lib/pages/collect/collect_controller.dart:49-76`

```dart
@action
Future<void> addCollect(BangumiItem bangumiItem, {type = 1}) async {
  if (type == 0) {
    await deleteCollect(bangumiItem);
    return;
  }

  final bool syncSucceeded = await _syncBangumiCollectIfEnabled(...);

  final int currentCollectType = getCollectType(bangumiItem);
  final int collectChangeAction = currentCollectType == 0 ? 1 : 2;

  await _collectCrudRepository.addCollectible(bangumiItem, type);
  await GStorage.appendCollectChange(
    bangumiId: bangumiItem.id,
    action: collectChangeAction,
    type: type,
  );
  loadCollectibles();
}
```

### 3.3 @computed

**作用**：基于已有状态计算出的派生值

**示例**：

```dart
@observable
int page = 1;

@observable
int pageSize = 20;

@observable
ObservableList<Item> allItems = ObservableList.of([]);

@computed
int get totalPages => (allItems.length / pageSize).ceil();

@computed
bool get hasMore => page < totalPages;

@computed
List<Item> get currentItems {
  final start = (page - 1) * pageSize;
  final end = start + pageSize;
  return allItems.sublist(
    start,
    end > allItems.length ? allItems.length : end,
  );
}
```

### 3.4 @observable.warp

用于标记数组/列表中的元素可被单独观察：

```dart
@observable
ObservableList<BangumiItem> bangumiList = ObservableList.of([]);

// 在页面中使用 Observer 包裹列表项
ListView.builder(
  itemBuilder: (context, index) {
    return Observer(
      builder: (_) => BangumiCard(item: bangumiList[index]),
    );
  },
)
```

---

## 4. Controller 编写模式

### 4.1 标准模板

📁 源码模板：`lib/pages/popular/popular_controller.dart`

```dart
import 'package:mobx/mobx.dart';

// 代码生成文件
part 'xxx_controller.g.dart';

// 公共类名 = 下划线类名 + _$下划线类名
class XxxController = _XxxController with _$XxxController;

// 下划线类名，使用 with Store 混入
abstract class _XxxController with Store {
  // ===== 依赖注入 =====
  final _repository = Modular.get<IXxxRepository>();

  // ===== Observable 状态 =====
  @observable
  int currentPage = 1;

  @observable
  bool isLoading = false;

  @observable
  ObservableList<Item> items = ObservableList.of([]);

  // ===== Computed =====
  @computed
  bool get hasMore => items.length >= pageSize;

  // ===== Action 方法 =====
  @action
  Future<void> loadMore() async {
    if (isLoading || !hasMore) return;

    isLoading = true;
    try {
      var result = await _repository.fetchItems(
        page: currentPage,
        pageSize: pageSize,
      );
      items.addAll(result);
      currentPage++;
    } catch (e) {
      // 错误处理
    }
    isLoading = false;
  }

  @action
  void refresh() {
    currentPage = 1;
    items.clear();
    loadMore();
  }

  @action
  void deleteItem(Item item) {
    items.remove(item);
    _repository.delete(item.id);
  }
}
```

### 4.2 依赖注入

```dart
// 在 IndexModule 中注册
i.addSingleton(PopularController.new);

// 在 Controller 中获取
abstract class _XxxController with Store {
  // 方式1：直接获取
  final _service = Modular.get<MyService>();

  // 方式2：构造方法注入
  final MyService service;
  _XxxController(this.service);

  // 方式3：延迟获取（推荐，用于避免循环依赖）
  MyService get _service {
    if (_cachedService == null) {
      _cachedService = Modular.get<MyService>();
    }
    return _cachedService!;
  }
  MyService? _cachedService;
}
```

📁 源码参考：`lib/pages/collect/collect_controller.dart:29-31`

```dart
abstract class _CollectController with Store {
  // 直接获取 Repository
  final _collectCrudRepository = Modular.get<ICollectCrudRepository>();
  final _collectRepository = Modular.get<ICollectRepository>();

  Box setting = GStorage.setting;
}
```

### 4.3 状态持久化

如果你需要在应用重启后恢复状态：

```dart
abstract class _SettingsController with Store {
  // 加载时从存储读取
  void loadFromStorage() {
    setting1 = GStorage.setting.get(SettingBoxKey.setting1, defaultValue: '');
    setting2 = GStorage.setting.get(SettingBoxKey.setting2, defaultValue: 0);
  }

  @action
  void updateSetting(String value) async {
    setting1 = value;
    await GStorage.setting.put(SettingBoxKey.setting1, value);
  }
}
```

---

## 5. 在页面中使用

### 5.1 引入 Controller

```dart
import 'package:flutter/material.dart';
import 'package:flutter_modular/flutter_modular.dart';
import 'package:flutter_mobx/flutter_mobx.dart';
import 'package:kazumi/pages/popular/popular_controller.dart';

class PopularPage extends StatelessWidget {
  // 获取 Controller 实例
  final controller = Modular.get<PopularController>();

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('推荐')),
      body: Observer(
        builder: (_) {
          // Observer 会自动监听 controller 的变化
          if (controller.isLoading) {
            return Center(child: CircularProgressIndicator());
          }
          return ListView.builder(
            itemCount: controller.bangumiList.length,
            itemBuilder: (context, index) {
              return BangumiCard(item: controller.bangumiList[index]);
            },
          );
        },
      ),
    );
  }
}
```

### 5.2 Observer 详解

```dart
// 单个字段监听
Observer(
  builder: (_) => Text('${controller.count}'),
)

// 多个字段监听（自动合并，只在需要时重建）
Observer(
  builder: (_) {
    // 这个函数会在任一字段变化时重新执行
    return Column(
      children: [
        Text('Count: ${controller.count}'),
        Text('Name: ${controller.name}'),
        if (controller.isLoading) CircularProgressIndicator(),
      ],
    );
  },
)

// 精确监听列表（性能优化）
Observer(
  builder: (_) {
    // 只监听长度变化
    return ListView.builder(
      itemCount: controller.items.length,  // ← 监听这个
      itemBuilder: (_, index) => ItemWidget(controller.items[index]),
    );
  },
)
```

### 5.3 Provider vs MobX

```dart
// Provider 用法（简单场景）
class MyPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Consumer<ThemeProvider>(
      builder: (context, theme, _) {
        return Text('Theme: ${theme.themeMode}');
      },
    );
  }
}

// MobX 用法（复杂场景）
class MyPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final controller = Modular.get<MyController>();

    return Observer(
      builder: (_) {
        return Column(
          children: [
            Text('Count: ${controller.count}'),
            ElevatedButton(
              onPressed: controller.increment,  // 直接调用 action
              child: Text('Add'),
            ),
          ],
        );
      },
    );
  }
}
```

---

## 6. 实际案例分析

### 案例 1：PopularController（推荐页）

📁 源码：`lib/pages/popular/popular_controller.dart`

```dart
// 完整代码解析
abstract class _PopularController with Store {
  // 滚动控制器（不需要响应式）
  final ScrollController scrollController = ScrollController();

  // ===== Observable 状态 =====
  @observable
  String currentTag = '';

  @observable
  ObservableList<BangumiItem> bangumiList = ObservableList.of([]);

  @observable
  ObservableList<BangumiItem> trendList = ObservableList.of([]);

  @observable
  bool isLoadingMore = false;

  @observable
  bool isTimeOut = false;

  // ===== 普通字段（不需要响应式）=====
  double scrollOffset = 0.0;  // 仅作为临时变量

  // ===== Action 方法 =====
  @action
  void setCurrentTag(String s) {
    currentTag = s;
  }

  @action
  void clearBangumiList() {
    bangumiList.clear();
  }

  @action
  Future<void> queryBangumiByTrend({String type = 'add'}) async {
    if (type == 'init') {
      trendList.clear();
    }
    isLoadingMore = true;
    var result = await BangumiHTTP.getBangumiTrendsList(
      offset: trendList.length,
    );
    trendList.addAll(result);
    isLoadingMore = false;
    isTimeOut = trendList.isEmpty;
  }

  @action
  Future<void> queryBangumiByTag({String type = 'add'}) async {
    if (type == 'init') {
      bangumiList.clear();
    }
    isLoadingMore = true;
    int randomNumber = Random().nextInt(8000) + 1;
    var tag = currentTag;
    var result = await BangumiHTTP.getBangumiList(
      rank: randomNumber,
      tag: tag,
    );
    bangumiList.addAll(result);
    isLoadingMore = false;
    isTimeOut = bangumiList.isEmpty;
  }
}
```

**设计思路分析**：

1. **分离两种数据源**：`trendList` 和 `bangumiList`
2. **支持增量加载**：`type == 'add'` 时不清空列表
3. **loading 状态**：UI 显示加载动画
4. **空状态处理**：`isTimeOut` 用于显示错误/空提示

### 案例 2：CollectController（收藏页）

📁 源码：`lib/pages/collect/collect_controller.dart`

```dart
abstract class _CollectController with Store {
  // 依赖注入
  final _collectCrudRepository = Modular.get<ICollectCrudRepository>();
  final _collectRepository = Modular.get<ICollectRepository>();

  // 存储引用
  Box setting = GStorage.setting;

  // ===== Observable =====
  @observable
  ObservableList<CollectedBangumi> collectibles =
      ObservableList<CollectedBangumi>();

  // ===== 方法 =====
  void loadCollectibles() {
    collectibles.clear();
    collectibles.addAll(_collectCrudRepository.getAllCollectibles());
  }

  @action
  Future<void> addCollect(BangumiItem bangumiItem, {type = 1}) async {
    // 1. 同步 Bangumi
    final bool syncSucceeded = await _syncBangumiCollectIfEnabled(...);
    if (!syncSucceeded) return;

    // 2. 本地存储
    await _collectCrudRepository.addCollectible(bangumiItem, type);
    await GStorage.appendCollectChange(...);
    loadCollectibles();  // 刷新列表
  }

  @action
  Future<void> deleteCollect(BangumiItem bangumiItem) async {
    // 处理删除逻辑
    final action = await _resolveBangumiDeleteSyncAction(bangumiItem);
    // ...
  }
}
```

**设计思路分析**：

1. **Repository 模式**：数据操作委托给 Repository
2. **同步机制**：操作本地数据，同时同步远端
3. **变化日志**：`appendCollectChange` 记录所有操作用于同步
4. **用户确认**：删除时弹出对话框让用户选择

---

## 7. 代码生成

### 7.1 生成命令

```bash
# 生成所有 .g.dart 文件
dart run build_runner build

# 如果有冲突，删除后再生成
dart run build_runner build --delete-conflicting-outputs

# 监听文件变化自动生成
dart run build_runner watch
```

### 7.2 生成的文件

运行 `build_runner` 后会生成 `.g.dart` 文件：

```dart
// lib/pages/popular/popular_controller.g.dart

// GENERATED CODE - DO NOT MODIFY BY HAND

part of 'popular_controller.dart';

mixin _$PopularController on _PopularController, Store {
  // 为每个 @observable 生成原子（Atom）
  late final _$currentTagAtom = ...;
  late final _$bangumiListAtom = ...;
  late final _$isLoadingMoreAtom = ...;

  @override
  String get currentTag { ... }
  @override
  set currentTag(String value) { ... }

  // 为每个 @action 生成包装器
  late final _$queryBangumiByTrendAsyncAction = ...;

  @override
  Future<void> queryBangumiByTrend({String type = 'add'}) {
    return _$queryBangumiByTrendAsyncAction.run(() =>
      super.queryBangumiByTrend(type: type)
    );
  }
}
```

### 7.3 不要手动修改

⚠️ **重要**：`.g.dart` 文件是自动生成的，手动修改会被 `build_runner` 覆盖！

如果生成失败，检查：
1. 注解是否正确（`@observable` 而不是 `@observable`）
2. 类名是否符合模式（`XxxController = _XxxController with _$XxxController`）
3. `part` 声明是否正确

---

## 8. 常见问题

### Q1: 为什么状态没更新？

```dart
// ❌ 错误：直接赋值，不触发响应
void wrong() {
  controller.items = newList;  // 应该用 clear/addAll
}

// ✅ 正确
void right() {
  controller.items.clear();
  controller.items.addAll(newList);
}
```

### Q2: 为什么 UI 没变化？

```dart
// ❌ 错误：Observable 放在 build 方法里创建
Widget build(BuildContext context) {
  final controller = PopularController();  // 每次 build 都新建
  return Observer(
    builder: (_) => Text('${controller.count}'),
  );
}

// ✅ 正确：从 Modular 获取
class MyPage extends StatelessWidget {
  final controller = Modular.get<PopularController>();  // 单例

  @override
  Widget build(BuildContext context) {
    return Observer(...);
  }
}
```

### Q3: 如何监听多个状态？

```dart
// ✅ 方式1：在一个 Observer 里用多个状态
Observer(
  builder: (_) {
    // items 或 isLoading 任意变化都会重建
    if (controller.isLoading) return LoadingWidget();
    return ListView(...);
  },
)

// ✅ 方式2：用 Computed 合并状态
@computed
bool get showLoading => isLoading && items.isEmpty;

@computed
bool get showList => !isLoading && items.isNotEmpty;

@computed
bool get showEmpty => !isLoading && items.isEmpty;
```

### Q4: 异步 action 的错误处理？

```dart
@action
Future<void> loadData() async {
  isLoading = true;
  error = null;  // 先清除旧错误

  try {
    data = await _service.fetch();
  } catch (e) {
    error = e;  // 保存错误供 UI 显示
    KazumiLogger().e('Load failed', error: e);
  }

  isLoading = false;
}

// UI 层
Observer(
  builder: (_) {
    if (controller.error != null) {
      return ErrorWidget(message: controller.error.toString());
    }
    // ...
  },
)
```

---

## 小练习

### 练习 1：添加新状态

📁 打开 `lib/pages/popular/popular_controller.dart`，添加：

1. 添加 `@observable bool hasError = false;`
2. 添加 `@action Future<void> refresh() async {...}`
3. 在 `queryBangumiByTag` 中捕获错误并设置 `hasError = true`

<details>
<summary>答案</summary>

```dart
@observable
bool hasError = false;

@action
Future<void> refresh() async {
  clearBangumiList();
  await queryBangumiByTag(type: 'init');
}

@action
Future<void> queryBangumiByTag({String type = 'add'}) async {
  hasError = false;
  try {
    // ... 原有的网络请求
  } catch (e) {
    hasError = true;
    KazumiLogger().e('Query failed', error: e);
  }
}
```

</details>

### 练习 2：添加 Computed

为 `PopularController` 添加一个计算属性 `totalItems`，返回两个列表的总和。

<details>
<summary>答案</summary>

```dart
@computed
int get totalItems => bangumiList.length + trendList.length;
```

</details>

---

## 下一课预告

下一课我们将学习 [Hive 数据存储](./04-HIVE-STORAGE.md)，了解数据是如何持久化保存的。