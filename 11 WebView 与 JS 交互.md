# 一、WebView 与 JS 交互

Android 与 JS 交互主要有 3 种方式：

## 1. JS 调用 Android（最常用）

### Android

```java
public class JsBridge {

    @JavascriptInterface
    public void showToast(String msg){
        Log.d("JS", msg);
    }
}
```

绑定：

```java
webView.addJavascriptInterface(
        new JsBridge(),
        "Android"
);
```

### JS

```javascript
Android.showToast("Hello Android");
```

---

### 原理

addJavascriptInterface()

↓

WebCore

↓

JavaBridge

↓

JNI

↓

Java对象

---

面试回答：

> Android通过addJavascriptInterface暴露Java对象给JS，  
> JS调用时经过WebCore和JNI最终执行Java方法。

---

# 2. Android 调用 JS

### Android

```java
webView.evaluateJavascript(
    "javascript:test('hello')",
    value -> {
        Log.d("JS", value);
    }
);
```

JS：

```javascript
function test(msg){
    return "receive:" + msg;
}
```

---

### 原理

Chromium IPC

↓

Render Process

↓

V8 Engine

↓

执行 JS

---

# 3. URL 拦截方式

JS：

```javascript
window.location =
"js://webview?msg=hello";
```

Android：

```java
@Override
public boolean shouldOverrideUrlLoading(
        WebView view,
        WebResourceRequest request) {

    Uri uri = request.getUrl();

    if("js".equals(uri.getScheme())){
        return true;
    }

    return false;
}
```

---

# 二、WebView 为什么启动慢

面试必问。

## WebView初始化做了什么

首次打开：

```text
WebView
 ↓
加载 Chromium
 ↓
启动 Render Process
 ↓
初始化 V8
 ↓
加载 Native 库
 ↓
创建 GPU Context
 ↓
创建网络线程
```

涉及：

```text
libwebviewchromium.so
libv8.so
```

几十 MB。

---

因此：

```text
首次打开：
300~1000ms

二次打开：
几十 ms
```

---

# 三、WebView 预加载

大厂常见优化。

## 方案1：Application预初始化

```java
public class App extends Application {

    @Override
    public void onCreate() {
        super.onCreate();

        new Thread(() -> {
            WebView webView =
                    new WebView(this);

            webView.destroy();
        }).start();
    }
}
```

作用：

```text
提前加载 Chromium
提前初始化 V8
提前加载 so
```

---

## 方案2：WebView 池

腾讯、支付宝常用。

### 创建池

```java
public class WebViewPool {

    private static WebView cache;

    public static WebView get(Context context){

        if(cache == null){
            cache = new WebView(context);
        }

        return cache;
    }
}
```

---

页面使用：

```java
WebView webView =
        WebViewPool.get(this);
```

避免：

```text
重复创建
重复加载Chromium
```

---

## 方案3：IdleHandler预热

```java
Looper.myQueue()
      .addIdleHandler(() -> {

          new WebView(context);

          return false;
      });
```

主线程空闲时初始化。

---

# 四、WebView 白屏问题

最常见线上问题。

## 原因1：页面未加载完成

```java
webView.loadUrl(url);
```

立刻显示。

实际上：

```text
DNS
TCP
SSL
HTML
CSS
JS
```

还没完成。

---

解决：

```java
webView.setWebViewClient(
    new WebViewClient(){

        @Override
        public void onPageFinished(
            WebView view,
            String url){

            progressBar.setGone();
        }
    }
);
```

---

## 原因2：JS阻塞

例如：

```javascript
while(true){
}
```

Render线程卡死。

页面白屏。

---

## 原因3：资源加载失败

```text
CDN失败
图片404
JS加载失败
```

导致页面空白。

---

排查：

```chrome
chrome://inspect
```

远程调试。

---

## 原因4：硬件加速问题

```java
webView.setLayerType(
        View.LAYER_TYPE_SOFTWARE,
        null
);
```

或者：

```xml
android:hardwareAccelerated="true"
```

---

## 原因5：缓存污染

```java
webView.clearCache(true);
```

---

# 五、WebView 黑屏问题

黑屏比白屏更难排查。

---

## 原因1：Render Process Crash

Android 8+

```java
@Override
public boolean onRenderProcessGone(
    WebView view,
    RenderProcessGoneDetail detail) {

    return true;
}
```

---

Chromium崩溃：

```text
GPU崩溃
V8崩溃
OOM
```

都会黑屏。

---

## 原因2：GPU问题

```java
android:hardwareAccelerated="true"
```

某些机型：

```text
GPU驱动Bug
```

导致黑屏。

解决：

```java
webView.setLayerType(
    View.LAYER_TYPE_SOFTWARE,
    null
);
```

---

## 原因3：Activity重建

例如：

```text
横竖屏切换
内存回收
```

WebView Surface 丢失。

出现黑屏。

---

解决：

```java
android:configChanges=
"orientation|screenSize"
```

---

## 原因4：SurfaceView冲突

WebView：

```text
TextureView
Surface
GPU
```

与：

```text
Camera
VideoView
GLSurfaceView
```

混用时可能黑屏。

---

# 六、WebView 性能优化（面试标准答案）

## 开启缓存

```java
settings.setDomStorageEnabled(true);

settings.setDatabaseEnabled(true);

settings.setCacheMode(
    WebSettings.LOAD_DEFAULT
);
```

---

## 预加载

```text
Application预热
WebView池
IdleHandler
```

---

## 复用WebView

```java
WebViewPool
```

避免频繁创建销毁。

---

## 按需加载图片

```java
settings.setBlockNetworkImage(true);
```

页面完成：

```java
settings.setBlockNetworkImage(false);
```

---

## 开启硬件加速

```xml
android:hardwareAccelerated="true"
```

---

# 面试总结（3分钟回答）

WebView 与 JS 交互主要通过：

1. addJavascriptInterface（JS 调 Android）
    
2. evaluateJavascript（Android 调 JS）
    
3. URL Scheme 拦截
    

WebView 首次启动慢是因为 Chromium 内核、V8 引擎、Native SO 和 Render 进程初始化导致，通常通过 Application 预热、IdleHandler 和 WebView 池复用优化。

白屏常见原因包括页面未加载完成、资源加载失败、JS 阻塞和缓存问题；黑屏通常是 RenderProcess 崩溃、GPU 渲染异常、Surface 丢失或硬件加速兼容性问题导致。

线上排查一般结合：

```text
chrome://inspect
adb logcat
onRenderProcessGone
ANR Trace
```

进行定位。