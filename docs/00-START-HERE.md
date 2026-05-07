# Kazumi Flutter 开发教程：面向项目学习

> 适用读者：有 Java/Spring Boot 开发经验，零 Dart/Flutter 基础的开发者

## 教学理念

本教程不是"先学语法再写项目"的传统模式。我们直接从 Kazumi 项目出发，每一章都围绕一个真实模块展开，在解决实际问题的过程中学习 Dart 和 Flutter。

你已经有 Spring Boot 的经验，很多概念可以直接迁移：
- Controller → MobX Store（状态管理）
- Spring IoC → Flutter Modular（依赖注入）
- JPA Repository → Hive Repository（数据持久化）
- RestTemplate/OkHttp → Dio（HTTP 客户端）
- application.yml → Hive Box（配置存储）

## 教程目录

```
docs/
├── 00-START-HERE.md           ← 你在这里
├── 01-DART-FOR-JAVA-DEVS.md   ← Dart 语法（对照 Java）
├── 02-FLUTTER-BASICS.md        ← Flutter Widget 体系与生命周期
├── 03-PROJECT-ARCHITECTURE.md  ← 项目架构与数据流
├── 04-MODULAR-ROUTING.md       ← 路由与依赖注入（对照 Spring IoC）
├── 05-MOBX-STATE.md            ← 响应式状态管理
├── 06-HIVE-STORAGE.md          ← 本地持久化（对照 JPA）
├── 07-DIO-NETWORK.md           ← 网络请求（对照 RestTemplate）
├── 08-PLUGIN-SYSTEM.md         ← 插件系统与 XPath 解析
├── 09-UI-COMPONENTS.md         ← UI 组件与主题系统
├── 10-VIDEO-PLAYER.md          ← 视频播放与弹幕
└── 11-PRACTICAL-GUIDE.md       ← 实战：从零开发一个新功能
```

## 学习路径

### 第一阶段：看懂代码（3-4 天）

| 章节 | 目标 | 对应 Java 概念 |
|------|------|---------------|
| 01 Dart 语法 | 能读懂项目中的 Dart 代码 | Java 语法差异 |
| 02 Flutter 基础 | 理解 Widget 树和 UI 构建方式 | JSP/Thymeleaf → 声明式 UI |
| 03 项目架构 | 理解整体分层和数据流向 | Spring 分层架构 |

### 第二阶段：理解核心模块（4-5 天）

| 章节 | 目标 | 对应 Java 概念 |
|------|------|---------------|
| 04 路由与 DI | 理解页面导航和依赖注入 | Spring IoC + Spring MVC |
| 05 MobX 状态 | 理解响应式数据绑定 | 观察者模式 |
| 06 Hive 存储 | 理解本地数据持久化 | JPA/MyBatis |
| 07 Dio 网络 | 理解 HTTP 请求封装 | RestTemplate/Feign |

### 第三阶段：深入业务模块（3-4 天）

| 章节 | 目标 |
|------|------|
| 08 插件系统 | 理解规则解析和数据采集 |
| 09 UI 组件 | 理解主题系统和可复用组件 |
| 10 视频播放 | 理解播放器和弹幕渲染 |
| 11 实战开发 | 独立完成一个新功能 |

## 环境准备

```bash
# 1. 安装 Flutter SDK（3.x）
# 参考 https://docs.flutter.dev/get-started/install

# 2. 验证安装
flutter doctor

# 3. 克隆项目后安装依赖
flutter pub get

# 4. 生成代码（MobX 和 Hive 的代码生成）
dart run build_runner build --delete-conflicting-outputs

# 5. 运行项目（选择你的平台）
flutter run              # 默认设备
flutter run -d windows   # Windows
flutter run -d macos     # macOS
flutter run -d chrome    # Web（部分功能不可用）
```

## 如何使用本教程

1. **对照源码读** — 每章都会引用具体文件路径，请打开对应文件同步阅读
2. **关注 Java 对照** — 每个新概念都会标注对应的 Java/Spring 概念
3. **动手改代码** — 每章末尾有练习，建议在本地分支上实践
4. **不必按顺序** — 如果你对某个模块特别感兴趣，可以跳着读

## 快速参考

### 常用命令

```bash
flutter pub get                                    # 安装依赖（≈ mvn install）
dart run build_runner build --delete-conflicting-outputs  # 代码生成
flutter analyze                                    # 静态分析（≈ checkstyle）
flutter test                                       # 运行测试
flutter run                                        # 启动应用
```

### 核心文件速查

| 你想了解... | 去看这个文件 |
|------------|-------------|
| 应用入口 | `lib/main.dart` |
| 全局路由配置 | `lib/pages/index_module.dart` |
| 本地存储 | `lib/utils/storage.dart` |
| HTTP 客户端 | `lib/request/request.dart` |
| 插件规则解析 | `lib/plugins/plugins.dart` |
| 主题配置 | `lib/bean/settings/theme_provider.dart` |
| 视频播放器 | `lib/pages/player/player_controller.dart` |

---

准备好了？开始第一章：[Dart 语法速通（Java 开发者版）](./01-DART-FOR-JAVA-DEVS.md)
