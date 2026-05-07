# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

Kazumi 是一个基于 Flutter 开发的跨平台番剧采集与在线观看程序，支持弹幕显示。使用最多五行基于 `Xpath` 语法的选择器构建自定义规则，支持规则导入与分享。

**平台支持**: Android 10+, Windows 10+, macOS 10.15+, Linux (实验性), iOS 13+ (自签名), HarmonyOS 5.0+

## 常用命令

```bash
# 安装依赖
flutter pub get

# 代码分析
flutter analyze

# 运行测试
flutter test
flutter test test/m3u8_parser_test.dart  # 运行单个测试文件

# 生成 MobX/Hive 代码（修改 @observable/@action 或 Hive TypeAdapter 后需执行）
dart run build_runner build --delete-conflicting-outputs

# 构建 Android APK
flutter build apk --debug
flutter build apk --release
flutter build apk --split-per-abi  # 分架构构建

# 构建 Windows
flutter build windows
dart run msix:create  # Windows MSIX 打包

# 构建 macOS
flutter build macos

# 构建 Linux
flutter build linux

# 构建 iOS (无签名)
flutter build ios --no-codesign
```

## 架构概览

### 技术栈
- **状态管理**: MobX (`@observable`/`@action` + `.g.dart` 代码生成)
- **路由/DI**: flutter_modular (模块化路由与依赖注入)
- **主题**: Provider (`ChangeNotifierProvider<ThemeProvider>`)
- **持久化**: Hive CE (`GStorage` 单例封装)
- **网络**: Dio + CookieJar + 自定义拦截器
- **视频播放**: media_kit (forked) + Anime4K 着色器超分
- **弹幕渲染**: canvas_danmaku

### 核心目录结构
```
lib/
├── main.dart                 # 应用入口，Hive/媒体库/窗口初始化
├── app_module.dart           # 根 Modular 模块配置
├── app_widget.dart           # MaterialApp 根组件，主题配置
├── pages/                    # UI 页面（每个页面含 *_controller.dart, *_module.dart, *_page.dart）
├── modules/                  # 数据模型（含 Hive TypeAdapter 的 .g.dart 文件）
├── bean/                     # 可复用 UI 组件（appbar, cards, dialogs, settings, widgets）
├── request/                  # HTTP 层（Dio 封装、拦截器、API 调用）
├── repositories/             # 数据仓库（collect, history, download, search_history）
├── utils/                    # 工具类（storage, proxy, m3u8, syncplay, webdav 等）
├── plugins/                  # 插件系统（Xpath 选择器解析与规则加载）
├── bbcode/                   # BBCode 解析器（ANTLR4 生成）
└── shaders/                  # Anime4K GPU 着色器
```

### 页面模块模式
每个页面遵循统一结构：
- `*_controller.dart` - MobX Store（状态管理），使用 `@observable`/`@action`
- `*_module.dart` - Flutter Modular Module（路由配置与依赖绑定）
- `*_page.dart` - Page 组件（使用 `ConsumerWidget` 或 `Observer`）

### 数据流
```
Page → Controller (MobX Store) → Repository → Request (Dio) → 外部 API
                ↓
            Hive (GStorage) 本地缓存
```

## 关键文件

- `lib/main.dart` - 应用入口，Hive 初始化、窗口管理、平台特定设置
- `lib/app_module.dart` - 根模块，绑定全局 Provider 和 Repository
- `lib/pages/index_module.dart` - 主页面路由与所有 Controller 绑定
- `lib/pages/router.dart` - 主导航路由定义（popular, timeline, collect, my）
- `lib/utils/storage.dart` - Hive 存储封装，GStorage 单例，所有 setting key 定义
- `lib/request/request.dart` - Dio 实例，代理配置
- `lib/request/api.dart` - 外部 API 调用（Bangumi、弹幕等）
- `lib/plugins/plugins.dart` - Plugin 类定义，Xpath 选择器解析
- `assets/plugins/*.json` - 内置插件规则配置文件

## 代码生成

以下文件由代码生成器产生，修改源文件后需重新生成：

```bash
dart run build_runner build --delete-conflicting-outputs
```

- `lib/hive_registrar.g.dart` - Hive 类型注册（修改 modules/ 下的 Hive 类型后需重新生成）
- `lib/modules/**/*.g.dart` - MobX Store 代码
- `lib/pages/**/*_controller.g.dart` - Controller 响应式代码
- `lib/bbcode/*.dart` - ANTLR4 生成的 BBCode 解析器

## 平台特定注意事项

- **Android/iOS**: 使用透明状态栏，需调用 `SystemChrome.setEnabledSystemUIMode`
- **Windows**: 窗口初始化有防闪烁处理（见 `main.dart`），支持 MSIX 打包
- **macOS**: 需要 entitlements 配置网络和文件访问权限
- **HarmonyOS**: 特殊适配，参考 `ohos/` 目录

## CI/CD

- `.github/workflows/pr.yaml` - PR 构建（5 平台，路径过滤，跳过 draft）
- `.github/workflows/release.yaml` - Tag 触发，构建所有平台并发布 Release
  - Windows: SignPath 签名
  - Android: APK 签名
  - Secrets: DanDanPlay API 凭证注入到 `lib/utils/mortis.dart`

## 插件系统

- 插件规则仅支持 `//` 开头的 Xpath 选择器
- 内置插件位于 `assets/plugins/*.json`
- 自定义规则可通过导入功能添加，存储在 Hive 中
- 贡献插件规则请前往 [KazumiRules](https://github.com/Predidit/KazumiRules)

## 重要依赖说明

| 依赖 | 用途 |
|------|------|
| `media_kit` (forked) | 跨平台视频播放 |
| `canvas_danmaku` | 弹幕渲染 |
| `xpath_selector` | 规则解析（Xpath 语法） |
| `antlr4` | BBCode 语法解析 |
| `webdav_client` | WebDAV 同步 |
| `dlna_dart` | DLNA 投屏 |
| `flutter_foreground_task` | 后台服务 |
| `trace.moe` | 以图搜番 |
