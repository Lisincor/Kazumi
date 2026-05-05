# 第四课：Hive 数据存储

> 📖 学习目标：理解 Hive 数据库的使用，创建新的数据模型，掌握 GStorage 的使用
> ⏱️ 预计时间：2-3 小时
> 📁 配套源码：`lib/utils/storage.dart`、`lib/modules/bangumi/bangumi_item.dart`

---

## 目录

1. [什么是 Hive？](#1-什么是-hive)
2. [数据模型定义](#2-数据模型定义)
3. [GStorage 单例](#3-gstorage-单例)
4. [Box 操作详解](#4-box-操作详解)
5. [数据同步机制](#5-数据同步机制)
6. [实际案例分析](#6-实际案例分析)
7. [常见问题](#7-常见问题)

---

## 1. 什么是 Hive？

### 1.1 类比理解

Hive 就像一个带有"标签"的储物柜：

```
传统文件：        Hive 数据库：
┌─────────┐       ┌────────────────────┐
│ 文件A   │       │ Box: users         │
├─────────┤       │ ┌──────┐┌──────┐  │
│ 文件B   │  →    │ │User1 ││User2 │  │
├─────────┤       │ └──────┘└──────┘  │
│ 文件C   │       └────────────────────┘
└─────────┘       │ Box: settings      │
                  │ ┌──────┐┌──────┐  │
                  │ │key1  ││key2  │  │
                  │ └──────┘└──────┘  │
                  └────────────────────┘
```

- **Box**：类似数据库中的"表"
- **Key-Value**：每个数据用"键"来标识
- **TypeAdapter**：自动把 Dart 对象转换为二进制存储

### 1.2 在本项目中的应用

📁 Hive 用于存储：

| 数据 | Box 名称 | 说明 |
|------|---------|------|
| 番剧信息 | favorites/collectibles | 收藏列表 |
| 历史记录 | histories | 观看历史 |
| 设置 | setting | 用户偏好 |
| 搜索历史 | searchHistory | 搜索记录 |
| 下载记录 | downloads | 下载任务 |

---

## 2. 数据模型定义

### 2.1 基本模型

📁 源码：`lib/modules/bangumi/bangumi_item.dart`

```dart
import 'package:hive_ce/hive.dart';

part 'bangumi_item.g.dart';

@HiveType(typeId: 0)  // 必须是唯一数字 (0-255)
class BangumiItem {
  // 每个字段需要对应的索引
  @HiveField(0)
  int id;

  @HiveField(1)
  int type;

  @HiveField(2)
  String name;

  @HiveField(3)
  String nameCn;

  @HiveField(4)
  String summary;

  @HiveField(5)
  String airDate;

  @HiveField(6)
  int airWeekday;

  @HiveField(7)
  int rank;

  @HiveField(8)
  Map<String, String> images;

  @HiveField(9, defaultValue: [])  // 指定默认值
  List<BangumiTag> tags;

  @HiveField(10, defaultValue: [])
  List<String> alias;

  @HiveField(11, defaultValue: 0.0)
  double ratingScore;

  // 构造函数
  BangumiItem({
    required this.id,
    required this.type,
    required this.name,
    // ... 其他参数
  });

  // 工厂构造函数（从 JSON 创建）
  factory BangumiItem.fromJson(Map<String, dynamic> json) {
    return BangumiItem(
      id: json['id'],
      // ... 字段映射
    );
  }
}
```

### 2.2 注解详解

```dart
@HiveType(typeId: 0)
```

- 标识这个类需要生成 TypeAdapter
- `typeId` 必须在 0-255 之间
- 每个模型类的 typeId 必须唯一

```dart
@HiveField(0)
int id;
```

- 标识字段的索引位置
- 从 0 开始，连续递增
- 顺序很重要，添加新字段时必须用新的索引

### 2.3 常用字段类型

```dart
@HiveField(0)  int id;                    // 整数
@HiveField(1)  double score;             // 浮点数
@HiveField(2)  String name;              // 字符串
@HiveField(3)  bool isActive;           // 布尔值
@HiveField(4)  DateTime createdAt;      // 日期时间
@HiveField(5)  List<String> tags;       // 列表
@HiveField(6)  Map<String, int> data;    // 字典
@HiveField(7)  SomeObject obj;          // 嵌套对象（需要自己的 TypeAdapter）
```

### 2.4 枚举类型

📁 源码：`lib/modules/collect/collect_type.dart`

```dart
// 枚举本身不需要 @HiveType
// 但存储时用 int，读取时转换

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

  static CollectType fromValue(int value) {
    return CollectType.values.firstWhere(
      (type) => type.value == value,
      orElse: () => CollectType.none,
    );
  }
}

// 使用
@HiveField(0)
int type;  // 存储为 int

// 读取时转换
CollectType collectType = CollectType.fromValue(type);
```

### 2.5 嵌套对象

📁 源码：`lib/modules/download/download_module.dart`

```dart
@HiveType(typeId: 7)
class DownloadRecord {
  @HiveField(0)
  int bangumiId;

  @HiveField(1)
  String bangumiName;

  // 嵌套对象
  @HiveField(4)
  Map<int, DownloadEpisode> episodes;
}

@HiveType(typeId: 8)
class DownloadEpisode {
  @HiveField(0)
  int episodeNumber;

  @HiveField(1)
  String episodeName;

  // ...
}
```

---

## 3. GStorage 单例

📁 源码：`lib/utils/storage.dart`

### 3.1 单例模式

```dart
class GStorage {
  // 静态属性（类变量，所有实例共享）
  static late Box<BangumiItem> favorites;
  static late Box<CollectedBangumi> collectibles;
  static late Box<History> histories;
  static late final Box<dynamic> setting;
  static late Box<String> shieldList;
  static late Box<SearchHistory> searchHistory;
  static late Box<DownloadRecord> downloads;

  // 初始化方法
  static Future<void> init() async {
    // 1. 注册 TypeAdapter
    Hive.registerAdapter(BangumiItemAdapter());
    Hive.registerAdapter(CollectedBangumiAdapter());
    // ... 其他 adapter

    // 2. 打开 Box
    favorites = await Hive.openBox<BangumiItem>('favorites');
    collectibles = await Hive.openBox<CollectedBangumi>('collectibles');
    // ...
  }

  // 私有构造函数（防止实例化）
  GStorage._();
}
```

### 3.2 初始化流程

📁 源码：`lib/main.dart:36-39`

```dart
try {
  final hivePath = '${(await getApplicationSupportDirectory()).path}/hive';
  await Hive.initFlutter(hivePath);
  await GStorage.init();
} catch (e) {
  // 处理初始化失败
}
```

### 3.3 安全打开 Box

📁 源码：`lib/utils/storage.dart:165-186`

```dart
/// 打开一个 Box，如果损坏则自动恢复
static Future<Box<T>> _openBoxSafe<T>(String boxName) async {
  try {
    return await Hive.openBox<T>(boxName);
  } catch (e) {
    KazumiLogger().e('GStorage: Box "$boxName" corrupted, attempting recovery', error: e);

    // 删除损坏的文件
    await _deleteBoxFiles(boxName);

    // 重新创建（数据会丢失）
    try {
      final box = await Hive.openBox<T>(boxName);
      KazumiLogger().i('GStorage: Box "$boxName" recovered successfully (data lost)');
      return box;
    } catch (e2) {
      KazumiLogger().e('GStorage: Failed to recover box "$boxName"', error: e2);
      rethrow;
    }
  }
}
```

### 3.4 Box 注册

📁 源码：`lib/utils/storage.dart:141-163`

```dart
static Future init() async {
  // 注册所有 TypeAdapter
  Hive.registerAdapter(BangumiItemAdapter());
  Hive.registerAdapter(BangumiTagAdapter());
  Hive.registerAdapter(CollectedBangumiAdapter());
  Hive.registerAdapter(ProgressAdapter());
  Hive.registerAdapter(HistoryAdapter());
  Hive.registerAdapter(CollectedBangumiChangeAdapter());
  Hive.registerAdapter(SearchHistoryAdapter());
  Hive.registerAdapter(DownloadRecordAdapter());
  Hive.registerAdapter(DownloadEpisodeAdapter());

  // 打开所有 Box
  favorites = await _openBoxSafe<BangumiItem>('favorites');
  collectibles = await _openBoxSafe<CollectedBangumi>('collectibles');
  histories = await _openBoxSafe<History>('histories');
  setting = await _openBoxSafe<dynamic>('setting');
  collectChanges = await _openBoxSafe<CollectedBangumiChange>('collectchanges');
  shieldList = await _openBoxSafe<String>('shieldList');
  searchHistory = await _openBoxSafe<SearchHistory>('searchHistory');
  downloads = await _openBoxSafe<DownloadRecord>('downloads');
}
```

---

## 4. Box 操作详解

### 4.1 基本 CRUD

```dart
// ===== Create / Update =====
await box.put('key', value);      // 添加或更新
await box.add(value);             // 添加（自动生成整数 key）
await box.putAt(index, value);    // 在指定位置更新

// ===== Read =====
var value = box.get('key');        // 通过 key 获取
var value = box.getAt(index);      // 通过索引获取
var all = box.values.toList();     // 获取所有值
var keys = box.keys.toList();      // 获取所有键

// ===== Delete =====
await box.delete('key');          // 删除指定 key
await box.deleteAt(index);        // 删除指定索引
await box.clear();                // 清空所有
await box.flush();                // 强制刷新到磁盘

// ===== Check =====
bool exists = box.containsKey('key');
bool isEmpty = box.isEmpty;
int length = box.length;
```

### 4.2 遍历操作

```dart
// 遍历所有键值对
for (var key in box.keys) {
  var value = box.get(key);
  print('$key: $value');
}

// 或者
box.forEach((key, value) {
  print('$key: $value');
});

// 获取符合条件的数据
var filtered = box.values.where((item) => item.type == 1).toList();
```

📁 源码参考：`lib/repositories/collect_repository.dart:68-81`

```dart
@override
Set<int> getBangumiIdsByType(CollectType type) {
  try {
    return _collectiblesBox.values
        .where((item) => item.type == type.value)  // 过滤
        .map<int>((item) => item.bangumiItem.id)   // 转换
        .toSet();                                  // 去重
  } catch (e) {
    return <int>{};
  }
}
```

### 4.3 默认值处理

```dart
// 获取值，如果不存在则返回默认值
var value = box.get('key', defaultValue: 'default');

// 处理 null
var value = box.get('nonexistent');
print(value);  // null

// 安全获取
var value = box.get('key') ?? 'default';
```

### 4.4 监听变化

```dart
// 监听 Box 变化
box.watch().listen((event) {
  print('Key: ${event.key}, Type: ${event.operation}');
});

// 只监听某个 key
box.watchKey('myKey').listen((event) {
  print('myKey changed to ${event.value}');
});
```

---

## 5. 数据同步机制

### 5.1 变化日志

📁 源码：`lib/utils/storage.dart:92-111`

```dart
/// 追加收藏变化记录（用于 WebDAV 同步）
static Future<CollectedBangumiChange> appendCollectChange({
  required int bangumiId,
  required int action,  // 1=add, 2=update, 3=delete
  required int type,
  int? timestamp,
}) {
  return _runCollectChangesWriteExclusive(() async {
    final change = CollectedBangumiChange(
      _generateCollectChangeIdLocked(),
      bangumiId,
      action,
      type,
      timestamp ?? (DateTime.now().millisecondsSinceEpoch ~/ 1000),
    );
    await collectChanges.put(change.id, change);
    await collectChanges.flush();
    return change;
  });
}
```

### 5.2 写队列（避免并发冲突）

📁 源码：`lib/utils/storage.dart:29-57`

```dart
// 写操作队列，确保串行执行
static Future<void> _collectChangesWriteQueue = Future.value();

static Future<T> _runCollectChangesWriteExclusive<T>(
  Future<T> Function() action,
) {
  final completer = Completer<T>();
  final previousWrite = _collectChangesWriteQueue;

  _collectChangesWriteQueue = (() async {
    try {
      await previousWrite;  // 等待前一个写操作完成
    } catch (_) {}

    try {
      completer.complete(await action());
    } catch (e, stackTrace) {
      completer.completeError(e, stackTrace);
    }
  })();

  return completer.future;
}
```

### 5.3 ID 生成器

📁 源码：`lib/utils/storage.dart:59-90`

```dart
static void _initializeNextCollectChangeIdLocked() {
  if (_collectChangeIdInitialized) return;

  var maxExistingId = 0;
  for (final key in collectChanges.keys) {
    if (key is int && key > maxExistingId) {
      maxExistingId = key;
    }
  }

  _nextCollectChangeId = maxExistingId;
  _collectChangeIdInitialized = true;
}

static int _generateCollectChangeIdLocked() {
  _initializeNextCollectChangeIdLocked();

  final currentSeconds = DateTime.now().millisecondsSinceEpoch ~/ 1000;
  // 确保 ID 大于任何现有 ID，或等于当前时间戳
  var nextId = _nextCollectChangeId < currentSeconds
      ? currentSeconds
      : _nextCollectChangeId + 1;

  while (collectChanges.containsKey(nextId)) {
    nextId++;
  }

  _nextCollectChangeId = nextId;
  return nextId;
}
```

---

## 6. 实际案例分析

### 案例 1：收藏系统

📁 源码：`lib/modules/collect/collect_module.dart`

```dart
@HiveType(typeId: 3)
class CollectedBangumi {
  @HiveField(0)
  BangumiItem bangumiItem;

  @HiveField(1)
  DateTime time;

  // 1=在看, 2=想看, 3=搁置, 4=看过, 5=抛弃
  @HiveField(2)
  int type;

  // 快捷访问
  String get key => bangumiItem.id.toString();

  CollectedBangumi(this.bangumiItem, this.time, this.type);
}
```

使用示例：

```dart
// 添加收藏
var bangumi = BangumiItem(...);
var collected = CollectedBangumi(bangumi, DateTime.now(), 1);
await GStorage.collectibles.put(bangumi.id.toString(), collected);

// 获取收藏
var collected = GStorage.collectibles.get(bangumiId.toString());
if (collected != null) {
  print('Type: ${CollectType.fromValue(collected.type).label}');
}

// 删除收藏
await GStorage.collectibles.delete(bangumiId.toString());
```

### 案例 2：下载记录

📁 源码：`lib/modules/download/download_module.dart`

```dart
@HiveType(typeId: 7)
class DownloadRecord {
  @HiveField(0)
  int bangumiId;

  @HiveField(1)
  String bangumiName;

  @HiveField(2)
  String bangumiCover;

  @HiveField(3)
  String pluginName;

  // 一个番剧可能有多个剧集
  @HiveField(4)
  Map<int, DownloadEpisode> episodes;

  @HiveField(5)
  DateTime createdAt;

  String get key => '${pluginName}_$bangumiId';
}

@HiveType(typeId: 8)
class DownloadEpisode {
  @HiveField(0)
  int episodeNumber;

  @HiveField(1)
  String episodeName;

  @HiveField(2)
  int road;

  // 0=pending, 1=resolving, 2=downloading, 3=completed, 4=failed, 5=paused
  @HiveField(3)
  int status;

  @HiveField(4)
  double progressPercent;

  @HiveField(5)
  int totalSegments;

  @HiveField(6)
  int downloadedSegments;

  // ...
}
```

使用示例：

```dart
// 创建下载记录
var record = DownloadRecord(
  bangumiId,
  bangumiName,
  cover,
  pluginName,
  {1: DownloadEpisode(...), 2: DownloadEpisode(...)},
  DateTime.now(),
);

// 保存
await GStorage.downloads.put(record.key, record);

// 更新进度
var record = GStorage.downloads.get(key);
record.episodes[1]!.progressPercent = 0.5;
await GStorage.downloads.put(key, record);
```

### 案例 3：设置存储

📁 源码：`lib/utils/storage.dart:388-475`

```dart
// 预设的设置 key
class SettingBoxKey {
  static const String hardwareDecoder = 'hardwareDecoder';
  static const String autoUpdate = 'autoUpdate';
  static const String themeMode = 'themeMode';
  static const String themeColor = 'themeColor';
  // ... 更多 key
}

// 存储设置
await GStorage.setting.put(SettingBoxKey.themeMode, 'dark');

// 读取设置
var themeMode = GStorage.setting.get(
  SettingBoxKey.themeMode,
  defaultValue: 'system',  // 默认值
);
```

---

## 7. 常见问题

### Q1: 怎么添加新字段？

1. 在模型类中添加新字段，给它一个新的 `@HiveField(n)`
2. 在 TypeAdapter 中处理（如果不处理会报错）
3. 运行 `dart run build_runner build`

```dart
@HiveType(typeId: 0)
class MyModel {
  @HiveField(0)
  int id;

  @HiveField(1)
  String name;

  @HiveField(2)  // 新字段
  String? description;

  @HiveField(3)  // 新字段
  int count = 0;  // 带默认值
}
```

### Q2: 修改字段类型后数据丢失？

是的！Hive 不支持字段类型自动转换。修改类型需要：
1. 备份数据
2. 删除 Box
3. 修改代码
4. 重新运行

### Q3: 如何监听数据变化？

```dart
// 监听 Box 整体变化
listener = box.listen((event) {
  print('Changed: ${event.key}');
});

// 在 dispose 时移除监听
@override
void dispose() {
  listener.cancel();
  super.dispose();
}
```

### Q4: Box 数据在哪里？

```
Android: /data/data/com.example.app/databases/
iOS:      /var/mobile/Containers/Data/Application/xxx/Library/Application Support/hive/
Windows:  %APPDATA%/kazumi/hive/
macOS:    ~/Library/Application Support/kazumi/hive/
Linux:    ~/.local/share/kazumi/hive/
```

---

## 小练习

### 练习 1：创建新模型

假设你要添加一个"观看进度"模型：

```dart
// 1. 添加 @HiveType(typeId: 9)
// 2. 添加字段：
//    - bangumiId (int)
//    - episodeId (int)
//    - progress (double, 0.0-1.0)
//    - lastWatchTime (DateTime)
// 3. 运行 build_runner
```

<details>
<summary>答案</summary>

```dart
part 'progress_module.g.dart';

@HiveType(typeId: 9)
class WatchProgress {
  @HiveField(0)
  int bangumiId;

  @HiveField(1)
  int episodeId;

  @HiveField(2, defaultValue: 0.0)
  double progress;

  @HiveField(3)
  DateTime lastWatchTime;

  WatchProgress(this.bangumiId, this.episodeId, this.progress, this.lastWatchTime);
}
```

</details>

### 练习 2：使用 Box 操作

完成以下操作：

1. 获取所有收藏类型为"在看"的番剧 ID
2. 删除 ID 为 123 的收藏
3. 更新 ID 为 456 的收藏类型为"看过"

<details>
<summary>答案</summary>

```dart
// 1. 获取在看番剧 ID
final watchingIds = GStorage.collectibles.values
    .where((item) => item.type == 1)
    .map((item) => item.bangumiItem.id)
    .toSet();

// 2. 删除收藏
await GStorage.collectibles.delete('123');

// 3. 更新收藏类型
var collected = GStorage.collectibles.get('456');
if (collected != null) {
  collected.type = 4;  // 看过
  await GStorage.collectibles.put('456', collected);
}
```

</details>

### 练习 3：数据迁移

如果要在原有模型中添加新字段，并保持向后兼容，应该怎么做？

<details>
<summary>提示</summary>

1. 在模型中给新字段设置 `defaultValue`
2. 在 `.g.dart` 的 read 方法中处理 null 值
3. 使用 `??` 操作符提供默认值

</details>

---

## 下一课预告

下一课我们将学习 [网络请求](./05-DIO-NETWORK.md)，了解如何与后端 API 通信。