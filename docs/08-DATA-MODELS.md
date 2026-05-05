# 第八课：数据模型

> 📖 学习目标：理解项目中各个数据模型的设计，掌握模型的定义和使用
> ⏱️ 预计时间：2-3 小时
> 📁 配套源码：`lib/modules/` 目录下的各个模型文件

---

## 目录

1. [模型概述](#1-模型概述)
2. [BangumiItem 番剧模型](#2-bangumiitem-番剧模型)
3. [CollectedBangumi 收藏模型](#3-collectedbangumi-收藏模型)
4. [History 观看历史](#4-history-观看历史)
5. [Download 下载模型](#5-download-下载模型)
6. [EpisodeInfo 剧集模型](#6-episodeinfo-剧集模型)
7. [SearchItem 搜索结果](#7-searchitem-搜索结果)
8. [Road 播放列表](#8-road-播放列表)
9. [模型关系图](#9-模型关系图)
10. [创建新模型](#10-创建新模型)

---

## 1. 模型概述

### 1.1 什么是数据模型？

数据模型是程序中用于描述"事物"的结构。就像：

```
现实世界          →      程序世界
────────────────────────────────
番剧              →      BangumiItem
收藏记录          →      CollectedBangumi
观看历史          →      History
下载任务          →      DownloadRecord
搜索结果          →      SearchItem
```

### 1.2 模型分类

| 类型 | 说明 | 存储方式 |
|------|------|---------|
| Hive 模型 | 需要持久化存储 | Hive TypeAdapter |
| 普通模型 | 只在内存中使用 | 普通 Dart 类 |
| API 模型 | 从服务器获取 | JSON 解析 |

### 1.3 项目中的模型

```
lib/modules/
├── bangumi/
│   ├── bangumi_item.dart       ← 番剧基本信息
│   ├── bangumi_tag.dart       ← 标签
│   ├── episode_item.dart      ← 剧集信息
│   └── bangumi_collection.dart ��� 收藏信息（Bangumi API）
│
├── collect/
│   ├── collect_module.dart    ← 收藏记录
│   └── collect_type.dart      ← 收藏类型枚举
│
├── download/
│   └── download_module.dart   ← 下载记录
│
├── history/
│   └── history_module.dart     ← 观看历史
│
└── search/
    └── plugin_search_module.dart ← 搜索结果
```

---

## 2. BangumiItem 番剧模型

📁 源码：`lib/modules/bangumi/bangumi_item.dart`

### 2.1 模型定义

```dart
@HiveType(typeId: 0)  // Hive 类型标识
class BangumiItem {
  @HiveField(0)  int id;              // Bangumi ID
  @HiveField(1)  int type;            // 类型（动画/书籍/音乐等）
  @HiveField(2)  String name;         // 原名
  @HiveField(3)  String nameCn;       // 中文名
  @HiveField(4)  String summary;      // 简介
  @HiveField(5)  String airDate;       // 首播日期
  @HiveField(6)  int airWeekday;      // 首播星期（1-7）
  @HiveField(7)  int rank;            // 排名
  @HiveField(8)  Map<String, String> images;  // 图片
  @HiveField(9)  List<BangumiTag> tags;        // 标签列表
  @HiveField(10) List<String> alias;           // 别名
  @HiveField(11) double ratingScore;           // 评分
  @HiveField(12) int votes;                    // 投票数
  @HiveField(13) List<int> votesCount;         // 各分数投票数
  @HiveField(14) String info;                  // 详细信息
}
```

### 2.2 构造函数

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

### 2.3 JSON 解析

```dart
factory BangumiItem.fromJson(Map<String, dynamic> json) {
  return BangumiItem(
    id: json['id'],
    type: json['type'] ?? 2,
    name: json['name'] ?? '',
    nameCn: (json['name_cn'] ?? '') == ''
        ? (((json['nameCN'] ?? '') == '') ? json['name'] : json['nameCN'])
        : json['name_cn'],
    summary: json['summary'] ?? '',
    airDate: json['date'] ?? '',
    airWeekday: Utils.dateStringToWeekday(json['date'] ?? '2000-11-11'),
    rank: json['rating']['rank'] ?? 0,
    images: Map<String, String>.from(json['images'] ?? {}),
    tags: (json['tags'] as List?)?.map((i) => BangumiTag.fromJson(i)).toList() ?? [],
    alias: bangumiAlias,
    ratingScore: double.parse((json['rating']['score'] ?? 0.0).toStringAsFixed(1)),
    votes: json['rating']['total'] ?? 0,
    votesCount: voteList,
    info: json['info'] ?? '',
  );
}
```

### 2.4 BangumiTag 标签

📁 源码：`lib/modules/bangumi/bangumi_tag.dart`

```dart
@HiveType(typeId: 6)
class BangumiTag {
  @HiveField(0)
  int id;

  @HiveField(1)
  String name;

  @HiveField(2, defaultValue: 0)
  int count;

  BangumiTag({
    required this.id,
    required this.name,
    this.count = 0,
  });

  factory BangumiTag.fromJson(Map<String, dynamic> json) {
    return BangumiTag(
      id: json['id'] ?? 0,
      name: json['name'] ?? '',
      count: json['count'] ?? 0,
    );
  }
}
```

---

## 3. CollectedBangumi 收藏模型

📁 源码：`lib/modules/collect/collect_module.dart`

### 3.1 模型定义

```dart
@HiveType(typeId: 3)
class CollectedBangumi {
  @HiveField(0)  BangumiItem bangumiItem;  // 番剧信息
  @HiveField(1)  DateTime time;           // 收藏时间
  @HiveField(2)  int type;               // 收藏类型

  String get key => bangumiItem.id.toString();

  CollectedBangumi(this.bangumiItem, this.time, this.type);
}
```

### 3.2 收藏类型枚举

📁 源码：`lib/modules/collect/collect_type.dart`

```dart
enum CollectType {
  none(0, '未收藏'),
  watching(1, '在看'),
  planToWatch(2, '想看'),
  onHold(3, '搁置'),
  watched(4, '看过'),
  abandoned(5, '抛弃');

  const CollectType(this.value, this.label);
  final int value;    // 存储用
  final String label;  // 显示用

  static CollectType fromValue(int value) {
    return CollectType.values.firstWhere(
      (type) => type.value == value,
      orElse: () => CollectType.none,
    );
  }

  bool get isCollected => this != CollectType.none;
}
```

### 3.3 使用示例

```dart
// 添加收藏
var bangumi = BangumiItem(...);
var collected = CollectedBangumi(bangumi, DateTime.now(), CollectType.watching.value);
await GStorage.collectibles.put(bangumi.id.toString(), collected);

// 读取收藏
var collected = GStorage.collectibles.get(bangumiId.toString());
if (collected != null) {
  final type = CollectType.fromValue(collected.type);
  print('收藏状态: ${type.label}');
}
```

### 3.4 收藏变化记录

📁 源码：`lib/modules/collect/collect_change_module.dart`

```dart
@HiveType(typeId: 5)
class CollectedBangumiChange {
  @HiveField(0) int id;              // 变化 ID
  @HiveField(1) int bangumiID;       // 番剧 ID
  @HiveField(2) int action;           // 动作 (1=add, 2=update, 3=delete)
  @HiveField(3) int type;            // 收藏类型
  @HiveField(4) int timestamp;      // 时间戳

  CollectedBangumiChange(this.id, this.bangumiID, this.action, this.type, this.timestamp);
}
```

这个模型用于记录所有收藏变化，支持多设备同步。

---

## 4. History 观看历史

📁 源码：`lib/modules/history/history_module.dart`

### 4.1 模型定义

```dart
@HiveType(typeId: 1)
class History {
  @HiveField(0)  Map<int, Progress> progresses = {};    // 播放进度
  @HiveField(1)  int lastWatchEpisode;                  // 最后观看的剧集
  @HiveField(2)  String adapterName;                   // 插件名称
  @HiveField(3)  BangumiItem bangumiItem;               // 番剧信息
  @HiveField(4)  DateTime lastWatchTime;               // 最后观看时间
  @HiveField(5)  String lastSrc;                        // 最后播放地址
  @HiveField(6, defaultValue: '') String lastWatchEpisodeName;  // 最后观看的剧集名

  String get key => adapterName + bangumiItem.id.toString();

  History(
    this.bangumiItem,
    this.lastWatchEpisode,
    this.adapterName,
    this.lastWatchTime,
    this.lastSrc,
    this.lastWatchEpisodeName,
  );
}
```

### 4.2 Progress 播放进度

```dart
@HiveType(typeId: 2)
class Progress {
  @HiveField(0)  int episode;             // 剧集号
  @HiveField(1)  int road;               // 播放源序号
  @HiveField(2)  int _progressInMilli;   // 进度（毫秒）

  // 转换为 Duration
  Duration get progress => Duration(milliseconds: _progressInMilli);
  set progress(Duration d) => _progressInMilli = d.inMilliseconds;

  Progress(this.episode, this.road, this._progressInMilli);
}
```

### 4.3 使用示例

```dart
// 保存历史
var history = History(
  bangumiItem,
  episode: 5,
  adapterName: pluginName,
  lastWatchTime: DateTime.now(),
  lastSrc: videoUrl,
  lastWatchEpisodeName: '第5集',
);
await GStorage.histories.put(history.key, history);

// 保存进度
history.progresses[episode] = Progress(episode, road, position.inMilliseconds);
await GStorage.histories.put(history.key, history);

// 读取历史
var allHistory = GStorage.histories.values.toList();
allHistory.sort((a, b) => b.lastWatchTime.compareTo(a.lastWatchTime));
```

---

## 5. Download 下载模型

📁 源码：`lib/modules/download/download_module.dart`

### 5.1 DownloadRecord 下载记录

```dart
@HiveType(typeId: 7)
class DownloadRecord {
  @HiveField(0)  int bangumiId;                    // 番剧 ID
  @HiveField(1)  String bangumiName;               // 番剧名称
  @HiveField(2)  String bangumiCover;              // 封面图
  @HiveField(3)  String pluginName;               // 插件名称
  @HiveField(4)  Map<int, DownloadEpisode> episodes;  // 剧集列表
  @HiveField(5)  DateTime createdAt;              // 创建时间

  String get key => '${pluginName}_$bangumiId';
}
```

### 5.2 DownloadEpisode 下载剧集

```dart
@HiveType(typeId: 8)
class DownloadEpisode {
  @HiveField(0)  int episodeNumber;       // 剧集号
  @HiveField(1)  String episodeName;      // 剧集名
  @HiveField(2)  int road;               // 播放源
  @HiveField(3)  int status;            // 状态
  @HiveField(4)  double progressPercent; // 进度百分比
  @HiveField(5)  int totalSegments;     // 总分片数
  @HiveField(6)  int downloadedSegments; // 已下载分片数
  @HiveField(7)  String localM3u8Path;   // 本地 M3U8 路径
  @HiveField(8)  String downloadDirectory;  // 下载目录
  @HiveField(9)  String networkM3u8Url;  // 网络 M3U8 URL
  @HiveField(10) DateTime? completedAt;  // 完成时间
  @HiveField(11, defaultValue: '') String errorMessage;  // 错误信息
  @HiveField(12, defaultValue: 0) int totalBytes;  // 总字节数
  @HiveField(13, defaultValue: '') String episodePageUrl;  // 剧集页面 URL
  @HiveField(14, defaultValue: '') String danmakuData;  // 弹幕数据
  @HiveField(15, defaultValue: 0) int danDanBangumiID;  // 弹弹番剧 ID
}
```

### 5.3 DownloadStatus 状态常量

```dart
class DownloadStatus {
  static const int pending = 0;      // 等待中
  static const int resolving = 1;  // 解析中
  static const int downloading = 2; // 下载中
  static const int completed = 3;   // 已完成
  static const int failed = 4;     // 失败
  static const int paused = 5;     // 暂停
}
```

### 5.4 使用示例

```dart
// 创建下载记录
var episode = DownloadEpisode(
  1, '第1集', 0,
  DownloadStatus.pending,
  0.0,  // 进度
  0, 0,  // 分片
  '', '', '',  // 路径
  null,
  '',
  0,
  '',
);
var record = DownloadRecord(
  bangumiId, bangumiName, cover,
  pluginName,
  {1: episode},
  DateTime.now(),
);
await GStorage.downloads.put(record.key, record);

// 更新进度
var record = GStorage.downloads.get(key);
record.episodes[1]!.progressPercent = 0.5;
record.episodes[1]!.status = DownloadStatus.downloading;
await GStorage.downloads.put(key, record);
```

---

## 6. EpisodeInfo 剧集模型

📁 源码：`lib/modules/bangumi/episode_item.dart`

### 6.1 模型定义

```dart
class EpisodeInfo {
  int id;           // 剧集 ID
  num episode;      // 集数（可能是小数，1.5 表示 SP）
  int type;         // 类型
  String name;      // 名称
  String nameCn;    // 中文名

  EpisodeInfo({
    required this.id,
    required this.episode,
    required this.type,
    required this.name,
    required this.nameCn,
  });
}
```

这是一个**非 Hive 模型**，只用于从 API 获取数据。

### 6.2 剧集类型

```dart
String readType() {
  switch (type) {
    case 0: return 'ep';   // 正篇
    case 1: return 'sp';   // 特别篇
    case 2: return 'op';   // 片头曲
    case 3: return 'ed';   // 片尾曲
    default: return '';
  }
}
```

### 6.3 JSON 解析

```dart
factory EpisodeInfo.fromJson(Map<String, dynamic> json) {
  return EpisodeInfo(
    id: json['id'] ?? 0,
    episode: json['sort'] ?? 0,  // 注意：字段名是 sort
    type: json['type'] ?? 0,
    name: json['name'] ?? '',
    nameCn: json['name_cn'] ?? '',
  );
}
```

---

## 7. SearchItem 搜索结果

📁 源码：`lib/modules/search/plugin_search_module.dart`

### 7.1 模型定义

```dart
class SearchItem {
  String name;   // 番剧名称
  String src;    // 详情页 URL

  SearchItem({required this.name, required this.src});
}
```

### 7.2 PluginSearchResponse 响应

```dart
class PluginSearchResponse {
  String pluginName;       // 插件名称
  List<SearchItem> data;   // 搜索结果

  PluginSearchResponse({required this.pluginName, required this.data});

  factory PluginSearchResponse.fromJson(Map<String, dynamic> json) {
    return PluginSearchResponse(
      pluginName: json['pluginName'],
      data: (json['data'] as List)
          .map((itemJson) => SearchItem.fromJson(itemJson))
          .toList(),
    );
  }
}
```

---

## 8. Road 播放列表

📁 源码：`lib/modules/roads/road_module.dart`

### 8.1 模型定义

```dart
class Road {
  String name;              // 播放列表名称（如"第1季"）
  List<String> data;         // 剧集 URL 列表
  List<String> identifier;  // 剧集名称列表

  Road({
    required this.name,
    required this.data,
    required this.identifier,
  });
}
```

### 8.2 使用示例

```dart
// 遍历播放列表
for (var road in roadList) {
  print('播放列表: ${road.name}');
  for (int i = 0; i < road.data.length; i++) {
    print('  ${road.identifier[i]}: ${road.data[i]}');
  }
}
```

---

## 9. 模型关系图

### 9.1 关系概览

```
BangumiItem (番剧)
    │
    ├── 被引用 ───────────────────→ CollectedBangumi (收藏)
    │                               │
    │                               └──→ CollectedBangumiChange (变化记录)
    │
    ├── 被引用 ───────────────────→ History (历史)
    │                               │
    │                               └──→ Progress (进度)
    │
    └── 被引用 ───────────────────→ DownloadRecord (下载)
                                    │
                                    └──→ DownloadEpisode (剧集)
```

### 9.2 数据流向

```
网络 API / Plugin
    │
    ├── BangumiItem ← BangumiHTTP
    ├── EpisodeInfo ← BangumiHTTP
    └── SearchItem ← Plugin.queryBangumi
         │
         ↓
    用户选择
         │
         ↓
    Plugin.querychapterRoads → Road (播放列表)
         │
         ↓
    播放视频 → History (记录历史)
         │
         ↓
    收藏 → CollectedBangumi
         │
         ↓
    下载 → DownloadRecord
```

### 9.3 存储对应

| 数据 | Box | 存储模型 |
|------|-----|---------|
| 番剧信息 | favorites | BangumiItem |
| 收藏 | collectibles | CollectedBangumi |
| 变化记录 | collectchanges | CollectedBangumiChange |
| 历史 | histories | History |
| 设置 | setting | dynamic |
| 下载 | downloads | DownloadRecord |

---

## 10. 创建新模型

### 10.1 需求

假设要创建一个"观看向导"功能，记录用户是否看过某个番剧的介绍。

### 10.2 步骤 1：定义模型

```dart
import 'package:hive_ce/hive.dart';

part 'guide_module.g.dart';

@HiveType(typeId: 10)
class Guide {
  @HiveField(0)
  int bangumiId;

  @HiveField(1)
  bool hasSeenIntro;

  @HiveField(2, defaultValue: false)
  bool hasSeenCharacter;

  @HiveField(3, defaultValue: false)
  bool hasSeenStaff;

  @HiveField(4, defaultValue: false)
  bool hasSeenComment;

  Guide({
    required this.bangumiId,
    this.hasSeenIntro = false,
    this.hasSeenCharacter = false,
    this.hasSeenStaff = false,
    this.hasSeenComment = false,
  });

  String get key => bangumiId.toString();
}
```

### 10.2 步骤 2：注册 TypeAdapter

在 `lib/utils/storage.dart` 中添加：

```dart
// 在 GStorage.init() 中
Hive.registerAdapter(GuideAdapter());

// 添加 Box
static late Box<Guide> guides;

// 打开 Box
guides = await _openBoxSafe<Guide>('guides');
```

### 10.3 步骤 3：生成代码

```bash
dart run build_runner build
```

### 10.4 步骤 4：使用

```dart
// 保存
var guide = Guide(bangumiId: 123);
guide.hasSeenIntro = true;
await GStorage.guides.put(guide.key, guide);

// 读取
var guide = GStorage.guides.get(123.toString());
if (guide != null && guide.hasSeenIntro) {
  // 跳过介绍页
}
```

---

## 小练习

### 练习 1：理解模型关系

回答以下问题：

1. `CollectedBangumi` 和 `BangumiItem` 是什么关系？
2. `Progress` 存储的 `_progressInMilli` 是什么单位？
3. `DownloadEpisode.status` 为 2 表示什么状态？

<details>
<summary>答案</summary>

1. `CollectedBangumi` 包含 `BangumiItem` 作为字段，是组合关系
2. `_progressInMilli` 是毫秒（Milliseconds）
3. `DownloadStatus.downloading` = 2，表示"下载中"

</details>

### 练习 2：创建新模型

创建一个"评分记录"模型，包含：
- `bangumiId` (int)
- `rating` (double，0.0-10.0)
- `ratedAt` (DateTime)

<details>
<summary>答案</summary>

```dart
@HiveType(typeId: 11)
class Rating {
  @HiveField(0)
  int bangumiId;

  @HiveField(1)
  double rating;

  @HiveField(2)
  DateTime ratedAt;

  Rating({
    required this.bangumiId,
    required this.rating,
    required this.ratedAt,
  });

  String get key => bangumiId.toString();
}
```

</details>

### 练习 3：扩展 History 模型

在 `History` 模型中添加一个新字段 `lastWatchPosition`，记录最后观看的进度（毫秒）。

<details>
<summary>答案</summary>

```dart
@HiveField(7, defaultValue: 0)
int lastWatchPosition;  // 毫秒

// 或使用 Duration 更语义化
@HiveField(7, defaultValue: 0)
int _lastWatchPositionMillis;

// 不需要加 getter/setter（直接用毫秒更简单）
```

</details>

---

## 下一课预告

下一课我们将学习 [实战：开发新页面](./09-PRACTICAL.md)，通过实际案例完整地开发一个新功能。