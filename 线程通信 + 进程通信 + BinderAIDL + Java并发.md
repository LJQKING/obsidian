
> **Android App → Binder/AIDL → CarService/AdaptApi → HIDL/AIDL → VHAL → BSP/MCU → CAN**

# 一、先建立整个岗位的通信体系

你可以把所有通信问题分成 4 层：

```text
                    Android 车载系统
┌──────────────────────────────────────────────┐
│                 App 进程                     │
│                                              │
│ UI Thread                                    │
│   │                                          │
│   ├── Handler / Looper / MessageQueue        │
│   ├── Executor / ThreadPoolExecutor          │
│   ├── Kotlin Coroutine / Flow                │
│   │                                          │
│   └────────────── Binder IPC ─────────────┐  │
└───────────────────────────────────────────│──┘
                                            ↓
┌──────────────────────────────────────────────┐
│              System Server / CarService      │
│                                              │
│ Binder ThreadPool                            │
│       ↓                                      │
│ CarPropertyService                           │
│       ↓                                      │
│ VehicleHal                                   │
└───────────────────────┬──────────────────────┘
                        ↓
                 HIDL / AIDL
                        ↓
┌──────────────────────────────────────────────┐
│              HAL / Native                    │
│                                              │
│ Native Daemon / C++                          │
│       ↓                                      │
│ BSP / Driver                                 │
│       ↓                                      │
│ MCU                                          │
│       ↓                                      │
│ CAN / LIN                                    │
└──────────────────────────────────────────────┘
```

这正好对应 JD 中的车辆信号链路。

所以面试官很可能从：

> `Handler → 线程池 → synchronized → Binder → AIDL → CarService → VHAL`

一路往下追。

---

# 二、第一重点：Android线程通信

## 1. Handler为什么可以实现线程通信？

这是非常高频的问题。

最简单代码：

```java
HandlerThread thread = new HandlerThread("VehicleThread");
thread.start();

Handler handler = new Handler(thread.getLooper());

handler.post(() -> {
    System.out.println("执行在线程：" +
            Thread.currentThread().getName());
});
```

面试官问：

> Handler为什么可以让任务跑到指定线程？

你不要回答：

> Handler就是切线程的。

这个回答太浅。

应该回答：

```text
Handler
   ↓
Looper
   ↓
MessageQueue
   ↓
目标线程
```

核心原理：

```java
handler.post(runnable)
```

实际上最终会：

```text
Runnable
 ↓
Message
 ↓
MessageQueue.enqueueMessage()
 ↓
Looper.loop()
 ↓
MessageQueue.next()
 ↓
Handler.dispatchMessage()
 ↓
Runnable.run()
```

---

# 三、Handler、Looper、MessageQueue三者关系

这是 Android 面试非常经典的一题。

### Looper

负责不断从 MessageQueue 中取消息：

```java
public static void loop() {

    for (;;) {

        Message msg = queue.next();

        if (msg == null) {
            continue;
        }

        msg.target.dispatchMessage(msg);
    }
}
```

### MessageQueue

负责存放消息。

```text
Message
Message
Message
Message
   ↓
MessageQueue
```

### Handler

负责：

```text
发送消息
+
处理消息
```

所以：

```text
Handler = 生产者/消费者接口

MessageQueue = 消息队列

Looper = 消费循环
```

---

# 四、为什么子线程不能直接创建Handler？

经常被问。

错误：

```java
new Thread(() -> {

    Handler handler = new Handler();

}).start();
```

可能抛：

```text
Can't create handler inside thread that has not called Looper.prepare()
```

原因：

```text
线程
 ↓
Looper.prepare()
 ↓
创建ThreadLocal中的Looper
 ↓
创建MessageQueue
 ↓
Looper.loop()
```

正确：

```java
new Thread(() -> {

    Looper.prepare();

    Handler handler = new Handler();

    Looper.loop();

}).start();
```

但是实际开发更推荐：

```java
HandlerThread thread =
        new HandlerThread("VehicleThread");

thread.start();

Handler handler =
        new Handler(thread.getLooper());
```

---

# 五、HandlerThread是什么？

面试官可能问：

> HandlerThread和普通Thread有什么区别？

核心：

```text
Thread
    ↓
自己没有Looper

HandlerThread
    ↓
Thread + Looper + MessageQueue
```

源码逻辑可以理解为：

```java
class HandlerThread extends Thread {

    @Override
    public void run() {

        Looper.prepare();

        looper = Looper.myLooper();

        Looper.loop();
    }
}
```

### 车载场景

比如：

```text
CAN车辆信号
      ↓
VehicleService
      ↓
HandlerThread
      ↓
解析
      ↓
通知UI
```

避免把：

```text
CAN信号解析
+
数据库
+
网络
+
UI
```

全部堆在主线程。

---

# 六、Handler vs Executor线程池

这是这个岗位非常值得准备的对比题。

|技术|核心|适合|
|---|---|---|
|Handler|消息队列|串行任务、线程切换|
|HandlerThread|单线程Looper|后台串行任务|
|Executor|线程池|并发任务|
|ThreadPoolExecutor|可配置线程池|大量任务|
|ScheduledExecutor|定时任务|心跳/周期任务|
|Coroutine|协程|异步业务|
|Flow|数据流|状态/事件流|

### Handler

```java
handler.post(() -> {
});
```

特点：

```text
一个Looper
↓
一个线程
↓
任务默认串行
```

### ThreadPoolExecutor

```java
ExecutorService executor =
        Executors.newFixedThreadPool(4);

executor.execute(() -> {
});
```

特点：

```text
多个Worker Thread
        ↓
      Queue
        ↓
    并发执行
```

---

# 七、ThreadPoolExecutor必须会

你之前已经在研究线程池，这个岗位非常值得继续深入。

构造：

```java
ThreadPoolExecutor(
    corePoolSize,
    maximumPoolSize,
    keepAliveTime,
    unit,
    workQueue,
    threadFactory,
    handler
);
```

真正重要的是：

```text
提交任务
   ↓
当前线程数 < core
   ↓
创建核心线程
   ↓
否则进入 BlockingQueue
   ↓
队列满
   ↓
线程数 < maximum
   ↓
创建非核心线程
   ↓
否则
   ↓
RejectedExecutionHandler
```

面试官：

> 为什么线程池不是线程不够了就创建线程？

回答：

因为 ThreadPoolExecutor 的设计优先使用：

```text
核心线程
   ↓
任务队列
   ↓
非核心线程
```

目的是控制线程创建成本。

---

# 八、车载场景：线程池怎么设计？

例如同时收到：

```text
车速
空调
车门
导航
T-Box
```

可以：

```java
ExecutorService vehicleExecutor =
        new ThreadPoolExecutor(
                2,
                4,
                30,
                TimeUnit.SECONDS,
                new LinkedBlockingQueue<>(100)
        );
```

然后：

```java
vehicleExecutor.execute(() -> {
    processVehicleSpeed();
});

vehicleExecutor.execute(() -> {
    processHvac();
});

vehicleExecutor.execute(() -> {
    processDoor();
});
```

但是有一个非常重要的问题：

> **车速这种高频信号适不适合无限排队？**

不一定。

例如：

```text
60
61
62
63
64
65
66
67
...
```

如果每一个都进入队列：

```text
Queue
60
61
62
63
64
65
...
```

实际上 UI 只需要最新值。

所以可以设计成：

```text
旧车速
   ↓
覆盖
   ↓
最新车速
```

这就是车载场景非常好的并发设计问题。

---

# 九、wait/notify是什么？

Java经典题。

```java
synchronized (lock) {

    while (!condition) {
        lock.wait();
    }

    // 执行任务
}
```

生产者：

```java
synchronized (lock) {

    data = newData;

    lock.notifyAll();
}
```

核心：

```text
wait()
 ↓
释放锁
 ↓
进入等待状态

notify()
 ↓
唤醒等待线程
```

注意：

### `sleep()`不会释放锁

```java
Thread.sleep(1000);
```

锁还在。

### `wait()`会释放锁

```java
lock.wait();
```

锁释放。

这是经典面试题。

---

# 十、wait/notify和BlockingQueue有什么区别？

推荐回答：

```text
wait/notify
    ↓
底层线程同步机制
    ↓
自己管理条件

BlockingQueue
    ↓
封装好的生产者消费者模型
```

例如：

```java
BlockingQueue<CarPropertyValue> queue =
        new LinkedBlockingQueue<>();
```

生产：

```java
queue.put(value);
```

消费：

```java
CarPropertyValue value =
        queue.take();
```

不用自己：

```java
wait();
notify();
```

实际业务开发更推荐 BlockingQueue。

---

# 十一、CountDownLatch

例如：

> 车机启动的时候需要同时初始化三个模块。

```text
Vehicle
Audio
Navigation
```

```java
CountDownLatch latch =
        new CountDownLatch(3);
```

三个任务：

```java
executor.execute(() -> {
    initVehicle();
    latch.countDown();
});

executor.execute(() -> {
    initAudio();
    latch.countDown();
});

executor.execute(() -> {
    initNavigation();
    latch.countDown();
});
```

等待：

```java
latch.await();
```

表示：

```text
3
 ↓
2
 ↓
1
 ↓
0
 ↓
继续执行
```

---

# 十二、volatile必须重点准备

面试官：

> volatile解决什么问题？

三个关键词：

```text
可见性
有序性
不保证原子性
```

例如：

```java
private volatile boolean running = true;
```

线程A：

```java
while (running) {
}
```

线程B：

```java
running = false;
```

volatile保证线程A能够及时看到修改。

---

# 十三、为什么volatile不能保证i++？

```java
volatile int count;

count++;
```

实际上：

```text
读取count
   ↓
count + 1
   ↓
写回count
```

多个线程：

```text
Thread A       Thread B

read 0         read 0
+1             +1
write 1        write 1
```

最后：

```text
1
```

而不是：

```text
2
```

所以：

```text
volatile
≠
原子性
```

---

# 十四、synchronized原理

这个岗位非常可能问。

```java
synchronized (lock) {

}
```

核心：

```text
Monitor
```

进入：

```text
monitorenter
```

退出：

```text
monitorexit
```

可以理解成：

```text
线程
 ↓
尝试获取Monitor
 ↓
成功
 ↓
执行临界区
 ↓
释放Monitor
```

Java对象和Monitor存在关联。

面试不要只回答：

> synchronized可以加锁。

应该进一步说：

> synchronized是基于对象Monitor实现的互斥同步机制，JVM层通过monitorenter/monitorexit指令控制锁的进入和退出。

---

# 十五、synchronized和ReentrantLock

||synchronized|ReentrantLock|
|---|---|---|
|JVM原生|是|Java API|
|自动释放|是|否|
|tryLock|❌|✅|
|可中断|❌|✅|
|公平锁|不直接提供|✅|
|Condition|❌|✅|
|使用复杂度|低|高|

代码：

```java
ReentrantLock lock =
        new ReentrantLock();

lock.lock();

try {

    process();

} finally {

    lock.unlock();
}
```

面试回答：

> 如果只是简单互斥，synchronized优先；如果需要可中断、公平锁、tryLock、多个Condition等高级能力，可以使用ReentrantLock。

---

# 十六、第二大重点：Android进程通信

这里才是这个 JD 的核心。

JD明确要求：

```text
App
 ↓
Binder / AIDL
 ↓
CarService / AdaptApi
```

所以：

# Binder一定要准备到源码原理级。

---

# 十七、Android IPC有哪些方式？

建议记这个表：

|IPC方式|原理|适合|
|---|---|---|
|Binder|Binder驱动|Android核心IPC|
|AIDL|Binder封装|跨进程Service|
|Messenger|AIDL + Handler|简单消息|
|Broadcast|AMS/Binder|广播事件|
|ContentProvider|Binder|数据共享|
|Socket|TCP/Unix Socket|大量流数据|
|文件|文件系统|简单数据共享|
|SharedMemory|共享内存|大数据|
|HIDL/AIDL HAL|Binder|Framework/HAL|

---

# 十八、为什么Android不用Socket作为主要IPC？

这是非常好的高级题。

Android选择Binder主要因为：

```text
性能
安全
身份管理
对象引用
内核支持
```

传统Socket：

```text
Process A
 ↓
copy
Kernel
 ↓
copy
Process B
```

Binder在Android中提供了专门的IPC机制：

```text
Client
 ↓
Binder Proxy
 ↓
Binder Driver
 ↓
Binder Node
 ↓
Server Stub
 ↓
Service
```

而且Binder天然携带：

```text
UID
PID
调用者身份
```

所以系统服务可以：

```java
Binder.getCallingUid();
```

判断调用者。

这对车控功能特别重要。

比如：

```text
普通App
 ↓
setHvacTemperature()
 ↓
CarService
 ↓
权限检查
 ↓
允许/拒绝
```

---

# 十九、AIDL到底是什么？

AIDL：

> Android Interface Definition Language

例如：

```aidl
interface IVehicleService {

    float getVehicleSpeed();

    void setHvacTemperature(int zone, float temperature);

}
```

然后：

```text
Client App
   ↓
IVehicleService.Stub.Proxy
   ↓
Binder Driver
   ↓
IVehicleService.Stub
   ↓
VehicleService
```

---

# 二十、AIDL底层到底发生了什么？

这是面试重点。

客户端：

```java
service.setHvacTemperature(0, 24);
```

表面看起来像普通方法调用。

实际上：

```text
App
 ↓
Proxy.setHvacTemperature()
 ↓
Parcel.writeInt()
Parcel.writeFloat()
 ↓
transact()
 ↓
Binder Driver
 ↓
Server Stub.onTransact()
 ↓
Parcel.readInt()
Parcel.readFloat()
 ↓
VehicleService.setHvacTemperature()
```

也就是说：

```text
Java方法调用
```

被转换成：

```text
跨进程Binder事务
```

---

# 二十一、为什么Binder调用不能传太大的数据？

这是高频题。

Binder Transaction Buffer存在大小限制，通常面试会记：

> 单次Binder事务数据大小大约受 **1 MB级别** 的缓冲区限制，实际可用空间还会受到其他并发事务影响。

所以：

### 错误

```java
aidlService.sendHugeBitmap(bitmap);
```

### 正确

```text
Bitmap
 ↓
文件
 ↓
共享内存
 ↓
ParcelFileDescriptor
 ↓
Binder传文件描述符
```

---

# 二十二、AIDL中的in/out/inout

例如：

```aidl
void sendData(in VehicleData data);
```

### in

客户端：

```text
Client
 ↓
数据
 ↓
Server
```

### out

```text
Server
 ↓
数据
 ↓
Client
```

### inout

```text
Client
 ↓
Server
 ↓
Client
```

核心是：

> AIDL需要对跨进程数据进行序列化/反序列化，因此参数必须是Binder支持的类型或Parcelable等可序列化类型。

---

# 二十三、Binder线程池

这个是非常重要的高级题。

很多人以为：

```text
Binder调用
 ↓
Server主线程
```

实际上不是简单这么理解。

服务端进程会有：

```text
Binder Thread Pool
```

多个Binder线程可以同时处理IPC请求。

所以：

```java
public void setHvacTemperature(...) {

    // 不一定运行在主线程
}
```

因此服务端不能随便操作UI。

而且：

```text
多个Binder请求
 ↓
可能并发执行
```

所以Service内部数据需要考虑：

```text
synchronized
Lock
Atomic
ConcurrentHashMap
```

---

# 二十四、AIDL oneway

例如：

```aidl
oneway interface ICarCallback {

    void onVehicleSpeedChanged(float speed);
}
```

`oneway`的核心思想：

> 调用方不需要同步等待服务端执行完成。

适合：

```text
事件通知
回调
```

例如：

```text
VHAL
 ↓
CarService
 ↓
AIDL callback
 ↓
App
```

但不要把：

```text
需要返回结果
强一致操作
```

随便设计成 `oneway`。

---

# 二十五、Binder死亡通知

车载系统特别重要。

比如：

```text
App
 ↓
CarService
```

如果：

```text
CarService进程挂了
```

App不能一直认为Binder有效。

可以：

```java
binder.linkToDeath(() -> {

    // Binder死亡

    reconnect();

}, 0);
```

面试回答：

> Binder提供DeathRecipient机制，客户端可以监听远程Binder对象死亡，在服务进程异常退出后重新获取Service并恢复监听状态。

---

# 二十六、车载真实场景：AIDL + CarService

这道题非常适合这个岗位。

面试官：

> 如果让你设计一个车机空调服务，App如何控制空调？

你可以这样回答。

```text
IVI App
   ↓
IVehicleClimateManager
   ↓
AIDL
   ↓
ClimateService
   ↓
VehicleService
   ↓
VHAL
   ↓
CAN
   ↓
MCU
   ↓
空调控制器
```

接口：

```aidl
interface IClimateService {

    float getTemperature(int zone);

    void setTemperature(int zone, float temperature);

    void registerCallback(IClimateCallback callback);

}
```

App：

```java
service.setTemperature(0, 24f);
```

Server：

```java
@Override
public void setTemperature(int zone, float temperature) {

    checkPermission();

    vehicleAdapter.setHvacTemperature(
        zone,
        temperature
    );
}
```

Adapter：

```java
interface IVehicleAdapter {

    void setHvacTemperature(
        int zone,
        float temperature
    );
}
```

这样就把：

```text
App业务
```

和：

```text
高通/瑞萨/NXP平台差异
```

隔离开。

这正是 JD 中 `CarService + AdaptApi` 的设计思想。

---

# 二十七、AIDL和Messenger区别

这是经典题。

Messenger本质：

```text
Messenger
 ↓
AIDL
 ↓
Handler
 ↓
Message
```

所以：

### Messenger

适合：

```text
简单消息通信
```

例如：

```text
Client
  ↓
Message
  ↓
Service
```

### AIDL

适合：

```text
复杂接口
大量方法
回调
高性能IPC
系统服务
```

车载：

```text
CarService
VehicleService
TBoxService
```

明显更适合 AIDL。

---

# 二十八、AIDL和Broadcast区别

||AIDL|Broadcast|
|---|---|---|
|类型|定向IPC|事件广播|
|是否有接口|有|Intent|
|返回结果|可以|不适合|
|实时性|高|相对弱|
|一对一|非常适合|不擅长|
|一对多|Callback可实现|很适合|
|系统服务|常用|事件通知|

例如：

### 空调控制

```text
AIDL
```

### “车辆进入倒车状态”

```text
Broadcast/Event
```

不过车载系统更底层的车辆属性变化，通常会走 CarProperty/VHAL 这套机制，而不是让每个 App 自己监听广播。JD也明确强调通过 `CarPropertyManager` 订阅车辆属性。

---

# 二十九、ContentProvider是什么IPC？

很多人误以为：

> ContentProvider只是数据库。

不准确。

本质：

```text
Client
 ↓
ContentResolver
 ↓
Binder
 ↓
ContentProvider
 ↓
SQLite / Room / File
```

所以：

> ContentProvider也是基于Binder的跨进程数据访问机制。

适合：

```text
结构化数据共享
```

例如：

```text
导航App
 ↓
ContentProvider
 ↓
地图数据
```

---

# 三十、Broadcast底层原理

广播：

```java
sendBroadcast(intent);
```

并不是：

```text
直接找到Receiver
```

而是大体：

```text
App
 ↓
Binder
 ↓
ActivityManagerService
 ↓
广播队列
 ↓
目标进程
 ↓
Receiver
```

所以 Broadcast 本身也是：

```text
Binder + AMS
```

---

# 三十一、同进程通信 vs 跨进程通信

一定要会。

### 同进程

```java
object.method();
```

本质：

```text
直接内存访问
```

### 跨进程

```text
Proxy
 ↓
Parcel
 ↓
Binder Driver
 ↓
Stub
 ↓
Server
```

因此跨进程存在：

```text
序列化
上下文切换
内核参与
线程调度
数据拷贝
```

所以：

> 不应该把Binder调用放在高频、大数据、实时循环里。

---

# 三十二、Binder和共享内存怎么选择？

比如车机投屏：

```text
视频流
```

不适合：

```text
Binder传每一帧
```

应该：

```text
MediaCodec
 ↓
Surface
 ↓
SharedMemory / GraphicBuffer
```

而 Binder 用于：

```text
控制命令
状态通知
Surface句柄
```

所以：

```text
Binder = 控制面
SharedMemory/Socket = 数据面
```

这是很高级、非常适合车载的回答。

---

# 三十三、HIDL和AIDL怎么区别？

JD明确涉及：

```text
Framework
 ↓
HIDL/AIDL
 ↓
VHAL
```

### HIDL

Android早期Treble时代用于：

```text
Framework ↔ HAL
```

### AIDL HAL

Android新版本越来越推荐：

```text
AIDL
```

所以面试可以说：

> HIDL是Android Treble时代用于Framework和HAL标准化IPC的接口描述机制；AIDL原本主要用于应用/Framework Binder IPC，后来也支持稳定的系统/HAL接口。新Android版本中很多HAL逐步从HIDL迁移到AIDL。

---

# 三十四、这个岗位的线程 + 进程通信完整场景题

面试官：

> 现在车辆速度从CAN总线过来，最终怎么显示到仪表App？

你应该能够完整回答：

```text
CAN
 ↓
MCU
 ↓
BSP Driver
 ↓
Native/VHAL
 ↓
Vehicle HAL
 ↓
CarPropertyService
 ↓
Binder
 ↓
CarPropertyManager
 ↓
App
 ↓
Callback
 ↓
Handler/Coroutine
 ↓
Main Thread
 ↓
Custom View
 ↓
仪表盘
```

JD本身给出了类似的数据链路：

```text
CAN Frame
 ↓
DBC解码
 ↓
VHAL VehiclePropValue
 ↓
Binder
 ↓
CarPropertyManager.onChangeEvent()
 ↓
App UI
```

---

# 三十五、这个场景中哪里是进程通信？

```text
App
       │
       │ Binder
       ↓
CarService
       │
       │ Binder/HIDL/AIDL
       ↓
VHAL
```

---

# 三十六、哪里是线程通信？

比如：

```text
Binder线程
   ↓
Handler
   ↓
主线程
   ↓
UI
```

或者：

```text
VHAL callback
   ↓
Coroutine
   ↓
Dispatchers.Default
   ↓
计算
   ↓
Dispatchers.Main
   ↓
UI
```

所以：

> **进程通信解决“不同进程之间怎么传数据”，线程通信解决“同一个进程内部不同线程怎么协作”。**

这是非常重要的一句话。

---

# 三十七、Java并发面试题清单

针对这个岗位，我建议至少准备下面这些。

## ⭐⭐⭐⭐⭐ 必须掌握

### 1 `volatile`原理是什么？

### 2 `synchronized`原理？

### 3 `synchronized`锁升级了解吗？

### 4 `synchronized`和`ReentrantLock`区别？

### 5 CAS是什么？

### 6 AtomicInteger为什么线程安全？

```java
AtomicInteger count =
        new AtomicInteger();

count.incrementAndGet();
```

核心：

```text
CAS
```

---

### 7CAS有什么问题？

```text
ABA问题
 ↓
版本号解决
```

例如：

```text
A → B → A
```

线程看到：

```text
还是A
```

但是实际上发生过变化。

可以使用：

```java
AtomicStampedReference
```

---

# 三十八、ConcurrentHashMap

必须会。

面试：

> HashMap为什么线程不安全？

回答：

```text
并发put
结构破坏
数据丢失
```

ConcurrentHashMap：

```text
JDK8
 ↓
CAS
+
synchronized
+
Node
+
红黑树
```

JDK8不再采用JDK7那种：

```text
Segment数组
```

而是：

```text
Node[]
 ↓
CAS
 ↓
synchronized
```

---

# 三十九、HashMap为什么容量通常是2的幂？

经典题。

因为：

```java
index = (n - 1) & hash;
```

如果：

```text
n = 16
```

那么：

```text
n - 1 = 15
```

二进制：

```text
1111
```

可以快速定位：

```text
hash & 1111
```

比：

```text
hash % 16
```

更适合实现高效索引。

---

# 四十、Java线程状态

必须背：

```text
NEW
 ↓
RUNNABLE
 ↓
BLOCKED
 ↓
WAITING
 ↓
TIMED_WAITING
 ↓
TERMINATED
```

例如：

```java
thread.join();
```

调用线程：

```text
WAITING
```

---

# 四十一、sleep / wait / join

||sleep|wait|join|
|---|---|---|---|
|所属|Thread|Object|Thread|
|释放锁|❌|✅|根据实现等待|
|唤醒|时间到|notify|被等待线程结束|
|用途|延时|条件等待|等线程结束|

---

# 四十二、ThreadLocal

车载系统也可能问。

```java
ThreadLocal<UserContext> context =
        new ThreadLocal<>();
```

核心：

```text
Thread
 ↓
ThreadLocalMap
 ↓
ThreadLocal → Value
```

不是：

```text
ThreadLocal保存数据
```

而是：

> 数据实际保存在当前Thread对象关联的ThreadLocalMap中。

---

# 四十三、ThreadLocal为什么可能内存泄漏？

因为：

```text
Thread
 ↓
ThreadLocalMap
 ↓
Entry
 ↓
WeakReference<ThreadLocal>
 ↓
Value
```

ThreadLocal key被GC后：

```text
key = null
value还存在
```

如果线程长期存活，比如线程池：

```text
线程池线程
 ↓
ThreadLocal Value
 ↓
一直存在
```

所以需要：

```java
try {

    threadLocal.set(data);

} finally {

    threadLocal.remove();
}
```

---

# 四十四、Java线程池面试必问：Executors为什么不推荐？

例如：

```java
Executors.newFixedThreadPool()
```

内部可能使用：

```text
LinkedBlockingQueue
```

无界队列。

任务大量进入：

```text
Task
Task
Task
Task
Task
...
```

可能造成：

```text
内存压力
```

生产环境更推荐明确指定：

```java
ThreadPoolExecutor
```

---

# 四十五、Android线程池怎么选择？

可以这么回答：

```text
CPU密集型
    ↓
CPU核心数附近

IO密集型
    ↓
适当增加线程

大量短任务
    ↓
线程池

串行后台任务
    ↓
HandlerThread

UI更新
    ↓
Main Thread

定时任务
    ↓
ScheduledExecutor / WorkManager

复杂异步业务
    ↓
Coroutine
```

---

# 四十六、Kotlin Coroutine和Java线程是什么关系？

非常容易被问。

协程：

```text
Coroutine
≠
Thread
```

例如：

```kotlin
launch {
    val result = repository.getVehicleInfo()
}
```

协程本身不是线程。

最终还是：

```text
Coroutine
 ↓
Dispatcher
 ↓
Thread / ThreadPool
```

例如：

```kotlin
Dispatchers.IO
```

底层使用线程池执行阻塞IO任务。

所以：

> 协程解决的是异步任务的组织和挂起恢复问题，不是凭空创造CPU执行能力。

---

# 四十七、Flow和车载信号非常匹配

例如车速：

```kotlin
val speedFlow: Flow<Float>
```

可以：

```kotlin
speedFlow
    .distinctUntilChanged()
    .collectLatest { speed ->
        updateSpeed(speed)
    }
```

车辆信号：

```text
VHAL
 ↓
CarPropertyManager
 ↓
callback
 ↓
callbackFlow
 ↓
Flow
 ↓
UI
```

这会是一个很好的现代 Android 答案。

---

# 四十八、StateFlow和SharedFlow怎么选？

### StateFlow

表示：

```text
当前状态
```

例如：

```kotlin
data class VehicleState(
    val speed: Float,
    val temperature: Float,
    val gear: Gear
)
```

```kotlin
val vehicleState: StateFlow<VehicleState>
```

特点：

```text
始终有最新值
```

### SharedFlow

表示：

```text
事件
```

例如：

```text
车门打开
系统故障
ADAS预警
```

所以：

```text
StateFlow → 当前车辆状态

SharedFlow → 一次性车辆事件
```

---

# 四十九、车载系统特别适合MVI

例如：

```kotlin
data class VehicleUiState(
    val speed: Float = 0f,
    val gear: Gear = Gear.P,
    val temperature: Float = 24f
)
```

事件：

```kotlin
sealed interface VehicleIntent {

    data object Refresh : VehicleIntent

    data class SetTemperature(
        val value: Float
    ) : VehicleIntent
}
```

状态：

```text
CAN
 ↓
Repository
 ↓
ViewModel
 ↓
StateFlow
 ↓
UI
```

这和你之前研究的：

```text
MVVM
Repository
Flow
StateFlow
```

可以直接结合起来。

---

# 五十、这个岗位最值得准备的Java题库

我建议你按下面优先级准备。

## 第一档：★★★★★

```text
1. Thread生命周期
2. Handler/Looper/MessageQueue
3. HandlerThread
4. wait/notify
5. synchronized
6. volatile
7. CAS
8. Atomic
9. ReentrantLock
10. ThreadPoolExecutor
11. BlockingQueue
12. ConcurrentHashMap
13. Binder
14. AIDL
15. Binder线程池
16. Binder死亡
17. Parcel
18. IPC性能
19. Activity跨进程
20. Service跨进程
```

---

# 五十一、第二档：★★★★

```text
21. Messenger
22. Broadcast
23. ContentProvider
24. SharedMemory
25. Socket
26. HIDL
27. AIDL HAL
28. Java线程状态
29. ThreadLocal
30. CountDownLatch
31. CyclicBarrier
32. Semaphore
33. Future
34. CompletableFuture
35. Executor
36. ScheduledExecutor
37. HashMap
38. ConcurrentHashMap
39. ArrayList
40. CopyOnWriteArrayList
```

---

# 五十二、第三档：★★★★ Android Java

```text
41. Activity启动流程
42. Activity跨进程启动
43. AMS
44. WMS
45. PMS
46. SystemServer
47. Zygote
48. Binder ServiceManager
49. Application启动
50. Looper启动
51. Choreographer
52. ANR
53. OOM
54. Leak
55. GC
56. ClassLoader
57. DexClassLoader
58. PathClassLoader
59. Parcelable
60. Serializable
```

---

# 五十三、这个JD特别容易出现的综合题

## 题目1

> 为什么Android的Binder比普通Socket更适合系统服务？

准备：

```text
Binder
 ↓
内核驱动
 ↓
UID/PID
 ↓
对象引用
 ↓
Parcel
 ↓
线程池
```

---

## 题目2

> App调用CarPropertyManager.setProperty后发生了什么？

回答：

```text
App
 ↓
CarPropertyManager
 ↓
Binder Proxy
 ↓
CarPropertyService
 ↓
权限检查
 ↓
VehicleHal
 ↓
HAL
 ↓
CAN/MCU
```

---

## 题目3

> CAN信号回来后如何更新UI？

```text
CAN
 ↓
VHAL
 ↓
CarService
 ↓
Binder
 ↓
CarPropertyManager callback
 ↓
后台线程
 ↓
Handler/Coroutine
 ↓
Main Thread
 ↓
UI
```

---

## 题目4

> 如果CarService进程挂掉怎么办？

回答：

```text
linkToDeath()
 ↓
DeathRecipient
 ↓
发现Binder死亡
 ↓
重新连接CarService
 ↓
重新注册Callback
 ↓
恢复状态
```

---

## 题目5

> 为什么不能直接在Binder回调里面操作UI？

因为：

```text
Binder callback
```

执行线程不一定是：

```text
Main Thread
```

所以应该：

```java
mainHandler.post(() -> {

    textView.setText(...);

});
```

或者：

```kotlin
withContext(Dispatchers.Main) {
    updateUI()
}
```

---

# 五十四、面试官可能继续深挖

例如你回答：

> “我用Handler切到主线程。”

面试官马上可能问：

### Q1

> Handler为什么能切线程？

### Q2

> Handler内部怎么找到目标线程？

### Q3

> Looper和Thread什么关系？

### Q4

> MessageQueue什么时候阻塞？

### Q5

> epoll是什么？

### Q6

> Looper为什么不会造成CPU 100%？

### Q7

> MessageQueue.next()没有消息时干什么？

### Q8

> Handler内存泄漏怎么产生？

### Q9

> 为什么Handler可以跨线程发送消息？

### Q10

> HandlerThread有什么缺点？

这就是所谓：

```text
API
 ↓
源码
 ↓
系统原理
```

---

# 五十五、Binder也会这样追问

你回答：

> “我们项目使用AIDL。”

面试官：

```text
① AIDL是什么？

② AIDL生成什么代码？

③ Proxy是什么？

④ Stub是什么？

⑤ transact做了什么？

⑥ Binder Driver是什么？

⑦ Parcel是什么？

⑧ Binder线程池在哪里？

⑨ 为什么Binder不能传大数据？

⑩ Binder死亡怎么办？

⑪ oneway是什么？

⑫ Binder权限怎么控制？

⑬ Binder调用为什么可能ANR？
```

如果这些都能回答，基本就不是“只会写业务”的 Android 工程师了。

---

# 五十六、结合这个JD，我建议你的面试知识路线

```text
                 Java基础
                    │
          ┌─────────┴─────────┐
          ↓                   ↓
       Java并发              JVM
          │                   │
   volatile/synchronized    GC
   CAS/Atomic               ClassLoader
   Lock                    内存模型
   ThreadPool
          │
          ↓
     Android线程
          │
   Handler/Looper
   MessageQueue
   HandlerThread
          │
          ↓
     Android IPC
          │
   Binder / AIDL
          │
   ┌──────┼────────┐
   ↓      ↓        ↓
Messenger Broadcast Provider
          │
          ↓
       Framework
          │
   AMS/WMS/PMS
          │
       CarService
          │
       AdaptApi
          │
     HIDL / AIDL
          │
         VHAL
          │
      BSP / MCU
          │
       CAN/LIN
```

这条线非常贴合这份 JD，而不是泛泛准备 Android 面试。JD 中还特别涉及高通、瑞萨、恩智浦的平台适配，以及 `CarService/AdaptApi` 对不同平台差异进行抽象。

---

# 五十七、最后给你一套“面试官连续追问模拟”

如果我是这个岗位的面试官，我很可能这样问你：

### 第一轮：Java

> 1. Java线程有几种状态？
>     
> 2. synchronized原理？
>     
> 3. volatile原理？
>     
> 4. volatile为什么不能保证i++？
>     
> 5. CAS是什么？
>     
> 6. ABA怎么解决？
>     
> 7. ReentrantLock和synchronized区别？
>     
> 8. ThreadPoolExecutor工作流程？
>     
> 9. 核心线程和最大线程什么时候创建？
>     
> 10. BlockingQueue有哪些实现？
>     

### 第二轮：Android线程

> 11. Handler原理？
>     
> 12. Looper为什么不会死循环导致CPU高？
>     
> 13. MessageQueue如何阻塞？
>     
> 14. HandlerThread原理？
>     
> 15. Handler内存泄漏？
>     
> 16. 子线程怎么创建Handler？
>     
> 17. Handler和线程池区别？
>     

### 第三轮：IPC

> 18. Android有哪些IPC？
>     
> 19. Binder原理？
>     
> 20. AIDL原理？
>     
> 21. Stub和Proxy是什么？
>     
> 22. transact发生了什么？
>     
> 23. Binder线程池？
>     
> 24. Binder死亡怎么处理？
>     
> 25. Binder为什么不能传大Bitmap？
>     
> 26. Messenger和AIDL区别？
>     
> 27. ContentProvider为什么也是IPC？
>     

### 第四轮：车载

> 28. CAN信号怎么到Android App？
>     
> 29. VHAL是什么？
>     
> 30. CarService是什么？
>     
> 31. CarPropertyManager怎么实现跨进程？
>     
> 32. AdaptApi为什么需要？
>     
> 33. 高通、瑞萨、NXP怎么做平台适配？
>     
> 34. 多屏怎么实现？
>     
> 35. HUD和普通Display有什么区别？
>     
> 36. T-Box怎么和Android通信？
>     
> 37. HiCar/Carlink/FlymeLink技术上有什么区别？
>     

### 第五轮：性能

> 38. Android启动慢怎么排查？
>     
> 39. Perfetto怎么看？
>     
> 40. dumpsys meminfo怎么看？
>     
> 41. ANR怎么分析？
>     
> 42. OOM怎么分析？
>     
> 43. Binder调用为什么可能导致卡顿？
>     

JD本身就明确要求启动、内存、多任务调度以及 `logcat/dumpsys/top/procrank/dmesg/getprop` 等系统排查能力，所以这部分也不能只准备理论。

---

## 你现在最应该重点突破的 5 个源码

如果时间有限，不要平均用力，优先把这几个真正搞懂：

```text
① Handler / Looper / MessageQueue
        ↓
② ThreadPoolExecutor
        ↓
③ Binder / AIDL
        ↓
④ CarPropertyManager / CarPropertyService
        ↓
⑤ VHAL / VehicleHal
```

尤其是 **③ + ④ + ⑤**。

因为这份 JD 和普通 Android 岗最大的区别，就是它不是停留在：

```text
Activity
Fragment
RecyclerView
MVVM
```

而是要求你能够把：

```text
Java线程
   ↓
Android线程
   ↓
Binder IPC
   ↓
CarService
   ↓
AIDL/HIDL
   ↓
VHAL
   ↓
MCU/CAN
```

整个链路讲清楚。JD 中的 IVI、车载信号、平台适配、T-Box、V2X、多屏以及底层团队协作，本质上都在考这条链路。

**如果按这个岗位准备，我建议下一步直接做成一份“车载 Android 中高级面试题库”**：按 **Java并发 → Handler源码 → Binder源码 → AIDL实战 → CarService/VHAL → 多屏/HUD → T-Box/V2X → ANR/性能 → 30道模拟面试题**，每道题给出**标准回答 + 源码流程 + 具体代码 + 面试官追问 + 3~5年Android工程师回答深度**。这样会比单独背知识点更适合这份 9–14K 的 JD。