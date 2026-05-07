# 第十一章：实战 — 从零开发一个新功能

> 目标：综合运用前面所学，完整走一遍"需求 → 设计 → 编码 → 测试"的流程
> 预计时间：3-4 小时

---

## 实战任务：添加"观看统计"功能

### 需求描述

在"我的"页面添加一个观看统计卡片，显示：
- 总观看集数
- 总观看时长（小时）
- 本周观看集数

数据来源：已有的 `History` 模型（`lib/modules/history/history_module.dart`）

---

## Step 1: 分析现有代码

### 1.1 了解 History 模型

打开 `lib/modules/history/history_module.dart`，了解历史记录的数据结构：

```dart
@HiveType(typeId: 2)
class History {
  @HiveField(0)
  int episodeNumber;      // 集数

  @HiveField(1)
  String title;           // 番剧标题

  @HiveField(2)
  int progress;           // 观看进度（秒）

  @HiveField(3)
  DateTime lastWatched;   // 最后观看时间
  // ...
}
```

### 1.2 了解 HistoryRepository

打开 `lib/repositories/history_repository.dart`，看看已有哪些方法可以复用。

### 1.3 了解"我的"页面

打开 `lib/pages/my/my_controller.dart` 和 `lib/pages/my/my_page.dart`，了解现有结构。

---

## Step 2: 设计方案

### 方案选择

因为统计数据是从已有的 History Box 计算得出，不需要新的数据模型或 Repository。
我们只需要：
1. 在 `MyController` 中添加统计计算方法
2. 在 `MyPage` 中添加统计卡片 UI

### 数据流

```
GStorage.histories (Hive Box)
      │
      ▼
MyController.loadStats()  ← 计算统计数据
      │
      ▼
MyPage (Observer)  ← 显示统计卡片
```

---

## Step 3: 修改 Controller

打开 `lib/pages/my/my_controller.dart`，添加统计相关的状态和方法：

```dart
// 在 _MyController 类中添加：

// ===== 观看统计 =====
@observable
int totalEpisodes = 0;

@observable
double totalHours = 0.0;

@observable
int weekEpisodes = 0;

void loadStats() {
  final histories = GStorage.histories.values.toList();

  // 总观看集数
  totalEpisodes = histories.length;

  // 总观看时长（progress 是秒，转换为小时）
  int totalSeconds = 0;
  for (var h in histories) {
    totalSeconds += h.progress;
  }
  totalHours = totalSeconds / 3600.0;

  // 本周观看集数
  final now = DateTime.now();
  final weekStart = now.subtract(Duration(days: now.weekday - 1));
  final startOfWeek = DateTime(weekStart.year, weekStart.month, weekStart.day);

  weekEpisodes = histories
      .where((h) => h.lastWatched.isAfter(startOfWeek))
      .length;
}
```

---

## Step 4: 修改 Page

打开 `lib/pages/my/my_page.dart`，添加统计卡片：

```dart
// 在 initState 中调用
@override
void initState() {
  super.initState();
  controller.loadStats();  // 加载统计数据
}

// 在 build 方法的合适位置添加统计卡片
Widget _buildStatsCard(BuildContext context) {
  final colorScheme = Theme.of(context).colorScheme;

  return Observer(
    builder: (_) => Card(
      margin: EdgeInsets.all(16),
      child: Padding(
        padding: EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text(
              '观看统计',
              style: Theme.of(context).textTheme.titleMedium,
            ),
            SizedBox(height: 16),
            Row(
              mainAxisAlignment: MainAxisAlignment.spaceAround,
              children: [
                _buildStatItem(
                  context,
                  '${controller.totalEpisodes}',
                  '总集数',
                  Icons.play_circle_outline,
                ),
                _buildStatItem(
                  context,
                  '${controller.totalHours.toStringAsFixed(1)}h',
                  '总时长',
                  Icons.access_time,
                ),
                _buildStatItem(
                  context,
                  '${controller.weekEpisodes}',
                  '本周',
                  Icons.calendar_today,
                ),
              ],
            ),
          ],
        ),
      ),
    ),
  );
}

Widget _buildStatItem(
  BuildContext context,
  String value,
  String label,
  IconData icon,
) {
  return Column(
    children: [
      Icon(icon, color: Theme.of(context).colorScheme.primary),
      SizedBox(height: 8),
      Text(
        value,
        style: Theme.of(context).textTheme.headlineSmall?.copyWith(
          fontWeight: FontWeight.bold,
        ),
      ),
      SizedBox(height: 4),
      Text(
        label,
        style: Theme.of(context).textTheme.bodySmall?.copyWith(
          color: Theme.of(context).colorScheme.onSurfaceVariant,
        ),
      ),
    ],
  );
}
```

---

## Step 5: 生成代码

```bash
dart run build_runner build --delete-conflicting-outputs
```

如果你添加了 `@observable` 注解，必须重新生成 `.g.dart` 文件。

---

## Step 6: 测试

### 6.1 运行应用

```bash
flutter run
```

### 6.2 验证功能

1. 打开"我的"页面，确认统计卡片显示
2. 观看一集番剧，返回"我的"页面，确认数据更新
3. 检查边界情况：没有历史记录时显示 0

### 6.3 代码分析

```bash
flutter analyze
```

确保没有类型错误或警告。

---

## Step 7: 代码审查清单

- [ ] `@observable` 字段是否都需要响应式？（统计数据需要，因为 UI 要监听）
- [ ] 是否处理了空数据情况？（histories 为空时不会崩溃）
- [ ] 是否遵循了项目的代码风格？（命名、缩进、文件组织）
- [ ] 是否需要新的依赖？（本例不需要）
- [ ] 是否影响了其他功能？（只是添加，不修改现有逻辑）

---

## 扩展练习

### 练习 1：添加"最常看的番剧"

在统计卡片下方显示观看次数最多的 3 部番剧。

提示：
```dart
// 按标题分组统计
Map<String, int> countByTitle = {};
for (var h in histories) {
  countByTitle[h.title] = (countByTitle[h.title] ?? 0) + 1;
}
// 排序取前 3
var top3 = countByTitle.entries.toList()
  ..sort((a, b) => b.value.compareTo(a.value));
```

### 练习 2：添加新页面"统计详情"

创建一个独立的统计详情页面，包含：
- 按日期的观看趋势图
- 按番剧的观看时长排行

需要：
1. 创建 `lib/pages/stats/stats_controller.dart`
2. 创建 `lib/pages/stats/stats_page.dart`
3. 创建 `lib/pages/stats/stats_module.dart`
4. 在 `IndexModule` 中注册
5. 从"我的"页面添加跳转入口

### 练习 3：添加数据导出

将观看历史导出为 JSON 文件，支持分享。

提示：
```dart
import 'dart:convert';
import 'dart:io';

Future<void> exportHistory() async {
  final histories = GStorage.histories.values.toList();
  final json = jsonEncode(histories.map((h) => h.toJson()).toList());
  final file = File('${downloadPath}/kazumi_history.json');
  await file.writeAsString(json);
}
```

---

## 开发流程总结

```
1. 分析需求
   └── 确定需要修改哪些文件

2. 查看现有代码
   └── 找到可复用的模型、方法、组件

3. 设计方案
   ├── 需要新的数据模型吗？→ 修改 modules/
   ├── 需要新的数据访问吗？→ 修改 repositories/
   ├── 需要新的网络请求吗？→ 修改 request/
   └── 需要新的页面吗？→ 创建 pages/xxx/

4. 编码
   ├── Controller（状态和逻辑）
   ├── Page（UI）
   └── Module（路由，如果是新页面）

5. 代码生成
   └── dart run build_runner build --delete-conflicting-outputs

6. 测试
   ├── flutter run（功能验证）
   ├── flutter analyze（静态分析）
   └── flutter test（单元测试）

7. 提交
   └── git commit
```

---

## 常见问题

### Q: 修改了 Controller 但 UI 没更新？

检查：
1. 字段是否标记了 `@observable`？
2. 是否重新运行了 `build_runner`？
3. UI 是否用 `Observer` 包裹了？

### Q: 热重载后状态丢失？

Controller 是单例，热重载不会重新创建。如果修改了 Controller 的初始值，需要热重启（按 R）。

### Q: 新页面路由 404？

检查：
1. Module 是否在 `IndexModule.routes()` 中注册？
2. 路由路径是否正确（注意前导 `/`）？
3. Controller 是否在 `IndexModule.binds()` 中注册？

### Q: build_runner 报错？

常见原因：
1. `part` 声明文件名不匹配
2. 类名模式不对（`class A = _A with _$A`）
3. 有语法错误导致分析失败

解决：先运行 `flutter analyze` 修复语法错误，再运行 build_runner。

---

恭喜你完成了整个教程！你现在应该能够：
- 读懂项目中的任何 Dart/Flutter 代码
- 理解数据从 UI 到存储的完整流向
- 独立开发新功能并集成到项目中

继续探索项目代码，遇到不懂的模式回来查阅对应章节。
