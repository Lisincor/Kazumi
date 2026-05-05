# 第一课：Dart 基础语法速通

> 📖 学习目标：学完本课后，你能看懂本项目中 90% 的 Dart 语法
> ⏱️ 预计时间：2-3 小时
> 📁 配套源码：`lib/pages/popular/popular_controller.dart`

---

## 目录

1. [变量与类型](#1-变量与类型)
2. [函数](#2-函数)
3. [类与构造函数](#3-类与构造函数)
4. [异步编程](#4-异步编程)
5. [集合操作](#5-集合操作)
6. [泛型](#6-泛型)
7. [Mixin 与代码生成](#7-mixin-与代码生成)
8. [本项目特有语法](#8-本项目特有语法)

---

## 1. 变量与类型

### 1.1 基本类型

```dart
// 不可变变量（推荐优先使用）
final String name = 'Kazumi';
final int count = 42;
final double score = 9.5;
final bool isActive = true;

// 可变变量
var title = 'Hello';      // 类型由初始值推断
String name = 'Hello';   // 显式声明类型
int? nullable = null;    // 可空类型（? 表示可空）
```

📁 源码参考：`lib/modules/collect/collect_type.dart:4-23`

```dart
enum CollectType {
  none(0, '未收藏'),        // 构造函数参数
  watching(1, '在看'),     // 构造函数参数
  planToWatch(2, '想看'),  // 构造函数参数
  onHold(3, '搁置'),       // 构造函数参数
  watched(4, '看过'),      // 构造函数参数
  abandoned(5, '抛弃');    // 构造函数参数

  const CollectType(this.value, this.label);  // 构造函数

  final int value;    // 属性
  final String label; // 属性
}
```

### 1.2 类型推断

```dart
// Dart 会自动推断类型
var list = [];           // List<dynamic>
var list = <int>[];      // List<int>
var map = <String, int>{}; // Map<String, int>
```

### 1.3 空安全（Null Safety）

```dart
// 不可空类型
String name = 'Kazumi';  // 永远不为 null

// 可空类型（必须在使用前检查）
String? name;
if (name != null) {
  print(name.length);  // 安全使用
}

// 空值合并
String displayName = name ?? '匿名用户';

// 条件属性访问
int? length = name?.length;
```

---

## 2. 函数

### 2.1 基本语法

```dart
// 标准函数
int add(int a, int b) {
  return a + b;
}

// 箭头函数（单行函数）
int add(int a, int b) => a + b;

// 可选参数
void greet(String name, {String? prefix}) {
  print('${prefix ?? ''}$name');
}
greet('小明', prefix: '你好，');

// 默认参数值
void setConfig({bool enabled = true, int timeout = 30}) {
  // ...
}
```

### 2.2 函数作为参数

```dart
// 回调函数类型
typedef ResultCallback<T> = void Function(T result);

// 使用
void fetchData(ResultCallback<String> onSuccess) {
  onSuccess('数据加载成功');
}

// 或者使用 Function 类型
void onComplete(void Function() callback) {
  callback();
}
```

📁 源码参考：`lib/utils/storage.dart:38-56`

```dart
// Future<T> Function() 是函数类型
// 表示一个返回 Future<T> 的函数
static Future<T> _runCollectChangesWriteExclusive<T>(
  Future<T> Function() action,
) {
  final completer = Completer<T>();
  // ...
  return completer.future;
}
```

---

## 3. 类与构造函数

### 3.1 标准构造函数

```dart
class BangumiItem {
  int id;
  String name;
  String summary;

  // 标准构造函数
  BangumiItem({
    required this.id,
    required this.name,
    required this.summary,
  });

  // 命名构造函数
  factory BangumiItem.fromJson(Map<String, dynamic> json) {
    return BangumiItem(
      id: json['id'],
      name: json['name'],
      summary: json['summary'],
    );
  }
}
```

📁 源码参考：`lib/modules/bangumi/bangumi_item.dart:40-56`

```dart
BangumiItem({
  required this.id,
  required this.type,
  required this.name,
  required this.nameCn,
  required this.summary,
  required this.airDate,
  required this.airWeekday,
  required this.rank,
  required this.images,
  required this.tags,
  required this.alias,
  required this.ratingScore,
  required this.votes,
  required this.votesCount,
  required this.info,
});
```

### 3.2 Getter 与 Setter

```dart
class DownloadEpisode {
  // ...

  // Getter
  bool get isCompleted => status == 3;

  // Computed property
  String get statusText {
    switch (status) {
      case 0: return '等待中';
      case 1: return '解析中';
      case 2: return '下载中';
      case 3: return '已完成';
      case 4: return '失败';
      case 5: return '暂停';
      default: return '未知';
    }
  }
}
```

📁 源码参考：`lib/modules/download/download_module.dart:110-117`

```dart
class DownloadStatus {
  static const int pending = 0;
  static const int resolving = 1;
  static const int downloading = 2;
  static const int completed = 3;
  static const int failed = 4;
  static const int paused = 5;
}
```

### 3.3 静态成员

```dart
class Api {
  static const String version = '2.1.0';    // 静态常量
  static const int apiLevel = 6;           // 静态常量

  static String formatUrl(String url, List<dynamic> params) {
    // 静态方法
  }
}

// 调用
Api.formatUrl(url, params);
```

📁 源码参考：`lib/request/api.dart:1-79`

---

## 4. 异步编程

> ⚠️ 这是 Dart 中最重要的概念之一，必须完全理解！

### 4.1 Future

```dart
// 返回 Future 的函数
Future<String> fetchData() async {
  await Future.delayed(Duration(seconds: 1));
  return '数据';
}

// 调用异步函数
void main() async {
  print('开始请求');

  String result = await fetchData();  // 等待结果

  print('收到数据: $result');
}
```

### 4.2 async/await 模式

```dart
// 同步 vs 异步
void syncFunction() {
  var result = heavyCalculation();  // 阻塞
  print(result);
}

Future<void> asyncFunction() async {
  var result = await fetchData();  // 不阻塞
  print(result);
}
```

### 4.3 then 链式调用

```dart
// 两种写法等价
void way1() async {
  var data = await fetchData();
  var parsed = await parseJson(data);
  var result = await saveData(parsed);
}

void way2() {
  fetchData()
    .then((data) => parseJson(data))
    .then((parsed) => saveData(parsed));
}
```

### 4.4 错误处理

```dart
Future<void> safeFetch() async {
  try {
    var result = await fetchData();
    print(result);
  } catch (e, stackTrace) {
    print('出错了: $e');
    print('堆栈: $stackTrace');
  } finally {
    print('无论成功失败都会执行');
  }
}
```

📁 源码参考：`lib/pages/my/my_controller.dart:67-85`

```dart
Future<bool> checkUpdate({String type = 'manual'}) async {
  try {
    final autoUpdater = AutoUpdater();

    if (type == 'manual') {
      await autoUpdater.manualCheckForUpdates();
    } else {
      await autoUpdater.autoCheckForUpdates();
    }

    return true;
  } catch (err) {
    KazumiLogger().e('Update: check update failed', error: err);
    if (type == 'manual') {
      KazumiDialog.showToast(message: '检查更新失败，请稍后重试');
    }
    return false;
  }
}
```

### 4.5 Future.wait 并行等待

```dart
// 并行执行多个异步任务
Future<void> fetchAll() async {
  var results = await Future.wait([
    fetchUser(),
    fetchPosts(),
    fetchComments(),
  ]);

  print('用户: ${results[0]}');
  print('帖子: ${results[1]}');
  print('评论: ${results[2]}');
}
```

---

## 5. 集合操作

### 5.1 List（列表）

```dart
// 创建
var list = <int>[1, 2, 3, 4, 5];
var emptyList = [];

// 常用操作
list.add(6);           // 添加元素
list.remove(3);        // 删除元素
list.length;            // 长度
list.isEmpty;           // 是否为空
list.first;             // 第一个元素
list.last;              // 最后一个元素

// 遍历
for (var item in list) {
  print(item);
}

list.forEach((item) => print(item));

// 转换
var doubled = list.map((e) => e * 2).toList();  // [2, 4, 6, 8, 10, 12]

// 过滤
var evens = list.where((e) => e % 2 == 0).toList();  // [2, 4, 6]

// 查找
var found = list.firstWhere((e) => e > 3);  // 4
var notFound = list.firstWhere((e) => e > 100, orElse: () => -1);

// 转换为其他类型
var set = list.toSet();
var map = list.asMap();  // {0: 1, 1: 2, 2: 3, ...}
```

📁 源码参考：`lib/repositories/collect_repository.dart:69-81`

```dart
Set<int> getBangumiIdsByType(CollectType type) {
  try {
    return _collectiblesBox.values
        .where((item) => item.type == type.value)       // 过滤
        .map<int>((item) => item.bangumiItem.id)         // 转换
        .toSet();                                         // 转 Set
  } catch (e) {
    return <int>{};
  }
}
```

### 5.2 Map（字典）

```dart
// 创建
var map = <String, int>{'a': 1, 'b': 2};
var emptyMap = <String, dynamic>{};

// 常用操作
map['c'] = 3;           // 添加/修改
var value = map['a'];    // 获取（可空）
map.remove('b');         // 删除
map.length;             // 长度
map.containsKey('a');    // 是否包含键

// 遍历
for (var entry in map.entries) {
  print('${entry.key}: ${entry.value}');
}

map.forEach((key, value) => print('$key: $value'));

// 转换
var keys = map.keys.toList();
var values = map.values.toList();
```

📁 源码参考：`lib/modules/bangumi/bangumi_item.dart:114-123`

```dart
images: Map<String, String>.from(
  json['images'] ??
    {
      "large": json['image'],
      "common": "",
      "medium": "",
      "small": "",
      "grid": ""
    },
),
```

### 5.3 Set（集合）

```dart
// 创建
var set = <int>{1, 2, 3, 2, 1};  // {1, 2, 3} 自动去重

// 常用操作
set.add(4);           // 添加
set.remove(2);         // 删除
set.contains(3);      // 是否包含
set.isEmpty;           // 是否为空

// 集合运算
var union = set1.union(set2);           // 并集
var intersection = set1.intersection(set2); // 交集
var difference = set1.difference(set2);    // 差集
```

📁 源码参考：`lib/repositories/collect_repository.dart:86-98`

```dart
Set<int> getBangumiIdsByTypes(List<CollectType> types) {
  try {
    final typeValues = types.map((t) => t.value).toSet();  // 转 Set
    return _collectiblesBox.values
        .where((item) => typeValues.contains(item.type))
        .map<int>((item) => item.bangumiItem.id)
        .toSet();
  } catch (e) {
    return <int>{};
  }
}
```

---

## 6. 泛型

### 6.1 基本用法

```dart
// 泛型类
class Box<T> {
  T value;
  Box(this.value);
}

// 使用
var intBox = Box<int>(42);
var strBox = Box<String>('hello');

// 泛型函数
T first<T>(List<T> list) {
  return list.first;
}
```

### 6.2 泛型约束

```dart
// T 必须继承自某个类
class ComparableBox<T extends Comparable> {
  T value;
}

// T 必须是某几个类型之一
void process<T extends num>(T value) {
  // value 可以是 int 或 double
}
```

📁 源码参考：`lib/utils/storage.dart:38-56`

```dart
// Future<T> 泛型
// T 是返回值的类型
static Future<T> _runCollectChangesWriteExclusive<T>(
  Future<T> Function() action,
) {
  // action 函数返回 Future<T>
  // _runCollectChangesWriteExclusive 本身也返回 Future<T>
}
```

### 6.3 泛型在集合中的使用

```dart
// List<dynamic> vs List<int>
List<dynamic> mixed = [1, 'a', true];      // 可以混合类型
List<int> numbers = [1, 2, 3];             // 只能是 int

// 类型转换
List<int> numbers = list.cast<int>();

// 泛型方法
var filtered = list.whereType<String>().toList();
```

---

## 7. Mixin 与代码生成

### 7.1 Mixin 概念

```dart
// Mixin - 用于复用代码的"混入"类
mixin Logger {
  void log(String message) => print('[LOG] $message');
}

// 使用 with 混入
class MyController with Logger {
  void doSomething() {
    log('做点什么');
  }
}
```

### 7.2 MobX 的 Mixin 模式

```dart
// 这是 MobX 规定的写法
part 'popular_controller.g.dart';

// 生成代码后，_$PopularController 是生成的 mixin
class PopularController = _PopularController with _$PopularController;
```

📁 源码参考：`lib/pages/popular/popular_controller.dart:7-9`

```dart
part 'popular_controller.g.dart';

class PopularController = _PopularController with _$PopularController;

abstract class _PopularController with Store {
  // 状态和方法定义在这里
}
```

### 7.3 代码生成流程

```
源代码 (.dart)
    ↓ 编辑器保存或运行 build_runner
    ↓
生成代码 (.g.dart)
    ↓
最终可执行代码
```

📁 生成的代码示例：`lib/modules/bangumi/bangumi_item.g.dart`

```dart
// 这是由代码生成器自动创建的文件
// 不要手动修改！

part of 'bangumi_item.dart';

class BangumiItemAdapter extends TypeAdapter<BangumiItem> {
  // ...
}
```

---

## 8. 本项目特有语法

### 8.1 Observable 注解

```dart
import 'package:mobx/mobx.dart';

part 'xxx.g.dart';

abstract class _XxxController with Store {
  @observable
  int count = 0;

  @action
  void increment() {
    count++;
  }
}
```

📁 源码参考：`lib/pages/popular/popular_controller.dart:11-31`

```dart
abstract class _PopularController with Store {
  @observable
  String currentTag = '';

  @observable
  ObservableList<BangumiItem> bangumiList = ObservableList.of([]);

  double scrollOffset = 0.0;

  @observable
  bool isLoadingMore = false;
}
```

### 8.2 Hive TypeAdapter 注解

```dart
import 'package:hive_ce/hive.dart';

part 'xxx.g.dart';

@HiveType(typeId: 0)  // 必须是唯一数字
class BangumiItem {
  @HiveField(0)       // 对应字段索引
  int id;

  @HiveField(1)
  String name;
}
```

📁 源码参考：`lib/modules/bangumi/bangumi_item.dart:7-15`

```dart
@HiveType(typeId: 0)
class BangumiItem {
  @HiveField(0)
  int id;

  @HiveField(1)
  int type;

  @HiveField(2)
  String name;
  // ...
}
```

### 8.3 字符串模板

```dart
var name = 'Kazumi';
var message = 'Hello, $name';           // Hello, Kazumi
var message = 'Count: ${1 + 1}';         // Count: 2

// 字符串拼接
var path = '$baseUrl$path';              // 连接字符串
```

📁 源码参考：`lib/plugins/plugins.dart:347-352`

```dart
String buildFullUrl(String urlItem) {
  if (urlItem.contains(baseUrl) ||
      urlItem.contains(baseUrl.replaceAll('https', 'http'))) {
    return urlItem;
  }
  return baseUrl + urlItem;  // 字符串拼接
}
```

### 8.4 Cascade Notation（级联操作）

```dart
// 不用级联
var list = [];
list.add(1);
list.add(2);
list.add(3);

// 用级联
var list = []
  ..add(1)
  ..add(2)
  ..add(3);
```

📁 源码参考：`lib/plugins/plugins_controller.dart:56-63`

```dart
pluginList.clear();
// 不用级联
KazumiLogger().i('Plugins Directory: ${newPluginDirectory!.path}');
// 不用级联
pluginList.addAll(getPluginListFromJson(jsonString));
```

---

## 小练习

### 练习 1：理解异步代码

📁 阅读 `lib/pages/popular/popular_controller.dart:39-48`

```dart
Future<void> queryBangumiByTrend({String type = 'add'}) async {
  if (type == 'init') {
    trendList.clear();
  }
  isLoadingMore = true;       // 状态1
  var result = await BangumiHTTP.getBangumiTrendsList(offset: trendList.length);
  trendList.addAll(result);    // 状态2
  isLoadingMore = false;      // 状态3
  isTimeOut = trendList.isEmpty; // 状态4
}
```

问题：
1. 这个函数返回什么类型？
2. 为什么要用 `await`？
3. `isLoadingMore = true` 和 `isLoadingMore = false` 的作用是什么？

<details>
<summary>答案</summary>

1. 返回 `Future<void>`，表示一个异步操作，不返回具体值
2. `await` 让代码等待网络请求完成后再继续执行，避免在数据还没返回时就添加到列表
3. 用于在 UI 层显示"加载中"状态，告诉用户"正在请求数据，请稍候"

</details>

---

### 练习 2：阅读数据模型

📁 阅读 `lib/modules/download/download_module.dart:37-89`

```dart
@HiveType(typeId: 8)
class DownloadEpisode {
  @HiveField(0)
  int episodeNumber;

  @HiveField(1)
  String episodeName;

  @HiveField(2)
  int road;

  @HiveField(3)
  int status;

  @HiveField(4)
  double progressPercent;
  // ...
}
```

问题：
1. `DownloadEpisode` 的 `typeId` 是多少？
2. `status` 字段的索引是多少？
3. 为什么要用 `@HiveType` 和 `@HiveField` 注解？

<details>
<summary>答案</summary>

1. `typeId` 是 8（定义在类上方的注解）
2. `status` 的 `@HiveField(3)` 说明索引是 3
3. 这些注解用于代码生成，生成 Hive 的 TypeAdapter，用于将类序列化/反序列化到本地存储

</details>

---

## 下一课预告

下一课我们将学习 [应用启动与初始化](./02-APPLICATION.md)，了解 Flutter 应用是如何启动的，以及各个模块是如何被初始化的。