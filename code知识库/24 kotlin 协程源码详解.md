
- [[5.1 Handler 源码级分析及原理.md]]  Handler消息机制
- [[0  Kotlin  函数｜MVVM｜Jetpack｜协程｜Retrofit + OkHttp.md]]  协程与Flow
- [[1.6 车载内Service和ContentProvider，BroadcastReceiver，AIDL.md]]  车载智能座舱
- [[24.1.1  Coroutine（Kotlin 协程）.md]]  协程与Flow
- [[5.0 防内存泄漏的 Handler.md]]  Handler消息机制
Kotlin 协程（Coroutines）在 JVM 层面的实现并没有引入任何操作系统级别的神秘魔法，其核心原理可以概括为：**CPS（Continuation Passing Style，续体传递风格）转换 + 状态机（State Machine）**。

Kotlin 编译器在编译期会将带有 `suspend` 关键字的代码切碎并重构为状态机，而官方库 `kotlinx.coroutines` 则负责调度、生命周期管理和取消协作。

## 一、 源码对照：编译器到底对你的代码做了什么？

当你写下一个 `suspend` 函数时，编译器会进行两项关键操作：

1. **参数追加**：在函数最后隐式追加一个 `Continuation` 类型的参数。
    
2. **状态机生成**：将函数体内的代码以 `suspend` 调用点为界限拆分为多个状态。
    

### 1. 原始 Kotlin 源代码

Kotlin

```
suspend fun getUserInfo(): String {
    println("Step 1: Start")
    val user = fetchUser() // 挂起点 1
    println("Step 2: Got user $user")
    val detail = fetchDetail(user) // 挂起点 2
    println("Step 3: Got detail $detail")
    return "$user - $detail"
}
```

### 2. 编译重构后的伪代码（等价 JVM 实现）

编译器会将上面的代码转换成类似如下的状态机与回调结构：

Java

```
// 编译器生成的 Continuation 接口
public interface Continuation<in T> {
    public val context: CoroutineContext
    public fun resumeWith(result: Result<T>)
}

// 转换后的 getUserInfo 方法（参数被添加了 Continuation）
public static final Object getUserInfo(Continuation $completion) {
    // 1. 实例化或复用状态机对象
    getUserInfoStateMachine sm;
    if ($completion instanceof getUserInfoStateMachine) {
        sm = (getUserInfoStateMachine) $completion;
    } else {
        sm = new getUserInfoStateMachine($completion) {
            @Override
            public final Object invokeSuspend(Object result) {
                this.result = result;
                this.label |= Integer.MIN_VALUE;
                return getUserInfo(this); // 重新触发状态机
            }
        };
    }

    Object $result = sm.result;
    Object COROUTINE_SUSPENDED = IntrinsicsKt.getCOROUTINE_SUSPENDED();

    // 2. 状态机路由控制
    switch (sm.label) {
        case 0:
            ResultKt.throwOnFailure($result);
            System.out.println("Step 1: Start");
            
            sm.label = 1; // 准备进入下一个状态
            Object user = fetchUser(sm); // 传入 sm 作为 continuation 回调
            if (user == COROUTINE_SUSPENDED) return COROUTINE_SUSPENDED; // 挂起，直接返回标志位
            break;

        case 1:
            ResultKt.throwOnFailure($result);
            Object userFromPrev = $result; // 从上次恢复的 result 中拿到返回值
            System.out.println("Step 2: Got user " + userFromPrev);
            
            sm.v1 = userFromPrev; // 局部变量暂存到 Continuation 实例中
            sm.label = 2;
            Object detail = fetchDetail(userFromPrev, sm);
            if (detail == COROUTINE_SUSPENDED) return COROUTINE_SUSPENDED;
            break;

        case 2:
            Object detailFromPrev = $result;
            Object userSaved = sm.v1; // 恢复局部变量
            System.out.println("Step 3: Got detail " + detailFromPrev);
            return userSaved + " - " + detailFromPrev;
    }
    return null;
}
```

### 核心原理分析

- **`COROUTINE_SUSPENDED` 挂起标志**：如果异步操作还没完成，`fetchUser` 会返回 `COROUTINE_SUSPENDED`，导致当前调用栈直接 `return` 退出（释放了当前线程，实现**不阻塞线程**）。
    
- **状态保存与恢复**：如果异步操作完成，回调会调用 `sm.resumeWith(result)`，从而再次触发 `invokeSuspend` / `getUserInfo`。由于 `sm.label` 已经被修改，`switch-case` 会精确跳转到下一个分支执行。
    
- **跨挂起点变量保存**：函数内的局部变量（如 `user`）会被上升为 Continuation 状态机类的成员变量（如 `sm.v1`），确保跨恢复时数据不丢失。
    

## 二、 官方库核心组件源码剖析

如果说编译器负责代码转换，那么 `kotlinx.coroutines` 库就是协程的“操作系统”。

```
CoroutineScope
   └── CoroutineContext (集合结构)
         ├── Job (生命周期控制)
         ├── CoroutineDispatcher (线程调度)
         └── CoroutineExceptionHandler (异常处理)
```

### 1. `Continuation`（续体）

源码定义：

Kotlin

```
public interface Continuation<in T> {
    public val context: CoroutineContext
    public fun resumeWith(result: Result<T>)
}
```

- **角色**：它是协程的“回调句柄”。当异步任务（如网络请求、定时器）完成时，通过调用 `resumeWith` 恢复协程状态机的运行。
    

### 2. `CoroutineContext`（上下文）

- **实现结构**：本质是一个类似 `Map` 的**持久化单向链表数据结构**。
    
- 每个元素称为 `Element`，它本身也是一个 `CoroutineContext`。通过重载 `+` 运算符（`plus` 方法）组合在一起：
    
    Kotlin
    
    ```
    val context = Dispatchers.IO + Job() + CoroutineName("MyCoroutine")
    ```
    

### 3. `Job` 与 `JobSupport`（生命周期与父子树）

- **树形生命周期**：每个新建的协程都会将父协程的 `Job` 绑定到自己的 `CoroutineContext` 中，形成一棵 **Job Tree**。
    
- **取消传播（Cancellation）**：当父 Job 被取消时，会递归取消所有子 Job；如果子 Job 发生未捕获异常，也会向上传播取消父 Job（除非使用 `SupervisorJob`）。
    

### 4. `CoroutineDispatcher`（线程调度器）

`CoroutineDispatcher` 实现了 `ContinuationInterceptor`（续体拦截器）接口：

Kotlin

```
public abstract class CoroutineDispatcher : ContinuationInterceptor {
    // 拦截原始 Continuation，包裹成 DispatchedContinuation
    override fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T> =
        DispatchedContinuation(this, continuation)

    // 子类实现具体的线程调度逻辑
    public abstract fun dispatch(context: CoroutineContext, block: Runnable)
}
```

- **工作机制**：在协程准备恢复（`resumeWith`）时，拦截器会先触发 `interceptContinuation`，返回一个 `DispatchedContinuation`。
    
- `DispatchedContinuation` 实现了 `Runnable`，它会将协程的恢复任务提交给对应的线程池（例如 `Dispatchers.IO` 底层的线程池或 `Dispatchers.Main` 的 Handler/Looper）。
    

## 三、 完整流程串联：`launch(Dispatchers.IO)` 的一生

1. **创建阶段**：`launch` 调用 `createCoroutineUnintercepted` 创建原始的 Continuation 状态机对象。
    
2. **拦截阶段**：调用 `intercepted()`，通过 `Dispatchers.IO` 将 Continuation 封装为 `DispatchedContinuation`。
    
3. **启动阶段**：调用 `resumeCancellableWith`，调度器将 Runnable 投递到线程池队列中。
    
4. **挂起阶段**：执行到 `suspend` 函数（如 `delay(1000)`），向 Handler 或定时器注册回调，并返回 `COROUTINE_SUSPENDED`，当前线程释放，回到线程池。
    
5. **恢复阶段**：定时器/回调触发，调用 `continuation.resumeWith()`，再次通过 `Dispatcher.dispatch()` 投递任务到目标线程，状态机 `label` 自增，进入 `switch` 的下一个分支。
    

您目前是在做协程的性能调优、排查协程泄漏问题，还是想进一步深入如 `Flow` 的背压机制或 `Channel` 的底层实现？