# Android MMKV 框架详解（面试必问）

MMKV 是腾讯微信团队开源的高性能 Key-Value 存储框架，主要用于替代 Android 的 SharedPreferences。其核心特点是：

- 读写速度快
    
- 支持多进程
    
- 支持加密
    
- 支持 mmap 内存映射
    
- 微信生产环境长期使用
    

官方项目：

[Tencent MMKV GitHub](https://github.com/Tencent/MMKV?utm_source=chatgpt.com)

MMKV 基于 mmap（Memory Mapping）和 Protobuf 实现，而不是 XML 文件存储。([GitHub](https://github.com/Tencent/MMKV?utm_source=chatgpt.com "GitHub - Tencent/MMKV: An efficient, small mobile key-value storage framework developed by WeChat. Works on Android, iOS, macOS, Windows, POSIX, and OHOS."))

---

# 一、为什么 SharedPreferences 慢？

SharedPreferences 存储结构：

```xml
<map>
    <string name="token">abc</string>
    <int name="age" value="18"/>
</map>
```

写入流程：

```java
editor.putString("token","abc");
editor.apply();
```

内部：

```java
Map<String,Object>
      ↓
XML序列化
      ↓
磁盘IO
```

问题：

### 1 XML解析耗时

```java
XML -> Map
Map -> XML
```

大量数据性能下降。

---

### 2 全量写入

即使修改一个字段：

```java
token
```

也需要：

```java
整个XML重新写盘
```

---

### 3 多进程不安全

```java
A进程写
B进程写
```

容易数据覆盖。

---

# 二、MMKV核心原理

架构：

```text
Java
 ↓ JNI
C++
 ↓
MMKV Core
 ↓
mmap
 ↓
文件
```

源码目录：

```text
Android/MMKV
Core/MMKV.cpp
```

官方实现使用：

```cpp
mmap()
msync()
protobuf
```

完成数据存储。([GitHub](https://github.com/Tencent/MMKV?utm_source=chatgpt.com "GitHub - Tencent/MMKV: An efficient, small mobile key-value storage framework developed by WeChat. Works on Android, iOS, macOS, Windows, POSIX, and OHOS."))

---

# 三、什么是 mmap

传统IO：

```text
Disk
 ↓
Kernel Buffer
 ↓ copy
User Buffer
```

发生两次拷贝。

---

MMKV：

```text
Disk File
     ↕
 mmap
     ↕
Process Memory
```

直接映射到进程地址空间。

代码示意：

```cpp
void* ptr = mmap(
    nullptr,
    size,
    PROT_READ | PROT_WRITE,
    MAP_SHARED,
    fd,
    0
);
```

效果：

```java
写内存
≈
写文件
```

减少：

```java
read()
write()
```

系统调用。

---

# 四、MMKV存储结构

SharedPreferences：

```text
XML
```

MMKV：

```text
protobuf binary
```

例如：

```java
kv.encode("name","Tom");
kv.encode("age",18);
```

实际：

```text
name → bytes
age  → bytes
```

序列化：

```protobuf
key_length
key
value_length
value
```

比 XML 小很多。([GitHub](https://github.com/Tencent/MMKV?utm_source=chatgpt.com "GitHub - Tencent/MMKV: An efficient, small mobile key-value storage framework developed by WeChat. Works on Android, iOS, macOS, Windows, POSIX, and OHOS."))

---

# 五、MMKV初始化

Gradle：

```gradle
implementation "com.tencent:mmkv:2.4.0"
```

当前 Wiki 示例版本为 2.4.0。([GitHub](https://github.com/Tencent/MMKV/wiki/android_setup?utm_source=chatgpt.com "android_setup · Tencent/MMKV Wiki · GitHub"))

Application：

```java
public class App extends Application {

    @Override
    public void onCreate() {
        super.onCreate();

        String rootDir =
            MMKV.initialize(this);

        Log.d("MMKV", rootDir);
    }
}
```

---

# 六、基本使用

## 保存数据

```java
MMKV kv = MMKV.defaultMMKV();

kv.encode("name", "Tom");
kv.encode("age", 18);
kv.encode("vip", true);
```

---

## 读取数据

```java
String name =
    kv.decodeString("name");

int age =
    kv.decodeInt("age");

boolean vip =
    kv.decodeBool("vip");
```

---

## 删除

```java
kv.removeValueForKey("name");
```

---

## 清空

```java
kv.clearAll();
```

---

# 七、多实例

例如：

```java
用户信息
缓存
配置
```

分开存储。

```java
MMKV userKv =
    MMKV.mmkvWithID("user");

MMKV cacheKv =
    MMKV.mmkvWithID("cache");
```

生成：

```text
/user
/cache
```

独立文件。

---

# 八、多进程支持

SharedPreferences：

```java
MODE_MULTI_PROCESS
```

已经废弃。

---

MMKV：

```java
MMKV mmkv =
    MMKV.mmkvWithID(
        "config",
        MMKV.MULTI_PROCESS_MODE
    );
```

支持：

```text
主进程
RemoteService
Push进程
WebView进程
```

并发读写。([GitHub](https://github.com/Tencent/MMKV?utm_source=chatgpt.com "GitHub - Tencent/MMKV: An efficient, small mobile key-value storage framework developed by WeChat. Works on Android, iOS, macOS, Windows, POSIX, and OHOS."))

---

# 九、数据加密

微信大量使用。

创建：

```java
String key = "123456";

MMKV mmkv =
    MMKV.mmkvWithID(
        "secure",
        MMKV.SINGLE_PROCESS_MODE,
        key
    );
```

底层：

```cpp
AES
```

加密存储。

---

# 十、源码流程分析

## encode()

Java：

```java
kv.encode("token","abc");
```

↓

JNI

```cpp
nativeEncodeString()
```

↓

MMKV.cpp

```cpp
MMKV::set()
```

↓

Protobuf编码

↓

写入mmap

↓

msync

````

```text
encode()
   ↓
JNI
   ↓
MMKV::set()
   ↓
protobuf
   ↓
mmap
   ↓
file
````

---

# 十一、读取流程

```java
kv.decodeString("token");
```

↓

JNI

↓

MMKV::get()

↓

从 mmap 内存直接读取

↓

返回 Java

```text
decode()
   ↓
JNI
   ↓
MMKV::get()
   ↓
memory
```

几乎不需要磁盘IO。

---

# 十二、性能为什么快

MMKV官方 Benchmark 显示，在大量读写场景下，相比 SharedPreferences 和 SQLite，MMKV 写入性能更优，读取性能也具有竞争力。([GitHub](https://github.com/Tencent/MMKV/wiki/android_benchmark?utm_source=chatgpt.com "android_benchmark · Tencent/MMKV Wiki · GitHub"))

主要原因：

### mmap

避免频繁 read/write

---

### protobuf

二进制存储

```text
XML：
<name>Tom</name>

protobuf：
0x01 0x02
```

体积更小。

---

### 增量更新

修改：

```java
age
```

无需重写整个 XML。

---

# 十三、面试高频题

### MMKV为什么比SharedPreferences快？

答：

1. mmap内存映射
    
2. protobuf二进制序列化
    
3. 避免XML解析
    
4. 增量写入
    
5. 减少系统调用
    

---

### MMKV底层用了什么技术？

答：

```text
JNI
C++
mmap
protobuf
文件锁
AES
```

---

### MMKV为什么支持多进程？

答：

通过：

```cpp
FileLock
InterProcessLock
```

实现进程间同步。([GitHub](https://github.com/Tencent/MMKV?utm_source=chatgpt.com "GitHub - Tencent/MMKV: An efficient, small mobile key-value storage framework developed by WeChat. Works on Android, iOS, macOS, Windows, POSIX, and OHOS."))

---

### MMKV和DataStore怎么选？

|对比项|MMKV|DataStore|
|---|---|---|
|性能|极高|中等|
|多进程|支持|不支持|
|加密|支持|需自行实现|
|Kotlin Flow|不支持|支持|
|JNI|需要|不需要|
|适合场景|高频缓存|Jetpack项目|

---

# Android高级面试回答（推荐背诵）

> MMKV 是腾讯微信开源的高性能 Key-Value 存储框架，底层采用 mmap + Protobuf + JNI + C++ 实现。相比 SharedPreferences 的 XML 全量读写，MMKV 通过内存映射文件直接操作内存，大幅减少 IO 和序列化开销，同时支持多进程访问和数据加密，因此在微信等大型 App 中被广泛使用。其整体架构为 Java → JNI → MMKV Core(C++) → mmap → File。([GitHub](https://github.com/Tencent/MMKV?utm_source=chatgpt.com "GitHub - Tencent/MMKV: An efficient, small mobile key-value storage framework developed by WeChat. Works on Android, iOS, macOS, Windows, POSIX, and OHOS."))

如果你准备 Android 3-8 年经验岗位面试，我还可以继续给你做 **MMKV源码级分析（mmap、JNI、Protobuf、文件锁、多进程同步机制）**，包括核心 C++ 源码执行流程图。