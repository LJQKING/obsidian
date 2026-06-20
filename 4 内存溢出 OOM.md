# Android 内存溢出（OOM，Out Of Memory）

OOM 是 Android 开发中最常见、也是最严重的问题之一。当应用申请内存时，系统无法再分配足够空间，就会抛出：

```java
java.lang.OutOfMemoryError
```

应用通常会直接崩溃。

---

# 一、为什么会发生 OOM

Android 应用运行在独立进程中，每个进程都有最大内存限制：

|设备类型|单个App内存限制|
|---|---|
|普通手机|128MB~512MB|
|高端手机|512MB~1GB+|
|Android TV|更高|

查看当前限制：

```java
ActivityManager am =
    (ActivityManager)getSystemService(ACTIVITY_SERVICE);

int memoryClass = am.getMemoryClass();

Log.d("OOM", memoryClass + "MB");
```

---

# 二、最常见OOM场景

## 1. Bitmap图片过大

最常见，占80%以上OOM。

例如：

```java
Bitmap bitmap =
    BitmapFactory.decodeResource(
        getResources(),
        R.drawable.big_image);
```

假设：

```text
图片尺寸：
4000 × 3000

ARGB_8888：
4字节/像素
```

占用：

```text
4000 × 3000 × 4

≈ 48MB
```

如果加载多张：

```java
bitmap1
bitmap2
bitmap3
bitmap4
```

很容易OOM。

---

### 解决方案

#### 压缩加载

```java
BitmapFactory.Options options =
        new BitmapFactory.Options();

options.inSampleSize = 4;

Bitmap bitmap =
        BitmapFactory.decodeResource(
                getResources(),
                R.drawable.big_image,
                options);
```

缩小：

```text
4000×3000

↓

1000×750
```

内存减少16倍。

---

#### 使用 Glide

推荐生产环境：

```java
Glide.with(this)
        .load(url)
        .into(imageView);
```

优点：

- 自动压缩
    
- Bitmap复用
    
- 内存缓存
    
- 磁盘缓存
    

---

# 2. 内存泄漏导致OOM

例如：

```java
public class MainActivity
        extends AppCompatActivity {

    private static Context context;

    @Override
    protected void onCreate(
            Bundle savedInstanceState) {

        super.onCreate(savedInstanceState);

        context = this;
    }
}
```

问题：

```text
static
    ↓
持有Activity
    ↓
Activity无法回收
    ↓
越来越多
    ↓
OOM
```

---

### 正确写法

```java
context =
        getApplicationContext();
```

或者：

```java
WeakReference<Activity>
```

---

# 3. 集合无限增长

错误示例：

```java
List<Bitmap> list =
        new ArrayList<>();

while(true){

    Bitmap bitmap = createBitmap();

    list.add(bitmap);
}
```

结果：

```text
Bitmap越来越多

↓

GC回收不了

↓

OOM
```

---

### 解决

及时清理：

```java
list.clear();
```

或者：

```java
bitmap.recycle();
```

（Android 8.0以后一般不需要主动 recycle）

---

# 4. WebView导致OOM

错误：

```java
new WebView(this);
```

大量创建：

```java
WebView1
WebView2
WebView3
...
```

WebView本身非常耗内存。

---

### 释放

```java
webView.loadUrl("about:blank");

webView.clearHistory();

webView.removeAllViews();

webView.destroy();
```

---

# 5. RecyclerView图片缓存过多

例如电商首页：

```text
1000个商品

每个商品
3张图片
```

如果全部缓存：

```text
内存暴涨
```

---

### 解决

使用：

```java
Glide
Picasso
Coil
```

不要自己缓存 Bitmap。

---

# 6. 大对象频繁创建

例如：

```java
while(true){

    byte[] data =
        new byte[20 * 1024 * 1024];
}
```

每次：

```text
20MB
```

频繁触发：

```text
Young GC

↓

Full GC

↓

OOM
```

---

# 三、OOM日志怎么看

典型日志：

```java
java.lang.OutOfMemoryError:
Failed to allocate a 16777216 byte allocation
with 4194304 free bytes
and 10MB until OOM
```

含义：

```text
申请16MB

系统只剩4MB

失败
```

---

# 四、OOM分析工具

## Android Studio Memory Profiler

打开：

```text
View
 └── Tool Windows
      └── Profiler
```

查看：

- Java Heap
    
- Native Heap
    
- Bitmap
    
- GC次数
    

---

## LeakCanary

Android面试必问。

添加：

```gradle
debugImplementation
'com.squareup.leakcanary:
leakcanary-android:2.14'
```

自动检测：

```text
Activity泄漏

Fragment泄漏

Handler泄漏

Context泄漏
```

---

# 五、内存优化方案（面试高频）

## 图片优化

使用：

```java
Glide
```

压缩：

```java
inSampleSize
```

WebP：

```text
PNG → WebP
```

---

## 对象复用

例如：

```java
StringBuilder
```

不要：

```java
new StringBuilder()
```

反复创建。

---

## 使用LruCache

```java
LruCache<String, Bitmap>
```

自动淘汰缓存。

---

## 避免静态引用Activity

错误：

```java
static Activity activity;
```

正确：

```java
WeakReference<Activity>
```

---

# 六、Android 面试回答模板

**面试官：什么是 OOM？**

回答：

> OOM（OutOfMemoryError）是应用申请内存时超过系统为进程分配的最大内存限制而导致的异常。Android 中最常见的原因包括 **大图加载、内存泄漏、集合无限增长、WebView 未释放、缓存过多** 等。我通常会通过 **Memory Profiler、LeakCanary、MAT** 等工具定位问题，通过 **图片压缩、Glide、LruCache、对象复用、及时释放资源** 等方式进行优化。

**面试加分项：**

- 能区分 **Java Heap OOM** 和 **Native Heap OOM**
    
- 熟悉 **GC机制**
    
- 会使用 **LeakCanary**
    
- 会分析 **hprof内存快照**
    
- 了解 **Bitmap 内存计算公式**
    

Bitmap 内存计算：

```text
内存大小 =
宽 × 高 × 每像素字节数

ARGB_8888 = 4字节
RGB_565 = 2字节
```

例如：

4000\times3000\times4=48000000

约等于 **45.8MB**，单张图片就可能占据大量内存。