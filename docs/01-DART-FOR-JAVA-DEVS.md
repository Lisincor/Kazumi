# 第一章：Dart 语法速通（Java 开发者版）

> 目标：用 Java 对照的方式，30 分钟内掌握 Dart 核心语法差异
> 配套源码：`lib/pages/popular/popular_controller.dart`、`lib/modules/bangumi/bangumi_item.dart`

---

## 1. 变量声明

### Java vs Dart

```java
// Java
final String name = "Kazumi";
int count = 42;
String nullable = null;  // 编译器不管
```

```dart
// Dart
final name = 'Kazumi';    // 类型推断，不可变
var count = 42;            // 类型推断，可变
String? nullable = null;   // ? 表示可空，编译器强制检查
const pi = 3.14;           // 编译期常量（Java 没有对应概念）
```

关键差异：
- `final` ≈ Java 的 `final`，运行时确定
- `const` = 编译期常量，Java 没有直接对应
- `var` ≈ Java 10 的 `var`，但 Dart 从一开始就支持
- `?` 后缀 = 可空类型，Dart 的空安全比 Java 的 `@Nullable` 严格得多

### 空安全（Null Safety）

```dart
String? name;              // 可能为 null
String name = 'Kazumi';   // 永远不为 null

// 安全调用（≈ Java Optional.map）
int? length = name?.length;

// 空值合并（≈ Java Optional.orElse）
String display = name ?? '匿名';

// 非空断言（你确定不为 null 时使用，类似 Optional.get）
String sure = name!;  // 如果为 null 会抛异常
```

项目实例 — `lib/modules/bangumi/bangumi_item.dart`:
```dart
BangumiItem({
  required this.id,       // required = 必传参数
  required this.name,
  required this.summary,
});
```

---

## 2. 函数

### 基本语法

```java
// Java
public int add(int a, int b) {
    return a + b;
}
```

```dart
// Dart
int add(int a, int b) {
  return a + b;
}

// 箭头函数（单表达式）
int add(int a, int b) => a + b;
```

### 命名参数（Dart 独有，非常常用）

```dart
// 命名参数用 {} 包裹
void fetchData({required String url, int timeout = 30}) {
  // ...
}

// 调用时必须写参数名
fetchData(url: 'https://api.example.com', timeout: 60);
```

对照 Spring Boot：
```java
// Java 用 Builder 模式实现类似效果
Request.builder()
    .url("https://api.example.com")
    .timeout(60)
    .build();
```

Dart 的命名参数让你不需要 Builder 模式就能实现同样的可读性。

### 函数作为参数（≈ Java Function/Consumer）

```dart
// Java: Function<String, Integer> fn
// Dart:
void process(int Function(String) fn) {
  var result = fn('hello');
}

// 或者用 typedef 定义类型别名
typedef ResultCallback<T> = void Function(T result);
```

项目实例 — `lib/utils/storage.dart`:
```dart
static Future<T> _runCollectChangesWriteExclusive<T>(
  Future<T> Function() action,  // 接收一个返回 Future<T> 的函数
) { ... }
```

---

## 3. 类

### 构造函数

```java
// Java
public class BangumiItem {
    private int id;
    private String name;

    public BangumiItem(int id, String name) {
        this.id = id;
        this.name = name;
    }
}
```

```dart
// Dart — 语法糖极多
class BangumiItem {
  int id;
  String name;

  // this.id 自动赋值，不需要 this.id = id
  BangumiItem({required this.id, required this.name});

  // 命名构造函数（Java 没有）
  BangumiItem.empty() : id = 0, name = '';

  // 工厂构造函数（≈ 静态工厂方法）
  factory BangumiItem.fromJson(Map<String, dynamic> json) {
    return BangumiItem(id: json['id'], name: json['name']);
  }
}
```

### Getter/Setter

```dart
class DownloadEpisode {
  int status;

  // Getter（≈ Java getIsCompleted()）
  bool get isCompleted => status == 3;

  // Setter
  set statusCode(int value) => status = value;
}
```

### 没有 interface 关键字

```dart
// Dart 中每个类都隐式定义了一个接口
// 用 abstract class 代替 Java interface
abstract class ICollectRepository {
  List<CollectedBangumi> getAllCollectibles();
  Future<void> addCollectible(BangumiItem item, int type);
}

// 实现接口用 implements
class CollectRepository implements ICollectRepository {
  @override
  List<CollectedBangumi> getAllCollectibles() { ... }
}
```

---

## 4. 异步编程（重点）

这是 Dart 和 Java 差异最大的地方。Dart 是单线程 + 事件循环模型，类似 JavaScript。

### Future ≈ CompletableFuture

```java
// Java
CompletableFuture<String> fetchData() {
    return CompletableFuture.supplyAsync(() -> {
        // 网络请求
        return "data";
    });
}
```

```dart
// Dart — 简洁得多
Future<String> fetchData() async {
  // await 暂停执行，等待结果（不阻塞线程）
  var response = await dio.get('https://api.example.com');
  return response.data;
}
```

### async/await 模式

```dart
Future<void> loadData() async {
  try {
    var result = await fetchData();   // 等待网络请求
    items.addAll(result);             // 处理结果
  } catch (e) {
    print('出错: $e');
  }
}
```

项目实例 — `lib/pages/popular/popular_controller.dart`:
```dart
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
```

### 并行执行

```dart
// 等价于 CompletableFuture.allOf()
var results = await Future.wait([
  fetchUser(),
  fetchPosts(),
  fetchComments(),
]);
```

---

## 5. 集合操作

Dart 的集合 API 和 Java Stream 很像，但更简洁：

```dart
var list = [1, 2, 3, 4, 5];

// Java: list.stream().filter(x -> x > 2).collect(Collectors.toList())
var filtered = list.where((x) => x > 2).toList();

// Java: list.stream().map(x -> x * 2).collect(Collectors.toList())
var mapped = list.map((x) => x * 2).toList();

// Java: list.stream().findFirst().orElse(-1)
var first = list.firstWhere((x) => x > 3, orElse: () => -1);
```

项目实例 — `lib/repositories/collect_repository.dart`:
```dart
Set<int> getBangumiIdsByType(CollectType type) {
  return _collectiblesBox.values
      .where((item) => item.type == type.value)   // 过滤
      .map<int>((item) => item.bangumiItem.id)     // 转换
      .toSet();                                     // 收集
}
```

---

## 6. Mixin（Java 没有的概念）

Mixin 解决了 Java 单继承的限制，类似于"可以带实现的接口"：

```dart
// 定义 mixin
mixin Logger {
  void log(String msg) => print('[LOG] $msg');
}

mixin Validator {
  bool validate(String input) => input.isNotEmpty;
}

// 一个类可以 with 多个 mixin
class MyService with Logger, Validator {
  void process(String data) {
    if (validate(data)) {
      log('Processing: $data');
    }
  }
}
```

在本项目中，MobX 大量使用 mixin：
```dart
// Store 是一个 mixin
abstract class _PopularController with Store {
  // ...
}
```

---

## 7. 代码生成（≈ Lombok/MapStruct）

Dart 用注解 + build_runner 生成代码，概念类似 Java 的 Lombok：

```dart
// 源文件声明"我有一个生成的部分"
part 'popular_controller.g.dart';

// 公共类 = 私有实现 + 生成的 mixin
class PopularController = _PopularController with _$PopularController;
```

| Java (Lombok) | Dart (build_runner) |
|---------------|---------------------|
| `@Data` | `@HiveType` + `@HiveField` |
| `@Builder` | 命名参数构造函数（语言内置） |
| `@Getter/@Setter` | 默认公开（无需注解） |
| 编译时生成 | `dart run build_runner build` |

修改了带注解的文件后，必须重新生成：
```bash
dart run build_runner build --delete-conflicting-outputs
```

---

## 8. 枚举

```dart
// Dart 枚举比 Java 更灵活
enum CollectType {
  none(0, '未收藏'),
  watching(1, '在看'),
  planToWatch(2, '想看'),
  onHold(3, '搁置'),
  watched(4, '看过'),
  abandoned(5, '抛弃');

  const CollectType(this.value, this.label);
  final int value;
  final String label;
}

// 使用
var type = CollectType.watching;
print(type.label);  // '在看'
print(type.value);  // 1
```

---

## 9. 字符串模板

```dart
var name = 'Kazumi';
var version = 2;

// 简单变量
var msg = 'Hello, $name';           // Hello, Kazumi

// 表达式
var msg = 'Version ${version + 1}'; // Version 3

// 多行字符串
var json = '''
{
  "name": "$name",
  "version": $version
}
''';
```

---

## 10. 本项目中的常见模式速查

| 你在代码中看到... | 它的意思是... |
|------------------|-------------|
| `part 'xxx.g.dart'` | 声明有一个代码生成的文件 |
| `class A = _A with _$A` | MobX 的标准写法，A 是公共类 |
| `abstract class _A with Store` | MobX Store 的实现类 |
| `@observable` | 这个字段变化时 UI 会自动更新 |
| `@action` | 这个方法会修改响应式状态 |
| `@HiveType(typeId: N)` | Hive 数据库的实体类标记 |
| `@HiveField(N)` | Hive 字段索引（类似数据库列） |
| `Modular.get<T>()` | 从 DI 容器获取实例（≈ @Autowired） |
| `ObservableList<T>` | 响应式列表，变化时通知 UI |
| `Future<void>` | 异步方法，无返回值 |
| `async/await` | 异步等待，不阻塞 |
| `?.` | 安全调用，对象为 null 时不执行 |
| `??` | 空值合并，左边为 null 时用右边 |
| `!` | 非空断言，告诉编译器"我确定不为 null" |
| `=> expr` | 箭头函数，等价于 `{ return expr; }` |
| `required` | 命名参数必传标记 |
| `late` | 延迟初始化（保证使用前会赋值） |

---

## 练习

打开 `lib/pages/popular/popular_controller.dart`，回答：

1. `ObservableList.of([])` 创建了什么？（提示：对照 Java 的 `new ArrayList<>()`）
2. `queryBangumiByTrend` 为什么返回 `Future<void>` 而不是 `void`？
3. `{String type = 'add'}` 是什么意思？

<details>
<summary>答案</summary>

1. 创建了一个空的响应式列表，当列表内容变化时会通知所有监听者（Observer widget）更新 UI
2. 因为方法内部使用了 `await`，任何使用 `await` 的方法必须标记为 `async` 并返回 `Future`
3. 这是一个命名参数，带默认值 `'add'`。调用时可以写 `queryBangumiByTrend()` 或 `queryBangumiByTrend(type: 'init')`

</details>

---

下一章：[Flutter 基础：Widget 体系与生命周期](./02-FLUTTER-BASICS.md)
