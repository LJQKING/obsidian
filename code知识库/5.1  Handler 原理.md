# Android Handler 原理及源码解读

Handler 是 Android 中最经典的线程通信机制之一，主要用于：

- **子线程切换到主线程更新UI**
    
- **线程间消息传递**
    
- **延时任务执行**
    
- **消息队列管理**
    

Android 四大核心组件（Activity、Service、BroadcastReceiver、ContentProvider）底层都大量使用了 Handler 机制。

---

# 一、Handler核心组成

Handler机制主要由4个核心类组成：

```java
Handler
Message
MessageQueue
Looper
```

整体架构：

```text
Thread
   │
   ▼
Looper
   │
   ▼
MessageQueue
   ▲
   │
Handler
   │
   ▼
Message
```

工作流程：

```text
发送消息
   │
   ▼
Handler.sendMessage()
   │
   ▼
MessageQueue.enqueueMessage()
   │
   ▼
Looper.loop()
   │
   ▼
取出Message
   │
   ▼
Handler.dispatchMessage()
   │
   ▼
handleMessage()
```

---

# 二、Handler工作原理

例如：

```java
Handler handler = new Handler(Looper.getMainLooper()) {
    @Override
    public void handleMessage(Message msg) {
        tv.setText("更新成功");
    }
};

new Thread(() -> {
    handler.sendEmptyMessage(1);
}).start();
```

执行流程：

### 第一步

子线程发送消息

```java
handler.sendEmptyMessage(1);
```

↓

### 第二步

创建Message

```java
Message msg = Message.obtain();
msg.what = 1;
```

↓

### 第三步

加入消息队列

```java
MessageQueue.enqueueMessage()
```

↓

### 第四步

Looper死循环

```java
Looper.loop()
```

不断从MessageQueue取消息

↓

### 第五步

回调Handler

```java
dispatchMessage(msg)
```

↓

### 第六步

执行

```java
handleMessage(msg)
```

---

# 三、Looper源码解析

## 主线程Looper创建

应用启动：

```java
ActivityThread.main()
```

源码：

```java
public static void main(String[] args) {

    Looper.prepareMainLooper();

    ActivityThread thread = new ActivityThread();

    thread.attach(false);

    Looper.loop();
}
```

发现：

```java
主线程启动时自动创建Looper
```

---

## Looper.prepare()

源码：

```java
public static void prepare() {

    if (sThreadLocal.get() != null) {
        throw new RuntimeException(
            "Only one Looper may be created per thread");
    }

    sThreadLocal.set(new Looper());
}
```

重点：

```java
private static final ThreadLocal<Looper> sThreadLocal
        = new ThreadLocal<>();
```

ThreadLocal保存Looper。

结构：

```text
线程A ---> LooperA

线程B ---> LooperB

线程C ---> LooperC
```

每个线程只能有一个Looper。

---

## Looper构造函数

```java
private Looper() {
    mQueue = new MessageQueue();
}
```

创建：

```java
MessageQueue
```

---

# 四、Looper.loop()源码

核心源码：

```java
public static void loop() {

    final Looper me = myLooper();

    final MessageQueue queue = me.mQueue;

    for (;;) {

        Message msg = queue.next();

        if (msg == null) {
            return;
        }

        msg.target.dispatchMessage(msg);

        msg.recycleUnchecked();
    }
}
```

等价于：

```java
while(true){

    Message msg = queue.next();

    dispatchMessage(msg);
}
```

这就是：

```java
消息循环
```

---

# 五、MessageQueue源码

很多人以为：

```java
MessageQueue = Queue
```

其实不是。

源码：

```java
Message mMessages;
```

MessageQueue内部：

```text
msg1
 ↓
msg2
 ↓
msg3
 ↓
msg4
```

实际上是：

```java
单向链表
```

不是队列容器。

原因：

```java
插入效率高
```

---

## enqueueMessage()

插入消息

源码简化：

```java
boolean enqueueMessage(Message msg,long when){
    
    Message p = mMessages;

    if (p == null || when == 0 || when < p.when) {

        msg.next = p;

        mMessages = msg;

    } else {

        while (p.next != null &&
                p.next.when <= when) {
            p = p.next;
        }

        msg.next = p.next;
        p.next = msg;
    }

    return true;
}
```

特点：

```text
按照执行时间排序
```

例如：

```text
msg4  4s
msg1  1s
msg3  3s
msg2  2s
```

插入后：

```text
msg1
 ↓
msg2
 ↓
msg3
 ↓
msg4
```

---

# 六、Message源码

## Message.obtain()

推荐：

```java
Message.obtain();
```

而不是：

```java
new Message();
```

源码：

```java
public static Message obtain() {

    synchronized (sPoolSync) {

        if (sPool != null) {

            Message m = sPool;

            sPool = m.next;

            return m;
        }
    }

    return new Message();
}
```

使用：

```java
消息池
```

结构：

```text
Message Pool

msg
 ↓
msg
 ↓
msg
```

避免频繁GC。

---

# 七、Handler源码

## Handler构造

```java
public Handler() {

    mLooper = Looper.myLooper();

    mQueue = mLooper.mQueue;
}
```

获取：

```java
当前线程Looper
```

例如：

```java
Handler handler = new Handler();
```

等价：

```java
Handler handler =
      new Handler(Looper.myLooper());
```

---

## sendMessage()

源码：

```java
public final boolean sendMessage(Message msg) {

    return sendMessageDelayed(msg,0);
}
```

↓

```java
sendMessageDelayed()
```

↓

```java
sendMessageAtTime()
```

↓

```java
enqueueMessage()
```

最终：

```java
mQueue.enqueueMessage(msg,time);
```

加入MessageQueue。

---

# 八、dispatchMessage源码

```java
public void dispatchMessage(Message msg) {

    if (msg.callback != null) {

        handleCallback(msg);

    } else {

        if (mCallback != null) {

            if (mCallback.handleMessage(msg)) {
                return;
            }
        }

        handleMessage(msg);
    }
}
```

执行优先级：

```text
Runnable
    ↓
Callback
    ↓
handleMessage
```

---

示例：

## 方式1

```java
handler.post(runnable);
```

执行：

```java
Runnable.run()
```

---

## 方式2

```java
new Handler(callback);
```

执行：

```java
callback.handleMessage()
```

---

## 方式3

```java
override handleMessage()
```

执行：

```java
handleMessage()
```

---

# 九、为什么主线程可以更新UI？

因为：

```java
ViewRootImpl.checkThread()
```

源码：

```java
void checkThread() {

    if (mThread != Thread.currentThread()) {

        throw new CalledFromWrongThreadException(
            "Only the original thread that created a view hierarchy can touch its views."
        );
    }
}
```

View创建时：

```java
mThread = Thread.currentThread();
```

记录的是：

```java
主线程
```

所以：

```java
只能主线程更新UI
```

---

# 十、Handler导致内存泄漏原因

错误写法：

```java
public class MainActivity extends Activity {

    Handler handler = new Handler(){

        @Override
        public void handleMessage(Message msg){

        }
    };
}
```

关系：

```text
Handler
   │
   ▼
匿名内部类
   │
   ▼
持有Activity
```

如果：

```java
handler.postDelayed(...,600000);
```

消息还在队列中。

```text
MessageQueue
   │
   ▼
Message
   │
   ▼
Handler
   │
   ▼
Activity
```

Activity无法回收。

---

解决方案：

```java
static class MyHandler extends Handler{

    WeakReference<Activity> ref;

    public MyHandler(Activity act){
        ref = new WeakReference<>(act);
    }
}
```

同时：

```java
@Override
protected void onDestroy() {
    super.onDestroy();

    handler.removeCallbacksAndMessages(null);
}
```

---

# 十一、面试高频问题

### 1.Handler为什么不会阻塞主线程？

因为：

```java
MessageQueue.next()
```

底层使用：

```java
nativePollOnce()
```

Linux epoll机制。

没有消息时：

```java
线程休眠
```

有消息时：

```java
立刻唤醒
```

不是死循环空转，因此不会导致CPU 100%。

---

### 2.一个线程可以有几个Looper？

```java
一个线程只能有一个Looper
```

源码：

```java
if (sThreadLocal.get() != null) {
    throw new RuntimeException();
}
```

---

### 3.一个Looper可以对应几个Handler？

```java
多个
```

例如：

```java
Handler h1
Handler h2
Handler h3
```

都可以共享：

```java
Looper.getMainLooper()
```

---

### 4.MessageQueue是真正的队列吗？

不是。

底层：

```java
单向链表
```

按时间排序。

---

### 5.post()和sendMessage()区别？

```java
post(Runnable)
```

本质：

```java
Runnable封装进Message
```

源码：

```java
msg.callback = runnable;
```

最终仍然进入：

```java
MessageQueue
```

---

# 一张图总结 Handler 机制

```text
                 Handler
                     │
        sendMessage/post()
                     │
                     ▼
                Message
                     │
                     ▼
             MessageQueue
             (单向链表)
                     │
                     ▼
              Looper.loop()
                无限循环
                     │
                     ▼
          dispatchMessage()
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
      Runnable    Callback   handleMessage
```

面试中如果被问到 **“Handler源码原理”**，推荐按以下顺序回答：

**ThreadLocal → Looper → MessageQueue（单链表）→ Handler发送消息 → Looper.loop死循环取消息 → dispatchMessage分发 → handleMessage执行 → Message复用池 → 内存泄漏与解决方案 → IdleHandler、同步屏障、Choreographer源码扩展。**

**==Handler 延迟消息是怎么实现的？==**
1. sendMessageDelayed 最终调用 sendMessageAtTime
2. Message 记录未来执行时间 when
3. MessageQueue 按 when 排序存储消息
4. Looper.loop() 不断调用 MessageQueue.next()
5. 如果未到执行时间，调用 nativePollOnce()
6. nativePollOnce 底层使用 Linux epoll_wait 阻塞等待
7. 时间到后唤醒线程，取出消息并执行
8. 因此 Handler 延迟消息本质是：
   MessageQueue + 时间戳排序 + epoll定时唤醒