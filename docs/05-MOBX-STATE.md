# 第五章：MobX 响应式状态管理

> 目标：理解 MobX 的响应式机制，学会编写 Controller
> 配套源码：`lib/pages/popular/popular_controller.dart`、`lib/pages/collect/collect_controller.dart`

---

## 1. 为什么需要状态管理

### 问题场景

假设你有一个番剧列表页面：
- 用户下拉刷新 → 显示加载动画 → 请求数据 → 更新列表 → 隐藏加载动画
- 用户收藏了一部番剧 → 收藏页需要同步更新

如果用原生 `setState`，每个状态变化都要手动通知 UI 更新，代码会变得混乱。

### MobX 的解决方案

MobX 是一个响应式状态管理库：
- 你标记哪些数据是"可观察的"（`@observable`）
- 你标记哪些方法会修改数据（`@action`）
- UI 自动监听数据变化并更新（`Observer` widget）

类比：
- `@observable` ≈ Vue 的 `ref()` / React 的 `useState`
- `@action` ≈ Vuex 的 mutation / Redux 的 dispatch
- `Observer` ≈ Vue 模板中的响应式绑定

---

## 2. 核心概念

### Observable — 可观察状态

```dart
@observable
String currentTag = '';

@observable
bool isLoadingMore = false;

@observable
ObservableList<BangumiItem> bangumiList = ObservableList.of([]);
```

当这些值变化时，所有监听它们的 `Observer` widget 会自动重建。

### Action — 修改状态的方法

```dart
Future<void> queryBangumiByTrend({String type = 'add'}) async {
  isLoadingMore = true;                    // 修改 observable
  var result = await BangumiHTTP.getBangumiTrendsList(offset: trendList.length);
  trendList.addAll(result);                // 修改 observable
  isLoadingMore = false;                   // 修改 observable
}
```

### Observer — UI 自动更新

```dart
Observer(
  builder: (_) {
    // 这个函数内引用的所有 @observable 变量
    // 任何一个变化，这个函数就会重新执行
    if (controller.isLoadingMore) {
      return CircularProgressIndicator();
    }
    return ListView.builder(
      itemCount: controller.bangumiList.length,
      itemBuilder: (_, i) => BangumiCard(item: controller.bangumiList[i]),
    );
  },
)
```

---

## 3. Controller 编写模板

### 完整模板

```dart
import 'package:mobx/mobx.dart';

// 1. 声明生成文件
part 'xxx_controller.g.dart';

// 2. 公共类 = 私有类 + 生成的 mixin
class XxxController = _XxxController with _$XxxController;

// 3. 私有抽象类，混入 Store
abstract class _XxxController with Store {
  // ===== 依赖 =====
  final _repository = Modular.get<IXxxRepository>();

  // ===== 状态 =====
  @observable
  bool isLoading = false;

  @observable
  ObservableList<Item> items = ObservableList.of([]);

  // ===== 非响应式字段（不需要触发 UI 更新的）=====
  double scrollOffset = 0.0;

  // ===== 方法 =====
  Future<void> loadData() async {
    isLoading = true;
    try {
      var result = await _repository.fetchAll();
      items.clear();
      items.addAll(result);
    } catch (e) {
      KazumiLogger().e('Load failed', error: e);
    }
    isLoading = false;
  }
}
```

### 为什么是这个结构？

```dart
class PopularController = _PopularController with _$PopularController;
```

这行代码的含义：
- `_PopularController` — 你写的实现（私有）
- `_$PopularController` — build_runner 生成的 mixin（包含响应式逻辑）
- `PopularController` — 最终的公共类（两者合并）

类比 Lombok：
```java
// Java + Lombok
@Data  // 编译时生成 getter/setter/toString
public class User { ... }

// Dart + MobX
// build_runner 生成响应式 getter/setter
class PopularController = _PopularController with _$PopularController;
```

---

## 4. 项目实例分析

### PopularController（推荐页）

`lib/pages/popular/popular_controller.dart`:

```dart
abstract class _PopularController with Store {
  final ScrollController scrollController = ScrollController();

  @observable
  String currentTag = '';

  @observable
  ObservableList<BangumiItem> bangumiList = ObservableList.of([]);

  @observable
  ObservableList<BangumiItem> trendList = ObservableList.of([]);

  double scrollOffset = 0.0;  // 不需要响应式

  @observable
  bool isLoadingMore = false;

  @observable
  bool isTimeOut = false;

  void setCurrentTag(String s) {
    currentTag = s;
  }

  Future<void> queryBangumiByTrend({String type = 'add'}) async {
    if (type == 'init') {
      trendList.clear();
    }
    isLoadingMore = true;
    var result = await BangumiHTTP.getBangumiTrendsList(offset: trendList.length);
    trendList.addAll(result);
    isLoadingMore = false;
    isTimeOut = trendList.isEmpty;
  }
}
```

设计要点：
- `scrollOffset` 没有 `@observable`，因为滚动位置不需要触发 UI 重建
- `isLoadingMore` 控制加载动画的显示/隐藏
- `isTimeOut` 控制空状态/错误提示的显示
- `type == 'init'` 区分"首次加载"和"加载更多"

### CollectController（收藏页）

```dart
abstract class _CollectController with Store {
  final _collectCrudRepository = Modular.get<ICollectCrudRepository>();
  final _collectRepository = Modular.get<ICollectRepository>();

  @observable
  ObservableList<CollectedBangumi> collectibles = ObservableList();

  void loadCollectibles() {
    collectibles.clear();
    collectibles.addAll(_collectCrudRepository.getAllCollectibles());
  }

  Future<void> addCollect(BangumiItem bangumiItem, {type = 1}) async {
    // 1. 远程同步
    await _syncBangumiCollectIfEnabled(...);
    // 2. 本地存储
    await _collectCrudRepository.addCollectible(bangumiItem, type);
    // 3. 刷新列表（触发 UI 更新）
    loadCollectibles();
  }
}
```

设计要点：
- 通过 Repository 接口访问数据（面向接口编程）
- `loadCollectibles()` 刷新整个列表，Observer 自动更新 UI
- 操作完成后主动刷新，确保 UI 和数据一致

---

## 5. 在 Page 中使用

### 基本模式

```dart
class _PopularPageState extends State<PopularPage> {
  // 从 DI 容器获取 Controller（单例）
  final controller = Modular.get<PopularController>();

  @override
  void initState() {
    super.initState();
    // 页面创建时加载数据
    controller.queryBangumiByTrend(type: 'init');
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Observer(
        builder: (_) {
          if (controller.isLoadingMore && controller.trendList.isEmpty) {
            return Center(child: CircularProgressIndicator());
          }
          return ListView.builder(
            itemCount: controller.trendList.length,
            itemBuilder: (_, index) {
              return BangumiCard(item: controller.trendList[index]);
            },
          );
        },
      ),
    );
  }
}
```

### Observer 的精准更新

```dart
// 只有 isLoadingMore 或 trendList 变化时才重建
Observer(
  builder: (_) {
    if (controller.isLoadingMore) return LoadingWidget();
    return ListWidget(items: controller.trendList);
  },
)

// 可以嵌套多个 Observer 实现局部更新
Column(
  children: [
    Observer(builder: (_) => Text('Tag: ${controller.currentTag}')),
    Observer(builder: (_) => ListView(...)),  // 列表变化不影响上面的 Text
  ],
)
```

---

## 6. 常见模式

### 列表加载 + 分页

```dart
@observable
ObservableList<Item> items = ObservableList.of([]);

@observable
bool isLoading = false;

@observable
bool hasMore = true;

int _page = 0;

Future<void> loadMore() async {
  if (isLoading || !hasMore) return;
  isLoading = true;
  var result = await api.fetch(page: _page);
  items.addAll(result);
  hasMore = result.length >= 20;
  _page++;
  isLoading = false;
}

void refresh() {
  _page = 0;
  items.clear();
  hasMore = true;
  loadMore();
}
```

### 搜索 + 防抖

```dart
@observable
String keyword = '';

@observable
ObservableList<SearchItem> results = ObservableList.of([]);

Timer? _debounce;

void onSearchChanged(String value) {
  keyword = value;
  _debounce?.cancel();
  _debounce = Timer(Duration(milliseconds: 300), () {
    _doSearch(value);
  });
}

Future<void> _doSearch(String query) async {
  if (query.isEmpty) {
    results.clear();
    return;
  }
  isLoading = true;
  results.clear();
  results.addAll(await api.search(query));
  isLoading = false;
}
```

### 设置项持久化

```dart
@observable
bool darkMode = false;

void loadSettings() {
  darkMode = GStorage.setting.get(SettingBoxKey.darkMode, defaultValue: false);
}

void setDarkMode(bool value) {
  darkMode = value;
  GStorage.setting.put(SettingBoxKey.darkMode, value);
}
```

---

## 7. 代码生成

修改了 Controller 后，必须重新生成 `.g.dart` 文件：

```bash
dart run build_runner build --delete-conflicting-outputs
```

生成的文件内容（你不需要手动修改）：
```dart
// popular_controller.g.dart（自动生成）
mixin _$PopularController on _PopularController, Store {
  // 为每个 @observable 生成 getter/setter 拦截器
  // 为每个 @action 生成事务包装器
}
```

如果生成失败，检查：
1. `part 'xxx_controller.g.dart';` 声明是否正确
2. 类名模式是否正确：`class A = _A with _$A;`
3. 是否有语法错误

---

## 8. MobX vs Spring 对照

| MobX 概念 | Spring/Java 对照 | 说明 |
|-----------|-----------------|------|
| `@observable` | 被 `@Observed` 的字段 | 变化时通知监听者 |
| `@action` | `@Transactional` 方法 | 批量修改状态 |
| `@computed` | 缓存的 getter | 基于其他状态计算 |
| `Observer` widget | `@EventListener` | 监听状态变化 |
| `ObservableList` | 带监听的 `ArrayList` | 增删改都会通知 |
| `Store` mixin | — | 标记这是一个状态容器 |
| `.g.dart` 文件 | Lombok 生成的代码 | 自动生成的样板代码 |

---

## 练习

1. 打开 `lib/pages/popular/popular_controller.dart`，识别哪些字段是响应式的，哪些不是
2. 如果你要给推荐页添加一个"收藏数量"显示，需要：
   - 在哪里添加 `@observable`？
   - 在哪里修改这个值？
   - UI 如何自动更新？

<details>
<summary>答案</summary>

1. 响应式：`currentTag`、`bangumiList`、`trendList`、`isLoadingMore`、`isTimeOut`。非响应式：`scrollController`、`scrollOffset`
2. 在 Controller 中添加 `@observable int collectCount = 0;`，在加载数据时更新它，UI 中用 `Observer(builder: (_) => Text('${controller.collectCount}'))` 显示

</details>

---

下一章：[Hive 本地持久化](./06-HIVE-STORAGE.md)
