# Kotlin 协程原理及使用（Android 面试高频）

协程（Coroutine）本质上是 **一种轻量级线程解决方案**，通过 **挂起（Suspend）+ 状态机（State Machine）+ Continuation（续体）** 实现异步编程。

相比 Thread：

|对比项|Thread|Coroutine|
|---|---|---|
|创建成本|高|极低|
|切换成本|内核态切换|用户态切换|
|数量|几千个就卡|几十万级|
|阻塞|阻塞线程|挂起协程|
|管理|复杂|结构化并发|

---

# 一、为什么需要协程

传统异步代码：

```java
new Thread(() -> {
    String result = request();

    runOnUiThread(() -> {
        tv.setText(result);
    });
}).start();
```

回调地狱：

```java
api.login(user, result -> {
    api.getInfo(result.id, info -> {
        api.getOrder(info.id, order -> {
            ...
        });
    });
});
```

协程：

```kotlin
lifecycleScope.launch {
    val user = login()
    val info = getInfo(user.id)
    val order = getOrder(info.id)
}
```

看起来像同步代码：

```kotlin
val result = request()
```

实际上是异步执行。

---

# 二、协程核心组件

## 1 CoroutineScope

协程作用域

负责：

```text
管理协程生命周期
管理取消
管理异常
```

例如：

```kotlin
lifecycleScope
viewModelScope
GlobalScope
```

创建：

```kotlin
val scope = CoroutineScope(Dispatchers.Main)
```

启动：

```kotlin
scope.launch {

}
```

---

## 2 Job

每个协程都有一个 Job。

作用：

```text
启动
取消
状态管理
```

例如：

```kotlin
val job = launch {

}
```

取消：

```kotlin
job.cancel()
```

查看状态：

```kotlin
job.isActive

job.isCancelled

job.isCompleted
```

---

## 3 Dispatcher

调度器

决定：

```text
协程在哪个线程执行
```

---

### Dispatchers.Main

主线程

```kotlin
launch(Dispatchers.Main) {

}
```

Android UI 更新：

```kotlin
withContext(Dispatchers.Main) {
    tv.text = "success"
}
```

---

### Dispatchers.IO

IO线程池

适合：

```text
网络请求
数据库
文件操作
```

```kotlin
launch(Dispatchers.IO) {
    request()
}
```

---

### Dispatchers.Default

CPU密集型

适合：

```text
排序
加密
图像处理
```

```kotlin
launch(Dispatchers.Default) {

}
```

---

### Dispatchers.Unconfined

不指定线程

很少使用

```kotlin
launch(Dispatchers.Unconfined)
```

---

# 三、launch 和 async

## launch

无返回值

```kotlin
launch {
    println("hello")
}
```

返回：

```kotlin
Job
```

---

## async

有返回值

```kotlin
val deferred = async {
    request()
}
```

获取：

```kotlin
val result = deferred.await()
```

返回：

```kotlin
Deferred
```

本质：

```kotlin
Deferred = Job + Result
```

---

# 四、withContext

线程切换神器

例如：

```kotlin
launch {

    val result = withContext(Dispatchers.IO) {
        request()
    }

    tv.text = result
}
```

执行流程：

```text
Main
 ↓
IO
 ↓
Main
```

无需手动切换线程。

---

# 五、suspend 关键字

协程核心

```kotlin
suspend fun requestData(): String {
    delay(1000)
    return "success"
}
```

调用：

```kotlin
launch {
    val result = requestData()
}
```

---

### suspend 不等于异步

很多人误解：

```kotlin
suspend
```

不是：

```text
新线程
异步
```

而是：

```text
可挂起
可恢复
```

---

# 六、delay 为什么不会阻塞线程

普通：

```kotlin
Thread.sleep(1000)
```

线程阻塞。

协程：

```kotlin
delay(1000)
```

线程立即释放。

1秒后恢复。

---

# 七、协程原理（面试重点）

假设：

```kotlin
suspend fun test() {
    delay(1000)
    println("hello")
}
```

编译后近似：

```java
Object test(Continuation cont){
    switch(label){

        case 0:
            label = 1;
            delay(1000, cont);
            return COROUTINE_SUSPENDED;

        case 1:
            System.out.println("hello");
            return Unit;
    }
}
```

---

## 状态机(State Machine)

编译器自动生成：

```text
状态0
 ↓
delay挂起
 ↓
状态1
 ↓
继续执行
```

因此：

```kotlin
suspend
```

最终变成：

```text
状态机 + Continuation
```

---

# 八、Continuation（续体）

协程底层核心。

定义：

```kotlin
interface Continuation<T> {
    val context: CoroutineContext

    fun resumeWith(result: Result<T>)
}
```

作用：

```text
记录当前执行位置
保存局部变量
恢复执行
```

例如：

```kotlin
suspend fun getData()
```

编译后：

```java
Object getData(
    Continuation continuation
)
```

实际上所有 suspend 函数都会多一个：

```kotlin
Continuation
```

参数。

---

# 九、协程取消机制

```kotlin
val job = launch {

    while (isActive) {

    }
}
```

取消：

```kotlin
job.cancel()
```

---

可取消挂起函数：

```kotlin
delay()
yield()
withContext()
```

都会检查：

```kotlin
isActive
```

---

# 十、结构化并发（重点）

错误写法：

```kotlin
GlobalScope.launch {

}
```

可能导致：

```text
Activity销毁
协程还在运行
内存泄漏
```

---

正确：

```kotlin
viewModelScope.launch {

}
```

或：

```kotlin
lifecycleScope.launch {

}
```

生命周期自动管理。

---

# 十一、Android 实战

## ViewModel

```kotlin
class NewsViewModel : ViewModel() {

    fun loadNews() {

        viewModelScope.launch {

            val list = withContext(
                Dispatchers.IO
            ) {
                repository.getNews()
            }

            _liveData.value = list
        }
    }
}
```

---

## Fragment

```kotlin
viewLifecycleOwner.lifecycleScope.launch {

    viewModel.news.collect {

        adapter.submitList(it)
    }
}
```

---

# 十二、Flow 配合协程

替代：

```kotlin
LiveData
RxJava
```

定义：

```kotlin
fun getNews() = flow {

    emit("1")

    emit("2")

    emit("3")
}
```

收集：

```kotlin
launch {

    getNews().collect {

        println(it)
    }
}
```

---

# 十三、面试必问：协程和 RxJava 区别

|项目|Coroutine|RxJava|
|---|---|---|
|学习成本|低|高|
|代码量|少|多|
|链式操作|一般|强|
|背压|Flow支持|原生支持|
|生命周期|强|需手动|
|Android官方|推荐|非官方|

---

# 十四、协程关键参数总结

### launch

```kotlin
launch(
    context = Dispatchers.IO,
    start = CoroutineStart.DEFAULT
)
```

参数：

```kotlin
context
协程上下文

start
启动模式

parent
父Job

exceptionHandler
异常处理器
```

---

### CoroutineStart

```kotlin
DEFAULT
```

立即调度

```kotlin
LAZY
```

调用：

```kotlin
job.start()
```

才执行

```kotlin
ATOMIC
```

启动前不可取消

```kotlin
UNDISPATCHED
```

立即在当前线程执行

---

### async

```kotlin
async(
    Dispatchers.IO
) {
    request()
}
```

返回：

```kotlin
Deferred<T>
```

获取：

```kotlin
await()
```

---

# Android 高级面试源码级回答

面试官问：

**Kotlin 协程为什么比线程轻量？**

回答：

> 协程并不是操作系统线程，而是运行在线程上的任务。挂起时不会触发内核态线程切换，而是编译器将 suspend 函数转换为状态机，通过 Continuation 保存执行现场。恢复时从状态机对应位置继续执行，因此切换成本远低于线程切换。

面试官问：

**suspend 的底层原理是什么？**

回答：

> suspend 函数编译后会额外接收一个 Continuation 参数。编译器把函数拆分成多个状态，并生成状态机。执行到挂起点时返回 COROUTINE_SUSPENDED，恢复时通过 Continuation.resumeWith() 回到之前状态继续执行。

面试官问：

**协程和线程是什么关系？**

回答：

> 协程运行在线程上，一个线程可以运行多个协程。协程负责任务调度，线程负责真正执行代码。Dispatchers 决定协程最终在哪个线程池执行。

如果你准备 Android 高级工程师面试，我还可以继续给你整理：

- 协程源码级分析（Job、Scope、Dispatcher实现）
    
- Flow源码解析
    
- Retrofit + 协程实现原理
    
- ViewModelScope源码实现
    
- Kotlin协程面试题100问（含答案）