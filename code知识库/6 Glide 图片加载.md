这是 Android 高级面试中关于 **Glide 图片加载框架** 经常会被追问的内容。

面试官通常会从：

1. 为什么封装 Glide
    
2. Glide 加载流程
    
3. Glide 三级缓存
    
4. LruCache原理
    
5. DiskLruCache原理
    
6. Bitmap复用
    
7. 生命周期绑定
    
8. Glide源码架构设计
    

一路追到源码。

---

# 一、为什么要二次封装 Glide

很多公司都会封装一个：

```java
ImageLoader.load(url)
        .placeholder(R.drawable.loading)
        .error(R.drawable.error)
        .into(imageView);
```

而不是直接：

```java
Glide.with(context)
        .load(url)
        .into(imageView);
```

目的：

### 统一入口

```java
ImageLoader.load()
```

底层可替换：

```java
Glide
Coil
Picasso
Fresco
```

业务无感知。

---

### 统一默认配置

```java
placeholder
error
transform
circleCrop
```

统一管理。

---

### 统计埋点

记录：

```java
图片加载耗时

缓存命中率

失败率
```

---

### 日志监控

类似：

```java
ImageMonitor.onSuccess()

ImageMonitor.onFail()
```

---

# 二、Glide整体架构

Glide核心架构：

```text
Glide.with()

    ↓

RequestManager

    ↓

RequestBuilder

    ↓

Engine

    ↓

ActiveResources

    ↓

MemoryCache

    ↓

DiskCache

    ↓

Network
```

源码：

```java
public class Glide {

    public RequestManagerRetriever getRequestManagerRetriever() {
        return requestManagerRetriever;
    }
}
```

---

# 三、Glide加载流程

代码：

```java
Glide.with(context)
     .load(url)
     .into(imageView);
```

流程：

```text
with()

↓

RequestManager

↓

RequestBuilder

↓

Engine.load()

↓

缓存查找

↓

网络下载

↓

Decode

↓

显示
```

---

# 四、with()源码

## Glide.with()

源码：

```java
public static RequestManager with(Context context) {
    return getRetriever(context).get(context);
}
```

进入：

```java
RequestManagerRetriever
```

源码：

```java
public RequestManager get(Context context)
```

作用：

创建：

```java
RequestManager
```

---

RequestManager负责：

```java
生命周期绑定

Activity

Fragment

Application
```

---

# 五、生命周期绑定原理

例如：

```java
Glide.with(activity)
```

源码：

```java
SupportRequestManagerFragment
```

Glide偷偷创建一个：

```java
Fragment
```

加入：

```java
FragmentManager
```

源码：

```java
fragmentManager.beginTransaction()
               .add(current, TAG)
               .commitAllowingStateLoss();
```

---

当Activity销毁：

```java
onDestroy()
```

自动：

```java
RequestManager.onDestroy()
```

取消请求。

避免：

```java
ImageView泄漏
```

---

# 六、into()发生什么

源码：

```java
public ViewTarget<ImageView, TranscodeType>
into(ImageView view)
```

最终：

```java
buildRequest()
```

创建：

```java
SingleRequest
```

---

真正开始：

```java
request.begin();
```

源码：

```java
SingleRequest.begin()
```

↓

```java
Engine.load()
```

---

# 七、Glide三级缓存

面试重点。

缓存顺序：

```text
ActiveResources

↓

MemoryCache

↓

DiskCache

↓

Network
```

---

# 八、第一级缓存 ActiveResources

作用：

```java
正在使用中的Bitmap
```

例如：

```java
RecyclerView
```

当前显示：

```java
头像A
```

已经加载过。

再次加载：

```java
直接返回
```

---

源码：

```java
class ActiveResources {

    final Map<Key, ResourceWeakReference>
            activeEngineResources;
}
```

结构：

```java
HashMap
```

存储：

```java
WeakReference
```

---

查找：

```java
Engine.loadFromActiveResources()
```

源码：

```java
EngineResource<?> active =
       activeResources.get(key);
```

命中直接返回。

---

# 九、第二级缓存 MemoryCache

核心：

```java
LruResourceCache
```

继承：

```java
LruCache
```

源码：

```java
public class LruResourceCache
       extends LruCache<Key, Resource<?>>
```

---

查找：

```java
Engine.loadFromCache()
```

源码：

```java
memoryCache.remove(key);
```

命中：

```java
Bitmap直接复用
```

---

# 十、LruCache源码解析

Android源码：

```java
android.util.LruCache
```

核心：

```java
LinkedHashMap
```

源码：

```java
private final LinkedHashMap<K, V> map;
```

构造：

```java
new LinkedHashMap<>(
        0,
        0.75f,
        true
);
```

重点：

```java
accessOrder=true
```

表示：

```text
最近访问放尾部

最久未访问放头部
```

---

淘汰：

```java
trimToSize()
```

源码：

```java
while (size > maxSize) {

    Map.Entry<K,V> toEvict =
         map.entrySet()
            .iterator()
            .next();

    map.remove(toEvict.getKey());
}
```

典型LRU算法。

---

# 十一、第三级缓存 DiskCache

实现：

```java
DiskLruCache
```

源码：

```java
DiskLruCacheWrapper
```

---

缓存目录：

```java
/data/data/package/cache/image_manager_disk_cache
```

---

查找：

```java
DecodeJob
```

↓

```java
ResourceCacheGenerator
```

↓

```java
DataCacheGenerator
```

---

缓存命中：

```java
直接读取文件
```

避免网络请求。

---

# 十二、DiskLruCache原理

核心文件：

```text
journal
```

记录：

```text
CLEAN

DIRTY

READ

REMOVE
```

---

例如：

```text
DIRTY key1

CLEAN key1
```

表示：

```java
正在写

写入完成
```

---

源码：

```java
Editor editor = cache.edit(key);
```

写入：

```java
editor.commit();
```

---

# 十三、缓存查找源码链路

Engine.java

```java
load()
```

↓

```java
loadFromActiveResources()
```

↓

```java
loadFromMemory()
```

↓

```java
waitForExistingOrStartNewJob()
```

↓

```java
EngineJob
```

↓

```java
DecodeJob
```

---

源码：

```java
EngineResource<?> memoryResource =
        loadFromMemory(key);
```

---

如果没命中：

```java
new DecodeJob()
```

开始解码。

---

# 十四、网络加载源码

默认：

```java
HttpUrlConnection
```

如果接入：

```java
implementation
"com.github.bumptech.glide:okhttp3-integration"
```

则：

```java
OkHttp
```

负责下载。

---

源码：

```java
OkHttpStreamFetcher
```

↓

```java
Call.Factory
```

↓

```java
OkHttpClient
```

---

# 十五、Bitmap复用机制

面试高频。

Glide：

```java
BitmapPool
```

源码：

```java
LruBitmapPool
```

---

取Bitmap：

```java
bitmapPool.get()
```

---

回收：

```java
bitmapPool.put()
```

---

避免：

```java
频繁创建Bitmap
```

减少：

```java
GC

OOM
```

---

# 十六、DecodeJob源码

真正耗时工作都在：

```java
DecodeJob
```

实现：

```java
Runnable
```

源码：

```java
class DecodeJob<R>
        implements Runnable
```

执行：

```java
run()
```

↓

```java
decodeFromRetrievedData()
```

↓

```java
decode()
```

↓

```java
BitmapFactory.decodeStream()
```

---

# 十七、Glide线程池

源码：

```java
GlideExecutor
```

创建：

```java
new ThreadPoolExecutor(...)
```

分类：

```text
磁盘线程池

网络线程池

动画线程池
```

---

# 十八、面试官最爱追问

### 问：Glide为什么比BitmapFactory快？

答：

```text
1 Bitmap池复用

2 多级缓存

3 生命周期管理

4 自动压缩

5 线程池调度

6 解码优化
```

---

### 问：三级缓存顺序？

答：

```text
ActiveResources

↓

MemoryCache

↓

DiskCache

↓

Network
```

---

### 问：为什么ActiveResources放最前？

答：

因为当前界面正在使用：

```java
ImageView
```

直接返回即可。

避免：

```java
内存Cache再次查找
```

性能最好。

---

### 问：LruCache为什么使用LinkedHashMap？

答：

因为：

```java
accessOrder=true
```

天然支持：

```text
最近最少使用(LRU)
```

时间复杂度：

```text
get O(1)

put O(1)
```

---

# 面试总结（必须背）

```text
Glide核心加载流程：

Glide.with()
    ↓
RequestManager
    ↓
RequestBuilder
    ↓
SingleRequest
    ↓
Engine
    ↓
ActiveResources
    ↓
MemoryCache(LruCache)
    ↓
DiskCache(DiskLruCache)
    ↓
Network
    ↓
DecodeJob
    ↓
BitmapPool复用
    ↓
回调主线程显示

三级缓存：

ActiveResources
MemoryCache
DiskCache

缓存算法：

LruCache
LinkedHashMap(accessOrder=true)

磁盘缓存：

DiskLruCache
journal日志

性能优化：

BitmapPool
线程池
生命周期绑定
自动压缩
缓存复用
```

这套回答基本覆盖了阿里、字节、美团、腾讯、理想、小鹏、华为 Android 高级开发关于 Glide 的源码级面试深挖。