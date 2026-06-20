# Binder IPC 原理（Android 系统服务通信核心）

Android 中的 **AMS、WMS、PMS、LocationManager、BluetoothManager** 等系统服务，本质上都是通过 **Binder IPC** 实现跨进程通信。

---

# 一、为什么需要 Binder

Android 四大组件运行在不同进程：

```text
App进程
   ↓
System Server进程
   ↓
Native Service进程
```

例如：

```java
startActivity(intent);
```

实际上：

```text
App进程
    ↓
ActivityManagerService(AMS)
    ↓
SystemServer
```

AMS 在另一个进程中。

普通 Java 方法无法跨进程调用。

因此需要：

```text
IPC（Inter Process Communication）
```

即：

```text
进程A 调用
      ↓
进程B 执行
      ↓
返回结果
```

---

# 二、Android IPC方案

Linux 原生：

```text
Pipe
Socket
Message Queue
Shared Memory
Signal
```

Android 选择：

```text
Binder
```

原因：

|方式|性能|安全|
|---|---|---|
|Socket|一般|差|
|Pipe|一般|差|
|SharedMemory|高|差|
|Binder|高|高|

Binder 特点：

```text
1 一次拷贝
2 权限校验
3 引用计数
4 面向对象
```

---

# 三、Binder架构

Binder 四层架构：

```text
┌─────────────────────┐
│ Application         │
│ ActivityManager     │
└─────────┬───────────┘
          │
┌─────────▼───────────┐
│ Java Binder         │
│ BinderProxy         │
│ Binder              │
└─────────┬───────────┘
          │ JNI
┌─────────▼───────────┐
│ Native Binder       │
│ BpBinder            │
│ BBinder             │
└─────────┬───────────┘
          │ ioctl
┌─────────▼───────────┐
│ Binder Driver       │
│ /dev/binder         │
└─────────────────────┘
```

---

# 四、Binder核心角色

Binder通信有4个角色：

```text
Client
Server
ServiceManager
Binder Driver
```

例如：

```java
startActivity()
```

实际：

```text
Client(App)
      ↓
Binder Driver
      ↓
AMS(Server)

ServiceManager负责查找AMS
```

---

# 五、一次Binder调用过程

例如：

```java
ActivityManager.getService()
        .startActivity(...)
```

流程：

```text
APP
 ↓
Proxy
 ↓
Parcel写数据
 ↓
Binder Driver
 ↓
AMS Stub
 ↓
AMS执行
 ↓
结果返回
```

图示：

```text
Client Process

Proxy
  |
transact()

=================
Binder Driver
=================

onTransact()

Stub

Server Process
```

---

# 六、AIDL生成代码

AIDL：

```aidl
interface IBookManager {
    void addBook(String name);
}
```

编译后生成：

```java
IBookManager.java
```

包含：

```java
Stub
Proxy
```

---

## Stub

服务端

```java
public abstract static class Stub
        extends Binder
        implements IBookManager
{
}
```

负责：

```text
接收Binder请求
解析Parcel
调用真正方法
```

---

## Proxy

客户端

```java
private static class Proxy
        implements IBookManager
{
}
```

负责：

```text
封装请求
发送给Binder Driver
```

---

# 七、源码分析

## Client调用

```java
proxy.addBook("Android");
```

进入：

```java
Proxy.addBook()
```

源码：

```java
Parcel data = Parcel.obtain();
Parcel reply = Parcel.obtain();

data.writeString("Android");

mRemote.transact(
        TRANSACTION_addBook,
        data,
        reply,
        0
);
```

这里：

```text
写入Parcel
发起transact
```

---

# 八、进入BinderProxy

Java层：

```java
BinderProxy.transact()
```

↓

JNI：

```cpp
android_os_BinderProxy_transact()
```

↓

Native：

```cpp
BpBinder::transact()
```

↓

Binder Driver：

```cpp
ioctl(fd, BINDER_WRITE_READ)
```

这里正式进入内核。

---

# 九、Binder Driver

驱动位置：

```text
drivers/android/binder.c
```

主要函数：

```cpp
binder_ioctl()
```

收到：

```text
BC_TRANSACTION
```

创建：

```cpp
binder_transaction
```

结构：

```cpp
struct binder_transaction {
    binder_proc* from;
    binder_proc* to;
    binder_buffer* buffer;
};
```

保存：

```text
发送方
接收方
数据
```

---

# 十、Server收到请求

Binder线程池：

```cpp
binder_thread_read()
```

取出：

```text
BR_TRANSACTION
```

然后：

```cpp
BBinder::transact()
```

↓

```cpp
BBinder::onTransact()
```

↓

```java
Stub.onTransact()
```

---

# 十一、Stub.onTransact

AIDL自动生成：

```java
@Override
public boolean onTransact(
        int code,
        Parcel data,
        Parcel reply,
        int flags)
{
    switch(code)
    {
        case TRANSACTION_addBook:
        {
            String name =
                data.readString();

            addBook(name);

            reply.writeNoException();

            return true;
        }
    }
}
```

作用：

```text
Parcel反序列化
调用真实方法
写回结果
```

---

# 十二、结果返回

服务端：

```java
reply.writeString("success");
```

↓

```text
Binder Driver
```

↓

```text
Client
```

↓

```java
reply.readString();
```

得到结果。

---

# 十三、Binder一次拷贝原理

传统Socket：

```text
用户空间
   ↓
内核
   ↓
用户空间
```

需要：

```text
2次拷贝
4次上下文切换
```

Binder：

```text
Client
   ↓
共享映射内存
   ↓
Server
```

只需：

```text
1次数据拷贝
```

因此性能更高。

---

# 十四、ServiceManager

系统启动时：

```text
AMS
WMS
PMS
```

都会注册：

```java
ServiceManager.addService()
```

例如：

```java
ServiceManager.addService(
    "activity",
    ams
);
```

保存到：

```text
ServiceManager
```

客户端获取：

```java
IBinder binder =
        ServiceManager.getService(
                "activity");
```

然后：

```java
IActivityManager.Stub
        .asInterface(binder);
```

得到：

```java
Proxy对象
```

---

# 十五、AMS调用源码

## startActivity

应用：

```java
startActivity(intent);
```

↓

```java
Instrumentation.execStartActivity()
```

↓

```java
ActivityTaskManager.getService()
```

↓

```java
IActivityTaskManager
```

↓

```java
Binder Proxy
```

↓

```java
transact()
```

↓

```java
ActivityTaskManagerService
```

↓

```java
ActivityStarter.execute()
```

↓

启动Activity

---

# 十六、Binder线程池

服务端不是主线程执行。

Binder Driver唤醒：

```cpp
binder_thread
```

线程池：

```text
Binder:1234_1
Binder:1234_2
Binder:1234_3
```

查看：

```bash
adb shell ps -T
```

会看到：

```text
Binder线程
```

---

# 十七、死亡通知机制

客户端监听：

```java
binder.linkToDeath(...)
```

服务端崩溃：

```text
Binder Driver
    ↓
DeathRecipient
```

回调：

```java
binderDied()
```

例如：

```java
BluetoothService挂掉
```

应用立即收到通知。

---

# 十八、Binder源码调用链（面试必背）

```text
Client

Proxy.addBook()

↓
BinderProxy.transact()

↓ JNI

android_os_BinderProxy_transact()

↓

BpBinder::transact()

↓

ioctl()

↓

Binder Driver

↓

binder_transaction()

↓

Server

↓

BBinder::transact()

↓

BBinder::onTransact()

↓

Stub.onTransact()

↓

真实业务方法

↓

reply

↓

返回Client
```

---

# 十九、面试高频问题

### 1. Binder为什么比Socket快？

答：

```text
Binder使用内存映射(mmap)
只需一次拷贝

Socket需要两次拷贝
```

---

### 2. Binder为什么安全？

答：

Binder Driver 自动携带：

```cpp
uid
pid
```

服务端可直接获取：

```java
Binder.getCallingUid()
Binder.getCallingPid()
```

进行权限校验。

---

### 3. AIDL生成哪些类？

```text
Stub
Proxy
Default
```

核心：

```text
Stub 服务端
Proxy 客户端
```

---

### 4. AMS/WMS/PMS如何通信？

```text
SystemServer中注册Service

ServiceManager

↓

Client获取Binder

↓

Proxy.transact()

↓

Stub.onTransact()

↓

AMS/WMS/PMS执行
```

---

# 一张图看懂 Binder

```text
App进程
──────────────────────

Proxy
  ↓
Parcel
  ↓
BinderProxy.transact()

══════════════════════
Binder Driver
/dev/binder
══════════════════════

BBinder
  ↓
Stub.onTransact()
  ↓
AMS/WMS/PMS

SystemServer进程
──────────────────────
```

理解这条链路后，再看 **AMS 启动 Activity、WMS 添加窗口、PMS 安装 APK、AIDL 跨进程通信**，本质上都是同一个 Binder 调用模型。