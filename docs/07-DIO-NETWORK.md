# 第七章：Dio 网络请求

> 目标：理解 Dio HTTP 客户端的封装方式，对照 RestTemplate/OkHttp 理解
> 配套源码：`lib/request/request.dart`、`lib/request/api.dart`、`lib/request/bangumi.dart`

---

## 1. Dio 是什么

Dio 是 Dart 生态中最流行的 HTTP 客户端，功能对标 Java 的 OkHttp + Retrofit。

### 对照 Java

| Dio | Java | 说明 |
|-----|------|------|
| `Dio()` | `new OkHttpClient()` | 创建 HTTP 客户端 |
| `dio.get()` | `restTemplate.getForObject()` | GET 请求 |
| `dio.post()` | `restTemplate.postForObject()` | POST 请求 |
| `Interceptor` | `OkHttp Interceptor` | 请求/响应拦截 |
| `Options` | `RequestConfig` | 请求配置 |
| `CookieJar` | `CookieManager` | Cookie 管理 |

---

## 2. Request 单例

`lib/request/request.dart` 封装了全局 Dio 实例：

```dart
class Request {
  static final Request _instance = Request._internal();
  static late final Dio dio;

  factory Request() {
    return _instance;
  }

  Request._internal() {
    dio = Dio(BaseOptions(
      connectTimeout: Duration(seconds: 12),
      receiveTimeout: Duration(seconds: 12),
    ));
    // 添加拦截器
    dio.interceptors.add(LogInterceptor());
  }
}
```

对照 Spring Boot 的 RestTemplate 配置：
```java
@Configuration
public class RestTemplateConfig {
    @Bean
    public RestTemplate restTemplate() {
        var factory = new SimpleClientHttpRequestFactory();
        factory.setConnectTimeout(12000);
        factory.setReadTimeout(12000);
        return new RestTemplate(factory);
    }
}
```

### 代理支持

```dart
// 设置 HTTP 代理（用于调试或翻墙）
static void setProxy(String proxyUrl) {
  dio.httpClientAdapter = IOHttpClientAdapter(
    createHttpClient: () {
      final client = HttpClient();
      client.findProxy = (uri) => 'PROXY $proxyUrl';
      return client;
    },
  );
}
```

---

## 3. API 调用示例

### Bangumi API

`lib/request/bangumi.dart` — 封装 Bangumi 网站的 API 调用：

```dart
class BangumiHTTP {
  // 获取热门番剧列表
  static Future<List<BangumiItem>> getBangumiTrendsList({int offset = 0}) async {
    try {
      var response = await Request.dio.get(
        '${Api.bangumiRankUrl}?limit=24&offset=$offset&type=2',
      );
      // 解析 JSON 响应
      List<BangumiItem> items = [];
      for (var json in response.data['data']) {
        items.add(BangumiItem.fromJson(json));
      }
      return items;
    } catch (e) {
      KazumiLogger().e('Failed to fetch trends', error: e);
      return [];
    }
  }

  // 按标签搜索
  static Future<List<BangumiItem>> getBangumiList({
    required int rank,
    required String tag,
  }) async {
    var response = await Request.dio.get(
      '${Api.bangumiSearchUrl}?limit=24&offset=$rank&filter[tag]=$tag',
    );
    // ...
  }
}
```

对照 Spring 的 Feign Client：
```java
@FeignClient(name = "bangumi", url = "${api.bangumi.url}")
public interface BangumiClient {
    @GetMapping("/rank?limit=24&offset={offset}&type=2")
    BangumiResponse getTrendsList(@PathVariable int offset);
}
```

### API 常量

`lib/request/api.dart` — 集中管理所有 API 地址：

```dart
class Api {
  static const String version = '2.1.0';
  static const int apiLevel = 6;

  // Bangumi API
  static const String bangumiRankUrl = 'https://api.bgm.tv/v0/subjects';
  static const String bangumiSearchUrl = 'https://api.bgm.tv/v0/subjects';

  // 弹幕 API
  static const String dandanSearchUrl = 'https://api.dandanplay.net/api/v2/search/episodes';

  // 更新检查
  static const String latestReleaseUrl = 'https://api.github.com/repos/Predidit/Kazumi/releases/latest';
}
```

---

## 4. 拦截器

### 请求拦截器（≈ Spring HandlerInterceptor）

```dart
class CustomInterceptor extends Interceptor {
  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) {
    // 在每个请求发出前执行（≈ preHandle）
    options.headers['User-Agent'] = 'Kazumi/$version';
    handler.next(options);  // 继续执行
  }

  @override
  void onResponse(Response response, ResponseInterceptorHandler handler) {
    // 在收到响应后执行（≈ postHandle）
    handler.next(response);
  }

  @override
  void onError(DioException err, ErrorInterceptorHandler handler) {
    // 在请求出错时执行（≈ @ExceptionHandler）
    KazumiLogger().e('Request failed: ${err.message}');
    handler.next(err);
  }
}
```

### Cookie 管理

```dart
static Future<void> setCookie() async {
  final cookieJar = CookieJar();
  dio.interceptors.add(CookieManager(cookieJar));
}
```

---

## 5. 错误处理

### DioException 类型

```dart
try {
  var response = await Request.dio.get(url);
} on DioException catch (e) {
  switch (e.type) {
    case DioExceptionType.connectionTimeout:
      // 连接超时
      break;
    case DioExceptionType.receiveTimeout:
      // 接收超时
      break;
    case DioExceptionType.badResponse:
      // 服务器返回错误状态码
      print('Status: ${e.response?.statusCode}');
      break;
    case DioExceptionType.cancel:
      // 请求被取消
      break;
    default:
      // 其他错误（网络不可用等）
      break;
  }
}
```

### 在 Controller 中的错误处理模式

```dart
Future<void> queryBangumiByTrend({String type = 'add'}) async {
  isLoadingMore = true;
  try {
    var result = await BangumiHTTP.getBangumiTrendsList(offset: trendList.length);
    trendList.addAll(result);
  } catch (e) {
    KazumiLogger().e('Query failed', error: e);
    isTimeOut = true;  // 通知 UI 显示错误状态
  }
  isLoadingMore = false;
}
```

---

## 6. 常用请求模式

### GET 请求

```dart
// 简单 GET
var response = await Request.dio.get('https://api.example.com/data');
var data = response.data;  // 自动解析 JSON

// 带查询参数
var response = await Request.dio.get(
  'https://api.example.com/search',
  queryParameters: {'keyword': '进击的巨人', 'page': 1},
);
```

### POST 请求

```dart
// JSON body
var response = await Request.dio.post(
  'https://api.example.com/login',
  data: {'username': 'user', 'password': 'pass'},
);

// Form data
var response = await Request.dio.post(
  'https://api.example.com/upload',
  data: FormData.fromMap({'file': await MultipartFile.fromFile(path)}),
);
```

### 自定义 Headers

```dart
var response = await Request.dio.get(
  url,
  options: Options(
    headers: {
      'Authorization': 'Bearer $token',
      'Referer': 'https://example.com',
    },
  ),
);
```

### 请求取消

```dart
final cancelToken = CancelToken();

// 发起请求
Request.dio.get(url, cancelToken: cancelToken);

// 取消请求（比如用户离开页面）
cancelToken.cancel('User navigated away');
```

---

## 7. 弹幕 API 调用

`lib/request/damaku.dart` — 弹幕数据获取：

```dart
class DanmakuHTTP {
  static Future<List<Danmaku>> getDanmaku(int episodeId) async {
    var response = await Request.dio.get(
      '${Api.dandanCommentUrl}/$episodeId',
      queryParameters: {'withRelated': true},
    );

    List<Danmaku> danmakuList = [];
    for (var item in response.data['comments']) {
      danmakuList.add(Danmaku.fromJson(item));
    }
    return danmakuList;
  }
}
```

---

## 8. 对照总结

| 场景 | Java (Spring) | Dart (Dio) |
|------|---------------|------------|
| 创建客户端 | `new RestTemplate()` | `Dio()` |
| GET 请求 | `restTemplate.getForObject(url, Type.class)` | `dio.get(url)` |
| POST 请求 | `restTemplate.postForObject(url, body, Type.class)` | `dio.post(url, data: body)` |
| 设置超时 | `factory.setConnectTimeout(ms)` | `BaseOptions(connectTimeout: Duration(...))` |
| 拦截器 | `implements HandlerInterceptor` | `extends Interceptor` |
| 错误处理 | `@ExceptionHandler` | `try/catch DioException` |
| Cookie | `CookieManager` | `CookieManager(CookieJar())` |
| 代理 | `System.setProperty("http.proxyHost", ...)` | `client.findProxy = (uri) => ...` |

---

## 练习

1. 打开 `lib/request/bangumi.dart`，找出 `getBangumiTrendsList` 方法请求了哪个 URL
2. 打开 `lib/request/request.dart`，找出 Dio 的超时配置是多少秒
3. 如果你要添加一个新的 API 调用（比如获取番剧评分），需要：
   - 在哪里定义 URL 常量？
   - 在哪里写请求方法？
   - Controller 如何调用？

<details>
<summary>答案</summary>

1. 请求 `Api.bangumiRankUrl`（即 `https://api.bgm.tv/v0/subjects`），带 limit、offset、type 参数
2. connectTimeout 和 receiveTimeout 都是 12 秒
3. URL 常量加在 `lib/request/api.dart`，请求方法写在 `lib/request/bangumi.dart`（或新建文件），Controller 中 `await BangumiHTTP.getScore(id)` 调用

</details>

---

下一章：[插件系统与 XPath 解析](./08-PLUGIN-SYSTEM.md)
