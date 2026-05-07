# 第六章：Hive 本地持久化

> 目标：理解 Hive 数据库的使用方式，对照 JPA/MyBatis 理解
> 配套源码：`lib/utils/storage.dart`、`lib/modules/bangumi/bangumi_item.dart`、`lib/repositories/collect_repository.dart`

---

## 1. Hive 是什么

Hive 是一个轻量级的 NoSQL 本地数据库，用于 Flutter 应用的数据持久化。

### 对照 Java 技术栈

| Hive 概念 | Java 对照 | 说明 |
|-----------|-----------|------|
| Box | 数据库表 / Redis Hash | 键值对存储容器 |
| TypeAdapter | JPA Entity 映射 | 对象序列化/反序列化 |
| `@HiveType` | `@Entity` | 标记可持久化的类 |
| `@HiveField` | `@Column` | 标记可持久化的字段 |
| `box.put(key, value)` | `repository.save()` | 写入数据 |
| `box.get(key)` | `repository.findById()` | 读取数据 |
| `box.values` | `repository.findAll()` | 获取所有数据 |

### 为什么选 Hive 而不是 SQLite

- 不需要写 SQL，纯 Dart 对象存取
- 性能极快（内存映射文件）
- 适合移动端和桌面端的轻量存储
- 缺点：不支持复杂查询（没有 JOIN、WHERE 子句）

---

## 2. GStorage — 存储封装

`lib/utils/storage.dart` 是整个应用的存储入口，类似 Spring 的 `DataSource` 配置：

```dart
class GStorage {
  // 各种 Box（≈ 各种数据库表）
  static late Box<CollectedBangumi> collectibles;  // 收藏表
  static late Box<History> histories;               // 历史表
  static late Box<DownloadRecord> downloads;        // 下载表
  static late Box<SearchHistory> searchHistory;     // 搜索历史表
  static late final Box<dynamic> setting;           // 设置表（键值对）

  // 初始化（在 main.dart 中调用）
  static Future<void> init() async {
    collectibles = await _openBoxSafe<CollectedBangumi>('collectibles');
    histories = await _openBoxSafe<History>('histories');
    downloads = await _openBoxSafe<DownloadRecord>('downloads');
    setting = await _openBoxSafe<dynamic>('setting');
    // ...
  }
}
```

### 设置项存取

```dart
// 读取设置（≈ application.properties 读取）
bool darkMode = GStorage.setting.get(SettingBoxKey.darkMode, defaultValue: false);
String proxyUrl = GStorage.setting.get(SettingBoxKey.proxyUrl, defaultValue: '');

// 写入设置
await GStorage.setting.put(SettingBoxKey.darkMode, true);
await GStorage.setting.put(SettingBoxKey.proxyUrl, 'http://127.0.0.1:7890');
```

`SettingBoxKey` 定义了所有设置项的 key（避免硬编码字符串）：
```dart
class SettingBoxKey {
  static const String themeMode = 'themeMode';
  static const String themeColor = 'themeColor';
  static const String danmakuArea = 'danmakuArea';
  static const String proxyUrl = 'proxyUrl';
  // ...
}
```

---

## 3. 数据模型定义

### 使用 @HiveType 和 @HiveField

```dart
// lib/modules/bangumi/bangumi_item.dart
@HiveType(typeId: 0)  // typeId 必须全局唯一（≈ 表名）
class BangumiItem {
  @HiveField(0)   // 字段索引，一旦发布不可修改
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

  // 构造函数
  BangumiItem({
    required this.id,
    required this.type,
    required this.name,
    required this.nameCn,
    required this.summary,
    required this.airDate,
  });
}
```

### 对照 JPA Entity

```java
// Java JPA
@Entity
@Table(name = "bangumi_item")
public class BangumiItem {
    @Id
    private int id;

    @Column(name = "type")
    private int type;

    @Column(name = "name")
    private String name;
}
```

### 关键规则

1. **typeId 全局唯一** — 每个 `@HiveType` 的 typeId 不能重复
2. **HiveField 索引不可修改** — 一旦发布，字段索引不能改变（否则旧数据无法读取）
3. **新增字段用新索引** — 添加新字段时使用下一个可用索引
4. **不能删除字段** — 只能废弃（保留索引但不再使用）

---

## 4. Repository 模式

### 接口定义

```dart
// lib/repositories/collect_repository.dart
abstract class ICollectRepository {
  Set<int> getBangumiIdsByType(CollectType type);
  Set<int> getBangumiIdsByTypes(List<CollectType> types);
  int getCollectType(int bangumiId);
}
```

### 实现类

```dart
class CollectRepository implements ICollectRepository {
  final Box<CollectedBangumi> _collectiblesBox = GStorage.collectibles;

  @override
  Set<int> getBangumiIdsByType(CollectType type) {
    try {
      return _collectiblesBox.values
          .where((item) => item.type == type.value)
          .map<int>((item) => item.bangumiItem.id)
          .toSet();
    } catch (e) {
      return <int>{};
    }
  }

  @override
  int getCollectType(int bangumiId) {
    try {
      final item = _collectiblesBox.values
          .firstWhere((item) => item.bangumiItem.id == bangumiId);
      return item.type;
    } catch (e) {
      return 0;
    }
  }
}
```

### CRUD 操作

```dart
// lib/repositories/collect_crud_repository.dart
abstract class ICollectCrudRepository {
  List<CollectedBangumi> getAllCollectibles();
  Future<void> addCollectible(BangumiItem item, int type);
  Future<void> deleteCollectible(int bangumiId);
}

class CollectCrudRepository implements ICollectCrudRepository {
  final Box<CollectedBangumi> _box = GStorage.collectibles;

  @override
  List<CollectedBangumi> getAllCollectibles() {
    return _box.values.toList();
  }

  @override
  Future<void> addCollectible(BangumiItem item, int type) async {
    final collected = CollectedBangumi(bangumiItem: item, type: type);
    await _box.put(item.id, collected);  // key = bangumiId
  }

  @override
  Future<void> deleteCollectible(int bangumiId) async {
    await _box.delete(bangumiId);
  }
}
```

对照 Spring Data JPA：
```java
public interface CollectRepository extends JpaRepository<CollectedBangumi, Integer> {
    List<CollectedBangumi> findByType(int type);
}
```

---

## 5. Box 操作速查

```dart
// 获取 Box 引用
final box = GStorage.collectibles;

// 写入（put = save）
await box.put('key', value);       // 指定 key
await box.add(value);              // 自动生成 key

// 读取
var item = box.get('key');                          // 按 key 获取
var item = box.get('key', defaultValue: fallback);  // 带默认值
var all = box.values.toList();                      // 获取所有

// 删除
await box.delete('key');
await box.clear();  // 清空整个 Box

// 查询（没有 SQL，用 Dart 集合操作）
var filtered = box.values
    .where((item) => item.type == 1)
    .toList();

// 检查
bool exists = box.containsKey('key');
int count = box.length;
```

---

## 6. 代码生成

修改了 `@HiveType` 类后，需要重新生成 TypeAdapter：

```bash
dart run build_runner build --delete-conflicting-outputs
```

生成的文件（`bangumi_item.g.dart`）包含序列化/反序列化逻辑：
```dart
// 自动生成，不要手动修改
class BangumiItemAdapter extends TypeAdapter<BangumiItem> {
  @override
  BangumiItem read(BinaryReader reader) {
    // 从二进制读取各字段
  }

  @override
  void write(BinaryWriter writer, BangumiItem obj) {
    // 将各字段写入二进制
  }
}
```

---

## 7. 数据模型一览

| 模型 | typeId | 用途 | 文件 |
|------|--------|------|------|
| `BangumiItem` | 0 | 番剧基本信息 | `modules/bangumi/bangumi_item.dart` |
| `CollectedBangumi` | 1 | 收藏记录 | `modules/collect/collect_module.dart` |
| `History` | 2 | 观看历史 | `modules/history/history_module.dart` |
| `SearchHistory` | 3 | 搜索历史 | `modules/search/search_history_module.dart` |
| `DownloadRecord` | 7 | 下载记录 | `modules/download/download_module.dart` |
| `DownloadEpisode` | 8 | 下载集数 | `modules/download/download_module.dart` |

---

## 8. 写入队列（并发安全）

`GStorage` 中有一个写入队列机制，确保收藏变更的顺序性：

```dart
static Future<T> _runCollectChangesWriteExclusive<T>(
  Future<T> Function() action,
) {
  final completer = Completer<T>();
  final previousWrite = _collectChangesWriteQueue;

  _collectChangesWriteQueue = (() async {
    try { await previousWrite; } catch (_) {}
    try {
      completer.complete(await action());
    } catch (e, stackTrace) {
      completer.completeError(e, stackTrace);
    }
  })();

  return completer.future;
}
```

这类似于 Java 的 `synchronized` 或 `ReentrantLock`，确保多个异步写入操作按顺序执行，避免数据竞争。

---

## 练习

1. 打开 `lib/modules/bangumi/bangumi_item.dart`，数一下有多少个 `@HiveField`
2. 打开 `lib/utils/storage.dart`，找出 `GStorage.init()` 方法初始化了哪些 Box
3. 如果你要新增一个"笔记"功能，需要：
   - 定义什么数据模型？
   - 在哪里注册 Box？
   - Repository 需要哪些方法？

<details>
<summary>答案提示</summary>

1. BangumiItem 有多个 HiveField（id, type, name, nameCn, summary, airDate, airWeekday, rank, images, tags 等）
2. `init()` 初始化了 collectibles, histories, downloads, setting, searchHistory, collectChanges, shieldList
3. 新增笔记功能需要：定义 `@HiveType(typeId: N) class Note`，在 `GStorage` 中添加 `static late Box<Note> notes`，创建 `INoteRepository` 接口和实现

</details>

---

下一章：[Dio 网络请求](./07-DIO-NETWORK.md)
