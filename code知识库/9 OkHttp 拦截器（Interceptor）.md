# OkHttp 拦截器（Interceptor）详解

拦截器（Interceptor）是 OkHttp 最强大的功能之一，可以在请求发送前、响应返回后统一处理逻辑，例如：

- 日志打印
    
- Token添加
    
- 请求参数统一处理
    
- 网络缓存
    
- 加密解密
    
- 统一错误处理
    
- 接口签名
    
- 性能监控
    

---

# 一、拦截器执行流程

```text
Request
   ↓
Interceptor1
   ↓
Interceptor2
   ↓
Interceptor3
   ↓
Server
   ↑
Interceptor3
   ↑
Interceptor2
   ↑
Interceptor1
   ↑
Response
```

类似责任链模式（Chain of Responsibility）。

---

# 二、Interceptor接口

源码：

```java
public interface Interceptor {

    Response intercept(Chain chain)
            throws IOException;

}
```

参数：

```java
Chain chain
```

包含：

```java
Request request = chain.request();
```

继续请求：

```java
Response response =
        chain.proceed(request);
```

---

# 三、自定义拦截器

## 添加请求头Token

```java
public class TokenInterceptor
        implements Interceptor {

    @Override
    public Response intercept(
            Chain chain)
            throws IOException {

        Request original =
                chain.request();

        Request request =
                original.newBuilder()
                        .header(
                            "token",
                            "123456")
                        .build();

        return chain.proceed(request);
    }
}
```

添加：

```java
OkHttpClient client =
        new OkHttpClient.Builder()
                .addInterceptor(
                        new TokenInterceptor())
                .build();
```

---

# 四、打印日志拦截器

## 自定义实现

```java
public class LogInterceptor
        implements Interceptor {

    @Override
    public Response intercept(
            Chain chain)
            throws IOException {

        Request request =
                chain.request();

        long start =
                System.currentTimeMillis();

        Log.d("OkHttp",
                "url="
                        + request.url());

        Response response =
                chain.proceed(request);

        long end =
                System.currentTimeMillis();

        Log.d("OkHttp",
                "耗时="
                        + (end - start));

        return response;
    }
}
```

---

## 官方日志拦截器

依赖：

```gradle
implementation
'com.squareup.okhttp3:logging-interceptor:4.12.0'
```

配置：

```java
HttpLoggingInterceptor log =
        new HttpLoggingInterceptor();

log.setLevel(
        HttpLoggingInterceptor
                .Level.BODY);

OkHttpClient client =
        new OkHttpClient.Builder()
                .addInterceptor(log)
                .build();
```

日志级别：

```java
NONE
```

不打印

```java
BASIC
```

请求行

```java
HEADERS
```

请求头

```java
BODY
```

全部内容

---

# 五、统一参数拦截器

例如每个请求都添加：

```text
appVersion
deviceId
channel
```

```java
public class CommonParamsInterceptor
        implements Interceptor {

    @Override
    public Response intercept(
            Chain chain)
            throws IOException {

        Request oldRequest =
                chain.request();

        HttpUrl url =
                oldRequest.url()
                        .newBuilder()
                        .addQueryParameter(
                                "version",
                                "1.0")
                        .addQueryParameter(
                                "deviceId",
                                "123")
                        .build();

        Request request =
                oldRequest.newBuilder()
                        .url(url)
                        .build();

        return chain.proceed(request);
    }
}
```

---

# 六、统一错误处理

```java
public class ErrorInterceptor
        implements Interceptor {

    @Override
    public Response intercept(
            Chain chain)
            throws IOException {

        Response response =
                chain.proceed(
                        chain.request());

        if (response.code() == 401) {

            // token失效

            EventBus.getDefault()
                    .post(
                        new LoginEvent());
        }

        return response;
    }
}
```

---

# 七、缓存拦截器

离线缓存：

```java
public class CacheInterceptor
        implements Interceptor {

    @Override
    public Response intercept(
            Chain chain)
            throws IOException {

        Request request =
                chain.request();

        if (!NetworkUtil.isConnected()) {

            request =
                    request.newBuilder()
                            .cacheControl(
                               CacheControl
                                    .FORCE_CACHE)
                            .build();
        }

        return chain.proceed(request);
    }
}
```

---

# 八、Application Interceptor 与 Network Interceptor

这是面试高频题。

## Application Interceptor

添加：

```java
addInterceptor()
```

```java
client = new OkHttpClient.Builder()
        .addInterceptor(
                new LogInterceptor())
        .build();
```

特点：

- 最先执行
    
- 只执行一次
    
- 可处理缓存
    
- 看不到重定向过程
    

---

## Network Interceptor

添加：

```java
addNetworkInterceptor()
```

```java
client = new OkHttpClient.Builder()
        .addNetworkInterceptor(
                new LogInterceptor())
        .build();
```

特点：

- 真正网络请求时执行
    
- 缓存命中不会执行
    
- 可以获取网络传输数据
    
- 能看到重定向过程
    

---

## 面试回答

```text
Application Interceptor：

位于应用层，
无论是否命中缓存都会执行，
且只执行一次。

Network Interceptor：

位于网络层，
只有真正访问网络才执行，
缓存命中不会执行，
能够观察重定向和重试过程。
```

---

# 九、拦截器源码分析

执行入口：

```java
RealCall.execute()
```

↓

```java
getResponseWithInterceptorChain()
```

↓

创建责任链：

```java
List<Interceptor> interceptors
```

源码：

```java
interceptors.addAll(
        client.interceptors());

interceptors.add(
        RetryAndFollowUpInterceptor);

interceptors.add(
        BridgeInterceptor);

interceptors.add(
        CacheInterceptor);

interceptors.add(
        ConnectInterceptor);

interceptors.addAll(
        client.networkInterceptors());

interceptors.add(
        CallServerInterceptor);
```

---

# 十、OkHttp内置拦截器

执行顺序：

```text
ApplicationInterceptor
        ↓
RetryAndFollowUpInterceptor
        ↓
BridgeInterceptor
        ↓
CacheInterceptor
        ↓
ConnectInterceptor
        ↓
NetworkInterceptor
        ↓
CallServerInterceptor
```

### 1. RetryAndFollowUpInterceptor

负责：

- 重试机制
    
- 重定向
    
- 301
    
- 302
    

---

### 2. BridgeInterceptor

桥接拦截器

自动添加：

```http
Content-Type
Content-Length
Host
Connection
Cookie
User-Agent
```

---

### 3. CacheInterceptor

缓存处理：

```text
强缓存
协商缓存
```

---

### 4. ConnectInterceptor

负责：

```text
TCP连接
TLS握手
连接池复用
```

---

### 5. CallServerInterceptor

真正发起HTTP请求

```java
socket.write()
```

读取：

```java
socket.read()
```

返回：

```java
Response
```

---

# Android高级工程师面试回答

**Q：OkHttp拦截器原理是什么？**

答：

> OkHttp拦截器采用责任链模式实现。请求会依次经过多个Interceptor处理，每个Interceptor都可以修改Request和Response。内部维护一个RealInterceptorChain，通过chain.proceed()递归调用下一个拦截器。最终由CallServerInterceptor发起网络请求，再按相反顺序返回Response。

**Q：Application Interceptor 和 Network Interceptor 区别？**

答：

> Application Interceptor位于应用层，无论是否命中缓存都会执行一次；Network Interceptor位于网络层，仅在真正访问网络时执行，可以观察重定向、重试及网络传输过程。

**Q：OkHttp内置拦截器有哪些？**

答：

> RetryAndFollowUpInterceptor、BridgeInterceptor、CacheInterceptor、ConnectInterceptor、CallServerInterceptor。分别负责重试重定向、请求头补充、缓存管理、连接建立以及真正发送网络请求。

如果你准备 Android 高级工程师面试，我还应该掌握 **OkHttp 整体架构（Dispatcher、连接池、线程池、HTTP/2、多路复用、缓存机制、源码流程）**，这些经常和拦截器一起组成一套完整的源码面试题。