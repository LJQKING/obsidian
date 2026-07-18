# Kotlin 协程源码级分析（Job、Scope、Dispatcher）

如果是 Android 高级工程师面试，面试官问：

> Kotlin 协程底层怎么实现？
> 
> Job、Scope、Dispatcher 分别干什么？
> 
> launch 到底发生了什么？

回答必须深入到：

```text
CoroutineScope
 ↓
launch
 ↓
CoroutineContext
 ↓
Dispatcher
 ↓
Continuation
 ↓
StateMachine
 ↓
ThreadPool
```

---

# 一、协程整体架构

一个最简单的协程：

```kotlin
viewModelScope.launch {
    delay(1000)
    println("Hello")
}
```

实际流程：

```text
CoroutineScope
      │
      ▼
 launch()
      │
      ▼
 创建Job
      │
      ▼
 创建Continuation
      │
      ▼
 Dispatcher调度
      │
      ▼
 ThreadPool执行
      │
      ▼
 suspend挂起
      │
      ▼
 StateMachine保存状态
      │
      ▼
 resume恢复
      │
      ▼
 继续执行
```

---

# 二、CoroutineContext源码

所有协程最终都有：

```kotlin
CoroutineContext
```

源码：

```kotlin
public interface CoroutineContext
```

本质：

```text
一个Map
```

存储：

```text
Job
Dispatcher
CoroutineName
ExceptionHandler
```

例如：

```kotlin
launch(
    Dispatchers.IO +
    CoroutineName("Net")
)
```

实际上：

```text
CoroutineContext
 ├─ Job
 ├─ Dispatcher
 ├─ Name
 └─ ExceptionHandler
```

---

源码：

```kotlin
operator fun plus(
    context: CoroutineContext
): CoroutineContext
```

因此：

```kotlin
Dispatchers.IO + Job()
```

成立。

---

# 三、CoroutineScope源码

定义：

```kotlin
public interface CoroutineScope {
    public val coroutineContext: CoroutineContext
}
```

就一个属性：

```kotlin
coroutineContext
```

例如：

```kotlin
val scope =
    CoroutineScope(
        Dispatchers.Main
    )
```

源码：

```kotlin
public fun CoroutineScope(
    context: CoroutineContext
): CoroutineScope
```

最终：

```kotlin
ContextScope(context)
```

---

本质：

```text
CoroutineScope
=
CoroutineContext容器
```

作用：

```text
管理生命周期
管理取消
管理异常
```

---

# 四、launch源码分析

调用：

```kotlin
scope.launch {

}
```

源码：

```kotlin
public fun CoroutineScope.launch(
    context: CoroutineContext,
    start: CoroutineStart,
    block: suspend CoroutineScope.() -> Unit
): Job
```

返回：

```kotlin
Job
```

---

内部：

```kotlin
val coroutine =
    StandaloneCoroutine(...)
```

源码：

```kotlin
StandaloneCoroutine
    : AbstractCoroutine<Unit>()
```

继承：

```text
JobSupport
   ↑
AbstractCoroutine
   ↑
StandaloneCoroutine
```

---

启动：

```kotlin
coroutine.start(
    start,
    coroutine,
    block
)
```

最终：

```text
创建协程对象
创建Job
创建Continuation
启动Dispatcher
```

---

# 五、Job源码分析

面试必问：

> Job是什么？

---

定义：

```kotlin
public interface Job :
    CoroutineContext.Element
```

说明：

```text
Job本身就是Context元素
```

放进：

```kotlin
CoroutineContext
```

中。

---

状态机：

```text
NEW
 ↓
ACTIVE
 ↓
COMPLETING
 ↓
COMPLETED
```

取消：

```text
ACTIVE
 ↓
CANCELLING
 ↓
CANCELLED
```

---

源码：

```kotlin
JobSupport
```

维护：

```kotlin
private var _state
```

状态：

```kotlin
Incomplete
Finishing
Completed
Cancelled
```

---

取消：

```kotlin
job.cancel()
```

源码：

```kotlin
cancelInternal()
```

最终：

```kotlin
makeCancelling()
```

状态变：

```text
ACTIVE
 ↓
CANCELLING
 ↓
CANCELLED
```

---

# 六、父子协程实现

例如：

```kotlin
launch {

    launch {

    }

    launch {

    }
}
```

结构：

```text
Parent Job
   │
 ├── Child1
 │
 └── Child2
```

源码：

```kotlin
ChildHandleNode
```

维护：

```kotlin
LockFreeLinkedList
```

---

父取消：

```kotlin
parent.cancel()
```

传播：

```text
Parent
 ↓
Child1
 ↓
Child2
```

全部取消。

---

# 七、Dispatcher源码分析

面试最爱问：

> Dispatchers.IO和Default有什么区别？

---

Dispatcher定义：

```kotlin
CoroutineDispatcher
```

继承：

```kotlin
AbstractCoroutineContextElement
```

也是：

```text
CoroutineContext元素
```

---

核心方法：

```kotlin
dispatch(
    CoroutineContext context,
    Runnable block
)
```

作用：

```text
把任务扔给线程池
```

---

# 八、Dispatchers.Main

Android：

```kotlin
Dispatchers.Main
```

源码：

```kotlin
HandlerContext
```

内部：

```kotlin
Handler
```

执行：

```kotlin
handler.post(runnable)
```

本质：

```text
Main Dispatcher
=
Handler
```

---

源码链：

```text
Dispatchers.Main

↓

HandlerDispatcher

↓

HandlerContext

↓

Handler.post()
```

---

# 九、Dispatchers.IO源码

定义：

```kotlin
Dispatchers.IO
```

源码：

```kotlin
DefaultIoScheduler
```

内部：

```kotlin
CoroutineScheduler
```

线程数：

```text
64
或者
CPU核心数
```

取较大值。

源码：

```kotlin
kotlinx.coroutines.io.parallelism
```

---

执行：

```kotlin
dispatch()
```

↓

```kotlin
CoroutineScheduler
```

↓

```kotlin
Worker Thread
```

---

适合：

```text
网络请求
数据库
文件
```

---

# 十、Dispatchers.Default源码

定义：

```kotlin
Dispatchers.Default
```

源码：

```kotlin
DefaultScheduler
```

内部：

```kotlin
CoroutineScheduler
```

线程数：

```kotlin
CPU核心数
```

例如：

```text
8核手机
=
8线程
```

---

适合：

```text
排序
压缩
解码
加密
```

CPU密集型任务。

---

# 十一、CoroutineScheduler源码

协程线程池核心。

源码：

```kotlin
CoroutineScheduler
```

类似：

```text
ThreadPoolExecutor
```

但更强。

---

内部：

```text
GlobalQueue
LocalQueue
Worker
```

结构：

```text
GlobalQueue
      │
      ▼
 Worker1
 Worker2
 Worker3
```

---

支持：

```text
Work Stealing
```

工作窃取。

例如：

```text
Worker1空闲

↓

偷Worker2任务
```

提升CPU利用率。

---

# 十二、Continuation源码

定义：

```kotlin
interface Continuation<T>
```

核心：

```kotlin
fun resumeWith(
    result: Result<T>
)
```

作用：

```text
恢复协程
```

---

例如：

```kotlin
delay(1000)
```

执行：

```text
保存Continuation
 ↓
挂起
 ↓
1秒后
 ↓
resumeWith()
 ↓
继续执行
```

---

# 十三、delay源码

面试常问：

> delay为什么不卡线程？

---

源码：

```kotlin
suspend fun delay(
    timeMillis: Long
)
```

内部：

```kotlin
suspendCancellableCoroutine {
    cont ->
}
```

拿到：

```kotlin
Continuation
```

---

然后：

```kotlin
scheduleResumeAfterDelay()
```

放入定时器。

---

立即返回：

```kotlin
COROUTINE_SUSPENDED
```

线程释放。

---

时间到：

```kotlin
continuation.resume()
```

恢复执行。

---

# 十四、suspend源码真相

代码：

```kotlin
suspend fun test() {
    delay(1000)
    println("A")
}
```

编译后近似：

```java
Object test(
   Continuation cont
){
   switch(label){

      case 0:

         label = 1;

         delay(
             1000,
             cont
         );

         return COROUTINE_SUSPENDED;

      case 1:

         println("A");

         return Unit;
   }
}
```

---

本质：

```text
suspend
=
Continuation
+
StateMachine
```

---

# 十五、ViewModelScope源码

定义：

```kotlin
viewModelScope
```

源码：

```kotlin
CloseableCoroutineScope(
    SupervisorJob() +
    Dispatchers.Main.immediate
)
```

即：

```kotlin
SupervisorJob()
+
Main
```

---

ViewModel销毁：

```kotlin
onCleared()
```

调用：

```kotlin
scope.cancel()
```

所有子协程取消。

---

# 十六、面试官最喜欢的源码级回答

### 问：launch底层做了什么？

答：

> launch 会创建 StandaloneCoroutine 对象，该对象继承 AbstractCoroutine 和 JobSupport。随后创建 Continuation，并通过 Dispatcher 将任务提交到对应线程池执行。

---

### 问：Dispatcher本质是什么？

答：

> Dispatcher 本质是 CoroutineDispatcher，其核心方法 dispatch()。作用是把协程任务封装成 Runnable 投递到 Handler 或 CoroutineScheduler 中执行。

---

### 问：delay为什么不会阻塞线程？

答：

> delay 底层基于 suspendCancellableCoroutine 获取 Continuation，并通过定时器注册恢复逻辑。调用后立即返回 COROUTINE_SUSPENDED，线程被释放，到时间后通过 continuation.resume() 恢复执行。

---

### 问：协程底层核心是什么？

答：

> Kotlin 编译器会把 suspend 函数转换成 CPS（Continuation Passing Style）形式，每个 suspend 函数都会多一个 Continuation 参数，同时生成状态机。协程挂起时保存状态和局部变量，恢复时通过 Continuation.resumeWith() 回到对应状态继续执行。

---

## 一张源码架构图记住

```text
CoroutineScope
       │
       ▼
CoroutineContext
       │
 ┌─────┼────────┐
 ▼     ▼        ▼
Job Dispatcher ExceptionHandler
 │
 ▼
launch()
 │
 ▼
StandaloneCoroutine
 │
 ▼
Continuation
 │
 ▼
StateMachine
 │
 ▼
Dispatcher.dispatch()
 │
 ▼
CoroutineScheduler
 │
 ▼
Worker Thread
 │
 ▼
执行协程代码
```

这是 Android 高级工程师（15K~40K+）面试中关于 Kotlin 协程源码的核心知识体系。