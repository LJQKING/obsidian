# Android 进程间通信（IPC）详解

Android 中，每个应用默认运行在独立进程中，不同进程拥有独立的虚拟机和内存空间，因此无法直接访问彼此的数据，需要通过 **IPC（Inter-Process Communication，进程间通信）** 来交换数据。

---

# 为什么需要 IPC？

例如：

- 微信分享图片给朋友圈
    
- 联系人应用提供联系人数据给其他 App
    
- 音乐播放器后台 Service 被多个页面控制
    
- 系统服务（AMS、PMS、WMS）与 App 通信
    

这些都涉及跨进程通信。

---

# Android 常见 IPC 方式

| 方式                | 使用场景            | 性能    |
| ----------------- | --------------- | ----- |
| Bundle            | Activity间简单数据传递 | ★★★★★ |
| 文件共享              | 简单数据共享          | ★★    |
| SharedPreferences | 配置共享            | ★★    |
| Messenger         | 单线程消息通信         | ★★★   |
| AIDL              | 高并发跨进程调用        | ★★★★★ |
| ContentProvider   | 数据共享            | ★★★★  |
| Socket            | 网络通信            | ★★★★  |
| Binder            | Android底层核心IPC  | ★★★★★ |

---

# 1. Bundle

最简单的 IPC。

```java
Intent intent = new Intent(this, MainActivity.class);
intent.putExtra("name", "Tom");
startActivity(intent);
```

接收：

```java
String name = getIntent().getStringExtra("name");
```

特点：

- 简单
    
- 只能传 Parcelable/Serializable
    
- 适用于组件间通信
    

---

# 2. 文件共享

进程A写文件：

```java
FileOutputStream fos =
    openFileOutput("user.txt", MODE_PRIVATE);

fos.write("Tom".getBytes());
```

进程B读取：

```java
FileInputStream fis =
    openFileInput("user.txt");
```

缺点：

- 效率低
    
- 需要同步控制
    

---

# 3. SharedPreferences

多进程共享配置。

旧方案：

```xml
android:process=":remote"
```

```java
getSharedPreferences(
        "config",
        MODE_MULTI_PROCESS);
```

但：

⚠️ MODE_MULTI_PROCESS 已废弃。

现在推荐：

- ContentProvider
    
- MMKV 多进程模式
    

例如：

[MMKV 官方网站](https://github.com/Tencent/MMKV?utm_source=chatgpt.com)

---

# 4. Messenger

基于 Binder + Handler。

适合：

- 客户端 → 服务端
    
- 不需要并发处理
    

---

## Service

```java
class MessengerService extends Service {

    private Handler handler =
        new Handler(Looper.getMainLooper()) {

        @Override
        public void handleMessage(Message msg) {

            if (msg.what == 1) {
                Log.d("TAG",
                     msg.getData()
                     .getString("msg"));
            }
        }
    };

    private Messenger messenger =
            new Messenger(handler);

    @Override
    public IBinder onBind(Intent intent) {
        return messenger.getBinder();
    }
}
```

---

## Client

```java
Messenger serviceMessenger;

Message message =
        Message.obtain();

message.what = 1;

Bundle bundle = new Bundle();
bundle.putString("msg","Hello");

message.setData(bundle);

serviceMessenger.send(message);
```

---

# 5. AIDL（面试重点）

AIDL = Android Interface Definition Language

适用于：

- 多客户端
    
- 高并发
    
- 大型项目
    

---

## AIDL 文件

```java
interface IUserManager {
    String getUserName();
}
```

---

编译后自动生成：

```java
IUserManager.Stub
```

---

## Service

```java
public class UserService extends Service {

    private IUserManager.Stub binder =
        new IUserManager.Stub() {

        @Override
        public String getUserName() {
            return "Tom";
        }
    };

    @Override
    public IBinder onBind(Intent intent) {
        return binder;
    }
}
```

---

## Client

```java
private ServiceConnection conn =
        new ServiceConnection() {

    @Override
    public void onServiceConnected(
            ComponentName name,
            IBinder service) {

        IUserManager manager =
                IUserManager.Stub
                .asInterface(service);

        String user =
                manager.getUserName();
    }
};
```

---

# AIDL 底层原理

调用：

```java
manager.getUserName();
```

实际上：

```text
Client
 ↓
Proxy
 ↓
Binder Driver
 ↓
Stub
 ↓
Service
```

流程：

```text
Client
  ↓
Proxy
  ↓
Parcel序列化
  ↓
Binder驱动
  ↓
Stub
  ↓
目标进程
```

---

# 6. ContentProvider

用于结构化数据共享。

典型案例：

- 联系人
    
- 相册
    
- 日历
    

Android 四大组件之一。

---

## Provider

```java
public class UserProvider
        extends ContentProvider {
}
```

---

## 查询

```java
Cursor cursor =
getContentResolver().query(
    Uri.parse(
      "content://user/info"),
    null,
    null,
    null,
    null);
```

---

# 联系人读取案例（面试常问）

读取系统联系人：

```java
Cursor cursor =
getContentResolver().query(
    ContactsContract.Contacts
        .CONTENT_URI,
    null,
    null,
    null,
    null);
```

这里实际通过：

- ContentResolver
    
- ContentProvider
    
- Binder
    

完成跨进程访问。

涉及实体：Android Contacts Provider

---

# 7. Socket

跨设备通信。

例如：

- 微信聊天
    
- IM
    
- 局域网通信
    

服务端：

```java
ServerSocket server =
        new ServerSocket(8888);
```

客户端：

```java
Socket socket =
        new Socket(
            "192.168.1.2",
            8888);
```

---

# Binder（Android IPC 核心）

Android 几乎所有 IPC 都基于 Binder。

例如：

- Activity启动
    
- Service启动
    
- Broadcast
    
- ContentProvider
    
- AIDL
    

都离不开 Binder。

---

# Binder VS Socket

|对比项|Binder|Socket|
|---|---|---|
|通信范围|本机|网络|
|性能|高|中|
|安全性|高|一般|
|Android支持|原生|通用|
|拷贝次数|1次|2次|

---

# Binder 为什么快？

传统 Socket：

```text
用户空间
 ↓
内核空间
 ↓
用户空间
```

需要：

**2 次内存拷贝**

---

Binder：

```text
Client
 ↓
Binder Driver
 ↓
Server
```

利用 mmap：

```text
一次拷贝
```

效率更高。

---

# Android 面试高频题

### 1. Android 有哪些 IPC 方式？

答：

- Bundle
    
- 文件共享
    
- SharedPreferences
    
- Messenger
    
- AIDL
    
- ContentProvider
    
- Socket
    
- Binder
    

---

### 2. AIDL 和 Messenger 区别？

|Messenger|AIDL|
|---|---|
|单线程|多线程|
|简单|复杂|
|不支持并发|支持并发|

---

### 3. ContentProvider 底层是什么？

答：

本质通过：

```text
ContentResolver
    ↓
Binder
    ↓
ContentProvider
```

实现跨进程通信。

---

### 4. Binder 为什么比 Socket 快？

答：

关键原因：

- 使用 mmap
    
- 减少数据拷贝
    
- 一次内存复制
    
- Android 专门优化
    

---

### 5. 系统服务如何与 App 通信？

例如：

```java
ActivityManager am =
(Context.ACTIVITY_SERVICE);
```

实际流程：

```text
App
 ↓
Binder
 ↓
SystemServer
 ↓
AMS
```

AMS 实体：ActivityManagerService

---

# Android高级工程师面试回答模板

> Android 中最核心的 IPC 机制是 Binder。AIDL、Messenger、ContentProvider 本质上都建立在 Binder 之上。对于简单通信我会使用 Bundle 或 Messenger；对于结构化数据共享会使用 ContentProvider；对于高并发跨进程调用则使用 AIDL。在实际项目中，例如蓝牙设备管理、音乐播放服务、多模块架构等场景，通常会通过 AIDL + Binder 实现稳定高效的跨进程通信。同时需要关注 Binder 线程池、Parcelable 序列化以及 ANR 风险等问题，这些也是面试官比较关注的源码级知识点。