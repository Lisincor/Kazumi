# 第五课：Dio 网络请求

> 📖 学习目标：理解 Dio 框架的使用，掌握网络请求封装，了解拦截器原理
> ⏱️ 预计时间：2-3 小时
> 📁 配套源码：`lib/request/request.dart`、`lib/request/interceptor.dart`、`lib/request/bangumi.dart`

---

## 目录

1. [什么是 Dio？](#1-什么是-dio)
2. [Request 单例](#2-request-单例)
3. [GET/POST 请求](#3-getpost-请求)
4. [拦截器](#4-拦截器)
5. [错误处理](#5-错误处理)
6. [代理支持](#6-代理支持)
7. [API 封装](#7-api-封装)
8. [实际案例分析](#8-实际案例分析)

---

## 1. 什么是 Dio？

### 1.1 类比理解

Dio 就像一个"快递员"：
- **请求**：你把信件（数据）交给快递员
- **拦截器**：快递员在出发前检查、修改信件（添加 header）
- **响应**：快递员带回包裹（服务器返回的数据）
- **错误处理**：快递员报告送信失败的原因

### 1.2 为什么用 Dio？

| 特性 | 说明 |
|------|------|
| 拦截器 | 全局处理请求/响应 |
| 适配器 | 支持不同平台（HTTP/IO） |
| 并发请求 | 同时发多个请求 |
| 表单数据 | 上传文件/表单 |
| 证书处理 | 自定义证书验证 |
| 超时控制 | 灵活设置超时时间 |
| 重试机制 | 自动重试失败请求 |

### 1.3 在本项目中的应用

```
lib/request/
├── request.dart     → Dio 单例封装
├── interceptor.dart → 拦截器（添加 header、处理错误）
├── api.dart        → API 常量（URL 模板）
├── bangumi.dart    → Bangumi API 调用
├── plugin.dart     → 插件 HTTP 调用
└── danmaku.dart    → 弹幕 API 调用
```

---

## 2. Request 单例

📁 源码：`lib/request/request.dart`

### 2.1 单例模式

```dart
class Request {
  // 私有实例
  static final Request _instance = Request._internal();

  // Dio 实例
  static late final Dio dio;

  // 工厂构造函数返回单例
  factory Request() => _instance;

  // 私有构造函数（只执行一次）
  Request._internal() {
    // 初始化 Dio
  }
}

// 使用
var request = Request();  // 始终返回同一个实例
```

### 2.2 Dio 配置

📁 源码：`lib/request/request.dart:81-118`

```dart
Request._internal() {
  BaseOptions options = BaseOptions(
    baseUrl: '',                                    // 基础 URL（这里为空，调用时指定）
    connectTimeout: const Duration(milliseconds: 12000),  // 连接超时 12 秒
    receiveTimeout: const Duration(milliseconds: 12000),  // 响应超时 12 秒
    headers: {},
  );

  dio = Dio(options);

  // 添加拦截器
  dio.interceptors.add(ApiInterceptor());

  // 日志拦截器（开发时查看请求日志）
  dio.interceptors.add(LogInterceptor(
    request: false,
    requestHeader: false,
    responseHeader: false,
  ));

  // 自定义 Transformer
  dio.transformer = BackgroundTransformer();

  // 只接受 2xx 状态码
  dio.options.validateStatus = (int? status) {
    return status! >= 200 && status < 300;
  };
}
```

### 2.3 配置参数详解

```dart
BaseOptions(
  baseUrl: 'https://api.example.com',  // 基础 URL
  connectTimeout: 12000,               // 连接超时（毫秒）
  receiveTimeout: 12000,               // 接收超时（毫秒）
  sendTimeout: 12000,                  // 发送超时（毫秒）
  headers: {},                         // 全局请求头
  contentType: 'application/json',     // 内容类型
  responseType: ResponseType.json,    // 响应类型
)

// Options 优先级更高
// RequestOptions 优先级最高
```

---

## 3. GET/POST 请求

### 3.1 GET 请求

📁 源码：`lib/request/request.dart:149-178`

```dart
Future<Response> get(
  url, {
  data,           // 查询参数（会拼接在 URL 后）
  options,        // Options 配置
  cancelToken,    // 取消标记
  extra,          // 额外参数（会传到拦截器）
  shouldRethrow,  // 是否抛出错误
}) async {
  try {
    response = await dio.get(
      url,
      queryParameters: data,  // 查询参数
      options: options,
      cancelToken: cancelToken,
    );
    return response;
  } on DioException catch (e) {
    if (shouldRethrow) {
      rethrow;
    }
    // 返回模拟的错误响应
    return Response(
      data: {'message': await ApiInterceptor.dioError(e)},
      statusCode: 200,
      requestOptions: RequestOptions(),
    );
  }
}
```

使用示例：

```dart
// 简单 GET
var res = await Request().get('https://api.bgm.tv/v0/subjects/1');

// 带查询参数
var res = await Request().get(
  'https://api.bgm.tv/v0/search/subjects',
  data: {'keyword': 'hello', 'limit': 20},
);

// 取消请求
var cancelToken = CancelToken();
Request().get(url, cancelToken: cancelToken);

// 取消
cancelToken.cancel();
```

### 3.2 POST 请求

📁 源码：`lib/request/request.dart:181-213`

```dart
Future<Response> post(
  url, {
  data,              // 请求体数据
  queryParameters,  // URL 查询参数
  options,           // Options 配置
  cancelToken,       // 取消标记
  extra,             // 额外参数
  shouldRethrow,     // 是否抛出错误
}) async {
  try {
    response = await dio.post(
      url,
      data: data,              // 请求体
      queryParameters: queryParameters,  // 查询参数
      options: options,
      cancelToken: cancelToken,
    );
    return response;
  } on DioException catch (e) {
    // 错误处理同 GET
  }
}
```

使用示例：

```dart
// JSON body
var res = await Request().post(
  'https://api.bgm.tv/v0/users/-/collections/1',
  data: {'type': 1},  // Dart 对象会转为 JSON
);

// Form Data
var res = await Request().post(
  'https://api.example.com/submit',
  data: FormData.fromMap({
    'name': 'hello',
    'age': 20,
  }),
);

// 带查询参数
var res = await Request().post(
  'https://api.example.com/search',
  data: {'keyword': 'test'},
  queryParameters: {'filter': 'active'},
);
```

### 3.3 extra 参数传递

📁 源码：`lib/request/request.dart:121-147`

```dart
void _applyExtraOptions(Options options, Map? extra) {
  if (extra == null) return;

  // 自定义 User-Agent
  if (extra['ua'] != null) {
    options.headers = {
      ...?options.headers,
      'user-agent': headerUa(type: extra['ua']),
    };
  }

  // 自定义错误消息
  if (extra['customError'] != null) {
    options.extra = {
      ...?options.extra,
      'customError': extra['customError'],
    };
  }

  // Bangumi 认证
  if (extra['requiresBangumiAuth'] != null) {
    options.extra = {
      ...?options.extra,
      'requiresBangumiAuth': true,
    };
  }

  // 响应类型
  if (extra['resType'] != null) {
    options.responseType = extra['resType'];
  }
}
```

使用示例：

```dart
// 发送请求并自定义错误消息（不显示默认错误）
await Request().get(
  url,
  extra: {'customError': ''},  // 空字符串表示不显示错误
);

// 请求 HTML（而非 JSON）
await Request().get(
  url,
  extra: {'resType': ResponseType.plain},
);

// 请求 Bangumi 需要认证的接口
await Request().get(
  url,
  extra: {'requiresBangumiAuth': true},
  shouldRethrow: true,  // 抛出错误而非返回模拟响应
);
```

---

## 4. 拦截器

### 4.1 拦截器是什么？

拦截器就像"流水线上的质检员"：

```
请求流程：
用户代码 → onRequest → Dio 发送 → 服务器响应 → onResponse → 用户代码

错误流程：
用户代码 → onRequest → Dio 发送 → 服务器错误/网络错误 → onError → 用户代码
```

### 4.2 ApiInterceptor 实现

📁 源码：`lib/request/interceptor.dart:11-54`

```dart
class ApiInterceptor extends Interceptor {
  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) {
    // 1. GitHub 代理
    if (options.path.contains('github')) {
      bool enableGitProxy = setting.get(SettingBoxKey.enableGitProxy, defaultValue: false);
      if (enableGitProxy) {
        options.path = Api.gitMirror + options.path;  // 替换为镜像地址
      }
    }

    // 2. 弹弹 API 添加签名
    if (options.path.contains(Api.dandanAPIDomain)) {
      var timestamp = DateTime.now().millisecondsSinceEpoch ~/ 1000;
      options.headers = {
        'user-agent': Utils.getRandomUA(),
        'referer': '',
        'X-Auth': 1,
        'X-AppId': mortis['id'],
        'X-Timestamp': timestamp,
        'X-Signature': Utils.generateDandanSignature(...),
      };
    }

    // 3. Bangumi API 添加认证
    if (options.path.contains(Api.bangumiAPIDomain)) {
      final mergedHeaders = {...options.headers, ...bangumiHTTPHeader};
      final bool bangumiSyncEnable = setting.get(SettingBoxKey.bangumiSyncEnable, defaultValue: false);
      final bool requiresBangumiAuth = options.extra['requiresBangumiAuth'] == true;
      final String token = setting.get(SettingBoxKey.bangumiAccessToken, defaultValue: '').toString().trim();

      if ((bangumiSyncEnable || requiresBangumiAuth) && token.isNotEmpty) {
        mergedHeaders['Authorization'] = 'Bearer $token';
      }
      options.headers = mergedHeaders;
    }

    // 继续处理（必须调用）
    handler.next(options);
  }

  @override
  void onResponse(Response response, ResponseInterceptorHandler handler) {
    // 这里可以统一处理响应，比如格式化、缓存等
    handler.next(response);  // 继续传递响应
  }

  @override
  void onError(DioException err, ErrorInterceptorHandler handler) async {
    // 统一错误处理
    handler.next(err);  // 继续传递错误（会触发用户的 catch）
  }
}
```

### 4.3 拦截器使用场景

```dart
// 1. 添加统一的 header
void onRequest(RequestOptions options, handler) {
  options.headers['Authorization'] = 'Bearer $token';
  options.headers['Accept-Language'] = 'zh-CN';
  handler.next(options);
}

// 2. 日志记录
void onRequest(RequestOptions options, handler) {
  KazumiLogger().i('Request: ${options.method} ${options.uri}');
  handler.next(options);
}

void onResponse(Response response, handler) {
  KazumiLogger().i('Response: ${response.statusCode}');
  handler.next(response);
}

// 3. 统一错误处理
void onError(DioException err, handler) {
  switch (err.type) {
    case DioExceptionType.connectionTimeout:
      // 处理超时
      break;
    // ...
  }
  handler.next(err);  // 继续传递错误
}

// 4. 重试机制
void onError(DioException err, handler) async {
  if (shouldRetry(err)) {
    // 重新发送请求
    final response = await dio.fetch(err.requestOptions);
    handler.resolve(response);
  } else {
    handler.next(err);
  }
}
```

---

## 5. 错误处理

### 5.1 错误类型

📁 源码：`lib/request/interceptor.dart:86-104`

```dart
switch (error.type) {
  case DioExceptionType.badCertificate:
    return '证书有误！';
  case DioExceptionType.badResponse:
    return '服务器异常，请稍后重试！';
  case DioExceptionType.cancel:
    return '请求已被取消，请重新请求';
  case DioExceptionType.connectionError:
    return '连接错误，请检查网络设置';
  case DioExceptionType.connectionTimeout:
    return '网络连接超时，请检查网络设置';
  case DioExceptionType.receiveTimeout:
    return '响应超时，请稍后重试！';
  case DioExceptionType.sendTimeout:
    return '发送请求超时，请检查网络设置';
  case DioExceptionType.unknown:
    // 需要进一步检查网络状态
    return '$res 网络异常';
}
```

### 5.2 错误处理模式

```dart
// 方式 1：直接 try-catch
try {
  var res = await Request().get(url, shouldRethrow: true);
} catch (e) {
  KazumiLogger().e('Request failed', error: e);
}

// 方式 2：检查响应状态
var res = await Request().get(url);
// res.statusCode 可能不是 200
if (res.data['message'] != null) {
  // 请求失败了，message 包含错误信息
  showToast(res.data['message']);
} else {
  // 请求成功，处理数据
}

// 方式 3：shouldRethrow 参数
var res = await Request().get(
  url,
  shouldRethrow: true,  // 抛出真正的异常
);
```

### 5.3 网络状态检测

📁 源码：`lib/request/interceptor.dart:107-128`

```dart
static Future<String> checkConnect() async {
  final connectivityResult = await Connectivity().checkConnectivity();

  if (connectivityResult.contains(ConnectivityResult.mobile)) {
    return '正在使用移动流量';
  }
  if (connectivityResult.contains(ConnectivityResult.wifi)) {
    return '正在使用wifi';
  }
  if (connectivityResult.contains(ConnectivityResult.ethernet)) {
    return '正在使用局域网';
  }
  if (connectivityResult.contains(ConnectivityResult.vpn)) {
    return '正在使用代理网络';
  }
  if (connectivityResult.contains(ConnectivityResult.none)) {
    return '未连接到任何网络';
  }
  return '';
}
```

---

## 6. 代理支持

📁 源码：`lib/request/request.dart:35-68`

### 6.1 设置 HTTP 代理

```dart
static void setProxy() {
  final bool proxyEnable = setting.get(SettingBoxKey.proxyEnable, defaultValue: false);
  if (!proxyEnable) {
    disableProxy();
    return;
  }

  final String proxyUrl = setting.get(SettingBoxKey.proxyUrl, defaultValue: '');
  final parsed = ProxyUtils.parseProxyUrl(proxyUrl);
  if (parsed == null) {
    KazumiLogger().w('Proxy: 代理地址格式错误或为空');
    return;
  }

  final (proxyHost, proxyPort) = parsed;

  dio.httpClientAdapter = IOHttpClientAdapter(
    createHttpClient: () {
      final HttpClient client = HttpClient();
      // 设置代理
      client.findProxy = (Uri uri) {
        return 'PROXY $proxyHost:$proxyPort';
      };
      // 忽略证书验证（开发环境）
      client.badCertificateCallback =
          (X509Certificate cert, String host, int port) => true;
      return client;
    },
  );
}
```

### 6.2 禁用代理

```dart
static void disableProxy() {
  dio.httpClientAdapter = IOHttpClientAdapter(
    createHttpClient: () {
      final HttpClient client = HttpClient();
      // 不设置代理，直接连接
      return client;
    },
  );
  KazumiLogger().i('Proxy: 代理已禁用');
}
```

---

## 7. API 封装

### 7.1 API 常量

📁 源码：`lib/request/api.dart`

```dart
class Api {
  // 基础配置
  static const String version = '2.1.0';
  static const int apiLevel = 6;

  // Bangumi API
  static const String bangumiAPIDomain = 'https://api.bgm.tv';
  static const String bangumiInfoByID = '/v0/subjects/{0}';
  static const String bangumiRankSearch = '/v0/search/subjects?limit={0}&offset={1}';

  // Bangumi Next API
  static const String bangumiAPINextDomain = 'https://next.bgm.tv';
  static const String bangumiCalendar = '/p1/calendar';

  // 弹弹 API
  static const String dandanAPIDomain = 'https://api.dandanplay.net';
  static const String dandanAPIComment = "/api/v2/comment/";

  // URL 格式化
  static String formatUrl(String url, List<dynamic> params) {
    for (int i = 0; i < params.length; i++) {
      url = url.replaceAll('{$i}', params[i].toString());
    }
    return url;
  }
}

// 使用
var url = Api.formatUrl(Api.bangumiInfoByID, [123]);
// 结果: '/v0/subjects/123'
```

### 7.2 HTTP 调用封装

📁 源码：`lib/request/bangumi.dart`

```dart
class BangumiHTTP {
  // 获取番剧列表
  static Future<List<BangumiItem>> getBangumiList({int rank = 2, String tag = ''}) async {
    List<BangumiItem> bangumiList = [];

    var params = <String, dynamic>{
      'keyword': '',
      'sort': 'rank',
      'filter': {
        'type': [2],
        'tag': [tag == '' ? '日本' : tag],
        'rank': [">$rank", "<=1050"],
        'nsfw': false,
      },
    };

    try {
      final res = await Request().post(
        Api.formatUrl(Api.bangumiAPIDomain + Api.bangumiRankSearch, [100, 0]),
        data: params,
      );
      final jsonData = res.data;
      final jsonList = jsonData['data'];

      for (dynamic jsonItem in jsonList) {
        if (jsonItem is Map<String, dynamic>) {
          bangumiList.add(BangumiItem.fromJson(jsonItem));
        }
      }
    } catch (e) {
      KazumiLogger().e('Network: resolve bangumi list failed', error: e);
    }

    return bangumiList;
  }

  // 获取番剧详情
  static Future<BangumiItem?> getBangumiInfoByID(int id) async {
    try {
      final res = await Request().get(
        Api.formatUrl(Api.bangumiAPIDomain + Api.bangumiInfoByID, [id]),
      );
      return BangumiItem.fromJson(res.data);
    } catch (e) {
      KazumiLogger().e('Network: resolve bangumi item failed', error: e);
      return null;
    }
  }

  // 获取趋势列表
  static Future<List<BangumiItem>> getBangumiTrendsList({
    int type = 2,
    int limit = 24,
    int offset = 0,
  }) async {
    var params = <String, dynamic>{
      'type': type,
      'limit': limit,
      'offset': offset,
    };

    try {
      final res = await Request().get(
        Api.bangumiAPINextDomain + Api.bangumiTrendsNext,
        data: params,
      );
      final jsonData = res.data;
      final jsonList = jsonData['data'];

      for (dynamic jsonItem in jsonList) {
        if (jsonItem is Map<String, dynamic>) {
          bangumiList.add(BangumiItem.fromJson(jsonItem['subject']));
        }
      }
    } catch (e) {
      KazumiLogger().e('Network: resolve bangumi trends list failed', error: e);
    }

    return bangumiList;
  }
}
```

### 7.3 调用示例

```dart
// 获取推荐列表
Future<List<BangumiItem>> loadPopularBangumis() async {
  var result = await BangumiHTTP.getBangumiList(rank: 2, tag: '');
  return result;
}

// 获取番剧详情
Future<BangumiItem?> loadBangumiDetail(int id) async {
  var result = await BangumiHTTP.getBangumiInfoByID(id);
  return result;
}

// 搜索
Future<List<BangumiItem>> searchBangumi(String keyword) async {
  var result = await BangumiHTTP.bangumiSearch(keyword);
  return result;
}
```

---

## 8. 实际案例分析

### 案例 1：PopularController 中的网络请求

📁 源码：`lib/pages/popular/popular_controller.dart:39-62`

```dart
@action
Future<void> queryBangumiByTag({String type = 'add'}) async {
  if (type == 'init') {
    bangumiList.clear();
  }

  isLoadingMore = true;

  // 使用 BangumiHTTP 获取数据
  int randomNumber = Random().nextInt(8000) + 1;
  var tag = currentTag;
  var result = await BangumiHTTP.getBangumiList(
    rank: randomNumber,
    tag: tag,
  );

  // 更新状态
  bangumiList.addAll(result);
  isLoadingMore = false;
  isTimeOut = bangumiList.isEmpty;
}
```

### 案例 2：分页加载

📁 源码：`lib/request/bangumi.dart:243-274`

```dart
static Future<List<EpisodeInfo>> getBangumiEpisodesByID(int id) async {
  final List<EpisodeInfo> episodeList = [];
  const int limit = 100;
  int offset = 0;
  int? total;

  try {
    do {
      final params = <String, dynamic>{
        'subject_id': id,
        'offset': offset,
        'limit': limit,
      };

      final res = await Request().get(
        Api.bangumiAPIDomain + Api.bangumiEpisodeByID,
        data: params,
      );

      final jsonData = res.data;
      total ??= jsonData['total'] as int?;
      final data = jsonData['data'] as List<dynamic>? ?? [];

      if (data.isEmpty) break;

      episodeList.addAll(data
          .whereType<Map<String, dynamic>>()
          .map((jsonItem) => EpisodeInfo.fromJson(jsonItem)));

      offset += data.length;
    } while (total == null || offset < total);
  } catch (e) {
    KazumiLogger().e('Network: resolve bangumi episode list failed', error: e);
  }

  return episodeList;
}
```

### 案例 3：带进度的批量请求

📁 源码：`lib/request/bangumi.dart:371-467`

```dart
static Future<List<BangumiCollection>> getBangumiCollectibles({
  List<BangumiCollectionType> includeBangumiTypes = const [...],
  String? username,
  required int limit,
  void Function(String message, int current, int total)? onProgress,
}) async {
  final List<BangumiCollection> bangumiCollection = [];
  // ...

  for (final collectionType in includeBangumiTypes) {
    // ...

    while (true) {
      try {
        final res = await Request().get(
          url,
          extra: {'customError': '', 'requiresBangumiAuth': true},
          shouldRethrow: true,
        );
        // ...
      } catch (e) {
        KazumiLogger().e('BangumiHTTP: fetch collection failed', error: e);
        rethrow;
      }

      // 通知进度
      onProgress?.call(
        '正在拉取${collectionType.label}收藏',
        progressCurrent,
        progressTotal,
      );

      // 避免请求过快
      await Future.delayed(Duration(milliseconds: 250));
    }
  }

  return bangumiCollection;
}
```

---

## 小练习

### 练习 1：添加新 API

假设你需要添加一个新 API `getBangumiByDay(int weekday)` 获取特定星期几的番剧。

<details>
<summary>答案</summary>

```dart
// 在 api.dart 添加
static const String bangumiCalendar = '/p1/calendar';

// 在 bangumi.dart 添加
static Future<List<BangumiItem>> getBangumiByDay(int weekday) async {
  List<BangumiItem> result = [];
  try {
    var res = await Request().get(Api.bangumiAPINextDomain + Api.bangumiCalendar);
    final jsonData = res.data;
    final dayList = jsonData['$weekday'] as List;
    for (var item in dayList) {
      result.add(BangumiItem.fromJson(item['subject']));
    }
  } catch (e) {
    KazumiLogger().e('Failed to get bangumi by day', error: e);
  }
  return result;
}
```

</details>

### 练习 2：添加请求拦截

假设要在所有请求中添加一个 `X-Client-Version` header，应该在哪里修改？

<details>
<summary>答案</summary>

在 `ApiInterceptor.onRequest` 方法中添加：

```dart
@override
void onRequest(RequestOptions options, RequestInterceptorHandler handler) {
  // 添加版本 header
  options.headers['X-Client-Version'] = Api.version;

  // ... 其他代码

  handler.next(options);
}
```

</details>

### 练习 3：实现重试机制

实现一个带重试的请求方法，失败后自动重试 3 次。

<details>
<summary>答案</summary>

```dart
Future<Response> getWithRetry(
  String url, {
  int maxRetries = 3,
  int delayMs = 1000,
}) async {
  int retryCount = 0;

  while (retryCount < maxRetries) {
    try {
      return await Request().get(url, shouldRethrow: true);
    } catch (e) {
      retryCount++;
      if (retryCount >= maxRetries) rethrow;
      KazumiLogger().w('Request failed, retrying ($retryCount/$maxRetries)');
      await Future.delayed(Duration(milliseconds: delayMs * retryCount));
    }
  }
  throw Exception('Max retries exceeded');
}
```

</details>

---

## 下一课预告

下一课我们将学习 [插件系统](./06-PLUGINS.md)，了解如何用 XPath 解析网页获取视频源。