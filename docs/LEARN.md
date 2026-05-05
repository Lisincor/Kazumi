# Kazumi 开发者学习路径

本项目是 Flutter 开发的实战教程，通过阅读和修改本项目，你将学会 Flutter 开发。

## 学习顺序

按以下顺序阅读文件和目录，从小功能开始，逐步掌握整个项目。

---

### 第一阶段：基础概念（1-3天）

#### 1. Dart 语法基础
学习前先了解 Dart 语法，推荐资源：
- [Dart 官方文档](https://dart.dev/language)
- [Flutter Apprentice 书籍](https://www.flutterapprentice.com/)

**核心概念重点学：**
- 类与构造函数（注意本项目的工厂构造函数用法）
- 异步编程 `async/await` 和 `Future`
- 泛型 `<T>`
- Mixin（MobX 使用了 mixin）
- 注解 `@HiveType` `@observable` 等代码生成标记

### 2. 系统化教程

推荐使用 `docs/` 目录下的教学文档：

| 文档 | 说明 | 预计时间 |
|------|------|---------|
| [docs/00-START-HERE.md](docs/00-START-HERE.md) | 总览和路线图 | 30分钟 |
| [docs/01-DART-BASICS.md](docs/01-DART-BASICS.md) | Dart 基础语法 | 2-3小时 |
| [docs/02-APPLICATION.md](docs/02-APPLICATION.md) | 应用启动与初始化 | 1-2小时 |
| [docs/03-MOBX-STATE.md](docs/03-MOBX-STATE.md) | MobX 状态管理 | 2-3小时 |
| [docs/04-HIVE-STORAGE.md](docs/04-HIVE-STORAGE.md) | Hive 数据存储 | 2-3小时 |
| [docs/05-DIO-NETWORK.md](docs/05-DIO-NETWORK.md) | 网络请求 | 2-3小时 |
| [docs/06-PLUGINS.md](docs/06-PLUGINS.md) | 插件系统 | 2-3小时 |
| [docs/07-ROUTING.md](docs/07-ROUTING.md) | 路由系统 | 2-3小时 |
| [docs/08-DATA-MODELS.md](docs/08-DATA-MODELS.md) | 数据模型 | 2-3小时 |
| [docs/09-PRACTICAL.md](docs/09-PRACTICAL.md) | 实战开发 | 3-4小时 |

**完整学习路径**：约 20-26 小时（�� 2-3 周）

---

### 第二阶段：核心模块（3-7天）

```dart
// 本项目常见的构造方式
class Example {
  final String name;
  final int id;

  // 标准构造函数
  Example({required this.name, required this.id});

  // 命名构造函数
  factory Example.fromJson(Map<String, dynamic> json) {
    return Example(
      name: json['name'],
      id: json['id'],
    );
  }
}
```

#### 2. Flutter 基础概念

**必读文件：** `lib/pages/popular/popular_controller.dart`

这个文件包含了 Flutter 最常用的模式：
- StatelessWidget vs StatefulWidget
- 状态管理
- 生命周期

```dart
// 典型 MobX 状态管理写法
import 'package:mobx/mobx.dart';

part 'popular_controller.g.dart';

class PopularController = _PopularController with _$PopularController;

abstract class _PopularController with Store {
  // Observable 状态 - 响应式数据
  @observable
  int currentPage = 1;

  // Computed 计算属性
  @computed
  bool get hasMore => currentPage > 0;

  // Action 方法
  @action
  void nextPage() {
    currentPage++;
  }
}
```

---

### 第二阶段：核心模块（3-7天）

#### 3. 数据存储 - Hive
**必读文件：** `lib/utils/storage.dart`

```dart
// Hive 模型定义方式
@HiveType(typeId: 0)
class BangumiItem {
  @HiveField(0)
  int id;

  @HiveField(1)
  String name;
}

// 存储使用
final box = await Hive.openBox('myBox');
await box.put('key', value);
var data = box.get('key');
```

**学习要点：**
- 理解 `Box<T>` 是 Hive 的数据库表
- 理解 `@HiveField(n)` 映射字段到索引
- 理解 `part` 文件是代码生成的结果
- GStorage 单例封装了所有存储操作

#### 4. HTTP 请求 - Dio
**必读文件：** `lib/request/request.dart`

```dart
// Dio GET 请求
var resp = await Request().get(url);

// Dio POST 请求
var resp = await Request().post(url, data: queryParams);
```

**学习要点：**
- Request 单例封装了 Dio
- 请求拦截器处理通用逻辑
- Options 配置请求头

#### 5. 插件系统
**必读文件：** `lib/plugins/plugins.dart`

理解本项目的核心功能：如何用 XPath 解析网页获取视频源。

```dart
// Plugin 类核心字段
class Plugin {
  String searchURL;      // 搜索 URL
  String searchList;    // 结果列表 XPath
  String searchName;    // 番剧名 XPath
  String searchResult;   // 链接 XPath
  String chapterRoads;  // 剧集列表 XPath
  String chapterResult;  // 剧集链接 XPath
}
```

---

### 第三阶段：页面与路由（7-14天）

#### 6. 路由系统 - Flutter Modular
**必读文件：** `lib/pages/index_module.dart`

```dart
// 模块定义
class IndexModule extends Module {
  @override
  void binds(i) {
    // 依赖注入
    i.addSingleton(MyService.new);
  }

  @override
  void routes(r) {
    // 路由配置
    r.child("/home", child: (_) => HomePage());
    r.module("/detail", module: DetailModule());
  }
}

// 页面跳转
Modular.to.pushNamed('/detail');
Modular.to.pop(); // 返回
```

**学习要点：**
- `Module` 定义路由和依赖
- `binds(i)` 注册依赖注入
- `routes(r)` 定义页面路径

#### 7. 页面结构
**对比阅读：**
- `lib/pages/popular/` - 简单页面
- `lib/pages/info/` - 复杂页面

每个页面通常有四个文件：
```
info/
├── info_controller.dart    # MobX 状态管理
├── info_controller.g.dart  # 代码生成，不要修改
├── info_module.dart        # 路由和依赖
└── info_page.dart          # UI 界面
```

---

### 第四阶段：高级功能（14-21天）

#### 8. 响应式状态 - MobX
**必读文件：** `lib/pages/my/my_controller.dart`

```dart
// 完整的 MobX Store
class MyController = _MyController with _$MyController;

abstract class _MyController with Store {
  @observable
  ObservableList<Item> items = ObservableList();

  @observable
  bool isLoading = false;

  @computed
  int get itemCount => items.length;

  @action
  Future<void> loadItems() async {
    isLoading = true;
    // 异步加载数据
    isLoading = false;
  }
}
```

#### 9. Repository 模式
**必读文件：** `lib/repositories/collect_repository.dart`

```dart
// 数据访问接口
abstract class ICollectRepository {
  Set<int> getBangumiIdsByType(CollectType type);
}

// 数据访问实现
class CollectRepository implements ICollectRepository {
  final _collectiblesBox = GStorage.collectibles;

  @override
  Set<int> getBangumiIdsByType(CollectType type) {
    return _collectiblesBox.values
        .where((item) => item.type == type.value)
        .map<int>((item) => item.bangumiItem.id)
        .toSet();
  }
}
```

---

## 关键文件索引

| 阶段 | 文件路径 | 说明 |
|-----|---------|------|
| 基础 | `lib/main.dart` | 应用入口，了解初始化流程 |
| 基础 | `pubspec.yaml` | 依赖管理，了解项目使用的库 |
| 状态 | `lib/pages/popular/popular_controller.dart` | MobX 状态管理示例 |
| 数据 | `lib/utils/storage.dart` | Hive 存储封装 |
| 网络 | `lib/request/request.dart` | Dio HTTP 请求 |
| 路由 | `lib/pages/index_module.dart` | Flutter Modular 路由 |
| 插件 | `lib/plugins/plugins.dart` | 规则解析核心 |
| 模型 | `lib/modules/bangumi/bangumi_item.dart` | 数据模型定义 |

---

## 代码生成

修改以下文件后需要运行代码生成：

```bash
# 生成 MobX 代码
dart run build_runner build

# 删除生成文件后重新生成
dart run build_runner build --delete-conflicting-outputs
```

生成的标志：
- `*.g.dart` 文件
- `lib/hive_registrar.g.dart`

---

## 调试技巧

1. **查看日志**：使用 `KazumiLogger()` 打印日志
   ```dart
   KazumiLogger().i('信息日志');
   KazumiLogger().w('警告日志');
   KazumiLogger().e('错误日志', error: e);
   ```

2. **热重载**：桌面端按 `R` 键，Android/iOS 保存后自动刷新

3. **断点调试**：VS Code 在行号旁点击即可设置断点

---

## 开发检查清单

修改代码前先检查：

- [ ] 新增页面需要创建 4 个文件（controller, module, page, controller.g.dart）
- [ ] 新增数据模型需要添加 Hive TypeAdapter 并注册
- [ ] 修改 controller 后需要运行 `build_runner`
- [ ] 添加新依赖后运行 `flutter pub get`
- [ ] 提交前运行 `flutter analyze` 检查代码风格

---

## 下一步

学完本项目后，推荐深入学习：

1. **Riverpod** - 另一种状态管理方案
2. **GoRouter** - 官方推荐的路由方案
3. **GetX** - 国内流行的全站式方案
4. **测试驱动开发** - 编写单元测试和集成测试