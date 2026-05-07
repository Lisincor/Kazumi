# 第十章：视频播放与弹幕

> 目标：理解视频播放器和弹幕渲染系统的架构
> 配套源码：`lib/pages/player/`、`lib/pages/video/`、`lib/modules/danmaku/`

---

## 1. 视频播放架构

### 整体流程

```
用户选择某一集
      │
      ▼
VideoPageController — 解析视频源
      │
      ├── WebView 方式：加载页面，拦截 .m3u8/.mp4 请求
      │
      └── 直接解析：从 HTML 中提取视频地址
      │
      ▼
获得视频 URL（通常是 m3u8 格式）
      │
      ▼
PlayerController — 控制播放
      │
      ├── media_kit 播放视频
      ├── 加载弹幕数据
      ├── 同步弹幕时间轴
      └── 处理用户交互（暂停、快进、音量等）
```

### 关键文件

| 文件 | 职责 |
|------|------|
| `lib/pages/video/video_controller.dart` | 视频源解析、集数管理 |
| `lib/pages/player/player_controller.dart` | 播放状态、弹幕、进度 |
| `lib/pages/player/player_item.dart` | 播放器 UI 组件 |

---

## 2. VideoPageController — 视频源解析

```dart
abstract class _VideoPageController with Store {
  @observable
  String videoUrl = '';  // 解析出的视频地址

  @observable
  bool isLoading = true;

  @observable
  List<Road> roadList = [];  // 播放线路列表

  // 切换集数
  Future<void> changeEpisode(int episode, {int road = 0}) async {
    isLoading = true;
    // 1. 获取该集的页面 URL
    String pageUrl = roadList[road].data[episode];
    // 2. 通过 WebView 或直接解析获取视频地址
    videoUrl = await _resolveVideoUrl(pageUrl);
    isLoading = false;
  }
}
```

### WebView 视频解析

大部分视频源的播放地址由 JavaScript 动态生成，需要用 WebView 加载页面后拦截网络请求：

```dart
// 简化的 WebView 解析流程
class WebViewVideoSourceProvider {
  Future<String> getVideoUrl(String pageUrl) async {
    // 1. 创建 headless WebView（不显示界面）
    // 2. 加载目标页面
    // 3. 监听所有网络请求
    // 4. 当发现 .m3u8 或 .mp4 请求时，返回该 URL
    // 5. 超时则返回空
  }
}
```

---

## 3. PlayerController — 播放控制

```dart
abstract class _PlayerController with Store {
  // media_kit 播放器实例
  late Player player;
  late VideoController videoController;

  // 播放状态
  @observable
  bool isPlaying = false;

  @observable
  Duration position = Duration.zero;

  @observable
  Duration duration = Duration.zero;

  @observable
  bool isBuffering = false;

  // 弹幕控制器
  DanmakuController? danmakuController;

  @observable
  List<Danmaku> danmakuList = [];

  // 初始化播放器
  void initPlayer() {
    player = Player();
    videoController = VideoController(player);

    // 监听播放状态
    player.stream.playing.listen((playing) {
      isPlaying = playing;
    });

    // 监听进度
    player.stream.position.listen((pos) {
      position = pos;
      // 同步弹幕时间轴
      _syncDanmaku(pos);
    });
  }

  // 播放视频
  Future<void> playVideo(String url) async {
    await player.open(Media(url));
  }

  // 加载弹幕
  Future<void> loadDanmaku(int episodeId) async {
    danmakuList = await DanmakuHTTP.getDanmaku(episodeId);
  }
}
```

---

## 4. 弹幕系统

### 弹幕数据模型

`lib/modules/danmaku/danmaku_module.dart`:

```dart
class Danmaku {
  String message;   // 弹幕文本
  double time;      // 出现时间（秒）
  int type;         // 类型：1=滚动, 4=底部, 5=顶部
  Color color;      // 颜色
  String source;    // 来源：BiliBili/Gamer/DanDan
}
```

### 弹幕来源

弹幕数据从多个来源获取：
- **DanDanPlay API** — 弹幕聚合平台
- **BiliBili** — B 站弹幕
- **Gamer** — 巴哈姆特弹幕

```dart
// 从 DanDanPlay 获取弹幕
static Future<List<Danmaku>> getDanmaku(int episodeId) async {
  var response = await Request.dio.get(
    '${Api.dandanCommentUrl}/$episodeId',
    queryParameters: {'withRelated': true},
  );

  List<Danmaku> list = [];
  for (var item in response.data['comments']) {
    // 解析弹幕格式: "time,type,color"
    var params = item['p'].split(',');
    list.add(Danmaku(
      message: item['m'],
      time: double.parse(params[0]),
      type: int.parse(params[1]),
      color: Color(int.parse(params[2])),
    ));
  }
  return list;
}
```

### 弹幕渲染

使用 `canvas_danmaku` 库在视频上层渲染弹幕：

```dart
// 在播放器 UI 中叠加弹幕层
Stack(
  children: [
    // 视频画面
    Video(controller: videoController),

    // 弹幕层（覆盖在视频上方）
    DanmakuView(
      controller: danmakuController,
      // 弹幕配置
      option: DanmakuOption(
        area: 0.8,          // 显示区域（80%）
        opacity: 0.9,       // 透明度
        fontSize: 16,       // 字体大小
        duration: 8,        // 滚动时长（秒）
      ),
    ),
  ],
)
```

### 弹幕同步

弹幕需要和视频进度同步：

```dart
void _syncDanmaku(Duration position) {
  double currentTime = position.inMilliseconds / 1000.0;

  // 找出当前时间点应该显示的弹幕
  for (var danmaku in danmakuList) {
    if (danmaku.time >= currentTime - 0.1 &&
        danmaku.time <= currentTime + 0.1) {
      // 发送弹幕到渲染层
      danmakuController?.addDanmaku(danmaku);
    }
  }
}
```

---

## 5. 弹幕设置

用户可以自定义弹幕显示效果：

```dart
// 存储在 Hive 中的弹幕设置
class DanmakuSettings {
  double area;          // 显示区域 0-100%
  double opacity;       // 透明度 10-100%
  double fontSize;      // 字体大小 10-48px
  int fontWeight;       // 字体粗细 1-9
  double duration;      // 滚动时长 2-16 秒
  bool showTop;         // 显示顶部弹幕
  bool showBottom;      // 显示底部弹幕
  bool showScroll;      // 显示滚动弹幕
  bool showBorder;      // 显示描边
  bool colorful;        // 彩色弹幕
  bool massiveMode;     // 海量模式（允许重叠）
  bool deduplicate;     // 去重
}
```

---

## 6. 播放器功能

### 硬件加速

```dart
// 支持多种解码器
enum DecoderType {
  auto,       // 自动选择
  hardware,   // 硬件解码（GPU）
  software,   // 软件解码（CPU）
}
```

### Anime4K 超分辨率

`lib/shaders/` 目录包含 Anime4K GPU 着色器，可以实时提升视频画质：

```dart
// 加载着色器
void enableAnime4K() {
  player.setProperty('glsl-shaders', anime4kShaderPath);
}
```

### 外部播放器

支持将视频 URL 发送到外部播放器（VLC、MPV 等）：

```dart
// lib/utils/external_player.dart
class ExternalPlayer {
  static Future<void> openWith(String url, String playerName) async {
    // 调用系统 Intent/URL Scheme 打开外部播放器
  }
}
```

### 画中画（PiP）

Android 支持画中画模式：

```dart
// lib/utils/pip_utils.dart
class PipUtils {
  static Future<void> enterPip() async {
    // 进入画中画模式
  }
}
```

---

## 7. 同步播放（SyncPlay）

`lib/utils/syncplay.dart` — 多人同步观看：

```dart
// 连接到 SyncPlay 服务器
class SyncPlayClient {
  // 同步播放进度
  void syncPosition(Duration position) { ... }

  // 同步播放/暂停状态
  void syncPlayState(bool isPlaying) { ... }

  // 接收其他用户的同步信息
  void onRemoteSync(SyncMessage msg) { ... }
}
```

---

## 8. M3U8 解析

`lib/utils/m3u8_parser.dart` — 解析 HLS 视频流：

```dart
class M3U8Parser {
  // 解析 m3u8 播放列表
  static Future<List<M3U8Segment>> parse(String url) async {
    var content = await Request.dio.get(url);
    // 解析 #EXTINF 标签获取分片信息
    // 处理相对路径
    // 返回视频分片列表
  }
}
```

`lib/utils/m3u8_ad_filter.dart` — 过滤 m3u8 中的广告分片：

```dart
class M3U8AdFilter {
  // 检测并移除广告分片
  // 通常广告分片的时长和域名与正片不同
  static List<M3U8Segment> filter(List<M3U8Segment> segments) { ... }
}
```

---

## 9. 数据流总结

```
VideoPage
  │
  ├── VideoPageController
  │     ├── 管理播放线路 (roadList)
  │     ├── 管理当前集数
  │     └── 解析视频 URL (WebView/直接)
  │
  └── PlayerController
        ├── media_kit Player (视频播放)
        ├── DanmakuController (弹幕渲染)
        ├── 进度同步
        ├── 弹幕加载与时间轴同步
        └── 播放器设置 (硬件加速/着色器/倍速)
```

---

## 练习

1. 追踪从"用户点击某一集"到"视频开始播放"的完整流程
2. 弹幕是如何与视频进度同步的？
3. 为什么大部分视频源需要 WebView 解析而不能直接 HTTP 请求？

<details>
<summary>答案</summary>

1. 点击集数 → VideoPageController.changeEpisode() → 获取页面 URL → WebView 加载并拦截视频请求 → 获得 m3u8 URL → PlayerController.playVideo(url) → media_kit 开始播放
2. PlayerController 监听 player.stream.position，每次进度更新时检查 danmakuList 中时间匹配的弹幕，发送到 DanmakuController 渲染
3. 因为视频地址通常由 JavaScript 动态生成（加密、防盗链），静态 HTML 中不包含真实视频 URL，必须执行 JS 才能获取

</details>

---

下一章：[实战：从零开发一个新功能](./11-PRACTICAL-GUIDE.md)
