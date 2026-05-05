# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

Kazumi 是一个基于 Flutter 开发的番剧采集与在线观看程序。使用最多五行基于 `Xpath` 语法的选择器构建自定义规则，支持规则导入与分享。

## 开发环境

- **Flutter SDK**: 3.41.9
- **Dart SDK**: >=3.3.4 <4.0.0
- **平台支持**: Android 10+, Windows 10+, macOS 10.15+, Linux, iOS 13+, HarmonyOS 5.0+

## 常用命令

```bash
# 安装依赖
flutter pub get

# 代码分析
flutter analyze

# 运行测试
flutter test

# 生成 MobX/Hive 代码（修改 .g.dart 文件后需执行）
dart run build_runner build

# 构建 Android APK
flutter build apk --debug
flutter build apk --release

# 构建 Windows
flutter build windows

# 构建 Linux (Flatpak)
flutter build linux
```

## 架构概览

### 状态管理与导航
- **Flutter Modular**: 模块化路由与依赖注入
- **MobX**: 响应式状态管理（Controller 层）
- **Provider**: 主题和全局状态

### 数据层
```
lib/repositories/   → 数据访问接口抽象
lib/utils/storage.dart → Hive 存储封装（GStorage 单例）
lib/modules/         → 数据模型（含 Hive TypeAdapter 的 .g.dart 文件）
```

### HTTP 请求
```
lib/request/
├── request.dart    → Dio 实例封装，代理支持
├── interceptor.dart → 请求/响应拦截器
└── api.dart        → 外部 API 调用（Bangumi、弹幕等）
```

### 插件系统
```
lib/plugins/plugins.dart → Plugin 类定义，Xpath 选择器解析
lib/plugins/plugins_controller.dart → 插件加载与管理
assets/plugins/*.json → 插件配置文件
```

### 核心工具
```
lib/utils/
├── storage.dart     → Hive 初始化与 GStorage 单例
├── download_manager.dart → 下载管理（支持 WebDAV）
├── syncplay.dart    → 一起看功能（SyncPlay 协议）
├── auto_updater.dart → 在线更新
└── m3u8_parser.dart → M3U8 解析与广告过滤
```

### 页面模块结构
```
lib/pages/
├── *_controller.dart      → MobX Store（状态管理）
├── *_module.dart          → Flutter Modular Module（路由配置）
├── *_page.dart            → Page 组件
└── menu/menu.dart         → 底部导航/侧边栏路由
```

## 关键文件

- `lib/main.dart` - 应用入口，初始化 Hive、媒体库、窗口管理
- `lib/app_module.dart` - 根模块配置
- `lib/pages/index_module.dart` - 主页面路由与 Controller 绑定
- `lib/utils/storage.dart` - 存储 key 定义（GStorage.setting）
- `assets/plugins/` - 内置插件 JSON

## 代码生成

部分文件由代码生成器产生，修改后需重新生成：
- `lib/hive_registrar.g.dart` - Hive 类型注册
- `lib/modules/**/*.g.dart` - MobX 代码
- `lib/pages/**/*_controller.g.dart` - Controller 响应式代码

## 注意事项

- Android/iOS 端使用透明状态栏，需调用 `SystemChrome.setEnabledSystemUIMode`
- Windows 窗口初始化有防闪烁处理（见 `main.dart`）
- 插件规则仅支持 `//` 开头的 Xpath 选择器
- 使用 `media-kit` 进行跨平台视频播放，支持 Anime4K 着色器超分