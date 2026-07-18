# OkHttp、Retrofit、RxJava 原理及组合使用详解

这是 Android 面试中高频考点，尤其是高级工程师岗位，经常会追问源码和设计思想。

---

# 一、三者关系图

```text
┌─────────────────────┐
│      RxJava         │
│   线程切换、异步流   │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│     Retrofit        │
│ 网络请求封装框架     │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│      OkHttp         │
│ 真正执行HTTP请求     │
└─────────────────────┘
```

执行流程：

```text
APP
 ↓
RxJava
 ↓
Retrofit
 ↓
OkHttp
 ↓
服务器
```

---

# 二、OkHttp原理

## 什么是OkHttp

Google推荐网络框架。

作用：

- HTTP请求
    
- HTTPS请求
    
- 文件上传
    
- 文件下载
    
- WebSocket
    

本质：

```java
Socket + HTTP协议封装
```

---

## OkHttp核心组件

### 1 Request

请求对象

```java
Request request = new Request.Builder()
        .url(url)
        .build();
```

包含：

```text
url
header
body
method
```

---

### 2 Call

代表一次请求

```java
Call call = client.newCall(request);
```

执行：

```java
call.execute(); //同步

call.enqueue(); //异步
```

---

### 3 Dispatcher

任务调度器

源码：

```java
public final class Dispatcher
```

维护：

```java
Ready Queue
Running Queue
```

默认：

```java
最大请求数 = 64

单个Host最大请求数 = 5
```

---

### 4 ConnectionPool

连接池

源码：

```java
ConnectionPool
```

作用：

TCP复用

避免：

```text
三次握手
四次挥手
```

提升性能。

默认：

```java
5个连接

5分钟回收
```

---

### 5 Interceptor

拦截器

设计模式：

```text
责任链模式
```

源码：

```java
Interceptor
```

系统内置：

```text
RetryAndFollowUpInterceptor

BridgeInterceptor

CacheInterceptor

ConnectInterceptor

CallServerInterceptor
```

执行流程：

```text
请求
 ↓
RetryAndFollowUp
 ↓
Bridge
 ↓
Cache
 ↓
Connect
 ↓
CallServer
 ↓
服务器
```

---

## OkHttp源码流程

### 发起请求

```java
client.newCall(request)
```

创建：

```java
RealCall
```

---

### execute()

同步请求

```java
RealCall.execute()
```

↓

```java
getResponseWithInterceptorChain()
```

↓

责任链执行

```java
RealInterceptorChain
```

↓

Socket通信

↓

返回Response

---

### enqueue()

异步请求

```java
call.enqueue()
```

↓

```java
Dispatcher
```

线程池执行

↓

回调主线程

---

# 三、Retrofit原理

Retrofit本质：

```text
对OkHttp的封装
```

特点：

```text
注解驱动
动态代理
自动解析JSON
```

---

# Retrofit使用

## 定义接口

```java
public interface ApiService {

    @GET("user/info")
    Call<User> getUser();
}
```

---

## 创建Retrofit

```java
Retrofit retrofit =
        new Retrofit.Builder()
                .baseUrl(BASE_URL)
                .addConverterFactory(
                    GsonConverterFactory.create())
                .build();
```

---

## 创建接口实现

```java
ApiService service =
        retrofit.create(ApiService.class);
```

这里最重要。

实际上：

```java
Proxy.newProxyInstance()
```

动态代理生成实现类。

---

## 调用

```java
service.getUser();
```

实际上进入：

```java
InvocationHandler.invoke()
```

---

# Retrofit源码流程

## 第一步

```java
retrofit.create(ApiService.class)
```

↓

动态代理

```java
Proxy.newProxyInstance()
```

---

## 第二步

解析注解

```java
@GET
@POST
@Path
@Query
```

生成：

```java
ServiceMethod
```

---

## 第三步

构造Request

```java
RequestFactory
```

↓

```java
OkHttp Request
```

---

## 第四步

调用OkHttp

```java
Call.Factory
```

默认：

```java
OkHttpClient
```

---

## 第五步

返回结果

```java
ResponseBody
```

↓

```java
GsonConverterFactory
```

↓

实体对象

```java
User
```

---

# 四、RxJava原理

作用：

```text
异步
线程切换
事件流
链式调用
```

---

## 核心思想

观察者模式

```text
Observable
     ↓
Observer
```

---

## 创建

```java
Observable.just("Hello")
```

---

## 订阅

```java
observable.subscribe(observer);
```

---

## 线程切换

```java
.subscribeOn(Schedulers.io())

.observeOn(AndroidSchedulers.mainThread())
```

---

## 原理

### subscribe()

```java
Observable.subscribe()
```

↓

```java
subscribeActual()
```

↓

Observer注册

↓

发送事件

---

## Scheduler原理

IO线程池：

```java
Schedulers.io()
```

源码：

```java
CachedThreadPool
```

---

主线程：

```java
AndroidSchedulers.mainThread()
```

源码：

```java
Handler
```

发送消息

```java
Handler.post()
```

切换到UI线程

---

# 五、Retrofit + RxJava组合

## 添加依赖

```gradle
implementation 'com.squareup.retrofit2:retrofit:2.9.0'

implementation 'com.squareup.retrofit2:converter-gson:2.9.0'

implementation 'com.squareup.retrofit2:adapter-rxjava3:2.9.0'

implementation 'io.reactivex.rxjava3:rxjava:3.1.8'

implementation 'io.reactivex.rxjava3:rxandroid:3.0.2'
```

---

## Api接口

```java
public interface ApiService {

    @GET("user/info")
    Observable<User> getUser();
}
```

---

## Retrofit配置

```java
Retrofit retrofit =
        new Retrofit.Builder()
                .baseUrl(BASE_URL)
                .client(okHttpClient)
                .addConverterFactory(
                    GsonConverterFactory.create())
                .addCallAdapterFactory(
                    RxJava3CallAdapterFactory.create())
                .build();
```

关键：

```java
RxJava3CallAdapterFactory
```

把：

```java
Call<User>
```

转换为：

```java
Observable<User>
```

---

## 调用

```java
service.getUser()
        .subscribeOn(Schedulers.io())
        .observeOn(AndroidSchedulers.mainThread())
        .subscribe(
                user -> {
                    tv.setText(user.getName());
                },
                throwable -> {
                }
        );
```

---

# 六、完整执行流程（面试重点）

```text
点击按钮
    ↓
RxJava订阅
    ↓
Retrofit动态代理
    ↓
解析GET注解
    ↓
生成Request
    ↓
OkHttp创建Call
    ↓
Dispatcher调度
    ↓
Interceptor责任链
    ↓
Socket通信
    ↓
服务器返回JSON
    ↓
Gson解析
    ↓
RxJava发送事件
    ↓
observeOn(MainThread)
    ↓
更新UI
```

---

# 七、高级面试追问（源码级）

### 1.Retrofit为什么要用动态代理？

因为只需要定义接口：

```java
@GET("user")
Observable<User> getUser();
```

运行时自动生成实现类，减少模板代码。

核心：

```java
Proxy.newProxyInstance()
```

---

### 2.Retrofit和OkHttp关系？

```text
Retrofit负责：
接口封装
注解解析
对象转换

OkHttp负责：
真正网络请求
连接池
缓存
拦截器
```

---

### 3.OkHttp为什么快？

原因：

- 连接池复用
    
- HTTP/2多路复用
    
- Gzip压缩
    
- 缓存机制
    
- Dispatcher线程池
    

---

### 4.RxJava为什么容易内存泄漏？

```java
Observable
持有Observer

Observer持有Activity
```

Activity销毁后仍未解除订阅。

解决：

```java
CompositeDisposable.clear();
```

或者结合生命周期框架：

- RxLifecycle
    
- AutoDispose
    

---

### 5.RxJava中的subscribeOn为什么只有第一次有效？

源码中：

```java
SubscribeOnObserver
```

只包装一次上游。

后续：

```java
subscribeOn()
```

不会再次改变事件源线程。

---

# 面试总结（3分钟回答版）

**OkHttp 是底层网络框架，采用责任链模式实现拦截器，通过 Dispatcher 调度请求、ConnectionPool 复用连接，并最终通过 Socket 完成 HTTP 通信。**

**Retrofit 是对 OkHttp 的封装，利用动态代理解析接口注解，自动构建 Request，并结合 Gson 完成对象转换。**

**RxJava 基于观察者模式，实现异步任务、线程切换和链式调用。**

**三者组合时，Retrofit 负责接口层封装，OkHttp 负责网络请求，RxJava 负责异步和线程管理。整个请求链路是：RxJava → Retrofit → OkHttp → Server → Gson解析 → RxJava回调主线程更新UI。**

这是 Android 高级工程师面试中关于网络框架最经典的一套知识体系。