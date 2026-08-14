
## 第一部分：Java并发编程（★★★★★）

### Q1: synchronized和volatile的区别？volatile能否保证原子性？

#### 1️⃣ **原理解释**

- [[0  Kotlin  函数｜MVVM｜Jetpack｜协程｜Retrofit + OkHttp.md]]  协程与Flow
- [[23.0  Kotlin 类委托、属性委托、泛型.md]]  协程与Flow
- [[23.1  Kotlin Coroutines、Flow、StateFlow 深度指南.md]]  协程与Flow
- [[24.1.1  Coroutine（Kotlin 协程）.md]]  协程与Flow
- [[24.1.2   Kotlin协程的阻塞（Blocking）与挂起（Suspending）.md]]  协程与Flow
**synchronized**：互斥锁机制

```
本质：监视器(Monitor)锁
工作流程：
1. 进入synchronized块 → 获取对象的Monitor锁
2. 执行代码块 → 持有锁，其他线程阻塞
3. 退出synchronized块 → 释放Monitor锁
4. 等待中的线程竞争锁

内存语义：
- 获取锁时：将工作内存清空，从主存读取
- 释放锁时：将工作内存的数据写回主存
```

**volatile**：内存可见性标记

```
工作原理：
1. 编译时：添加内存屏障指令
2. 运行时：禁止指令重排序
3. 对volatile变量的修改：立即写入主存
4. 对volatile变量的读取：从主存读取最新值
```

#### 2️⃣ **为什么这样设计**

|特性|synchronized|volatile|
|---|---|---|
|**互斥**|✅ 提供互斥访问|❌ 无互斥能力|
|**可见性**|✅ 保证可见性|✅ 保证可见性|
|**原子性**|✅ 保证（通过互斥）|❌ **不保证**|
|**有序性**|✅ 禁止指令重排|✅ 禁止指令重排|
|**性能**|❌ 低（阻塞式）|✅ 高（无阻塞）|
|**适用场景**|复杂逻辑、多步操作|简单标志位、状态变量|

#### 3️⃣ **volatile不能保证原子性的证明**

```kotlin
// 问题案例：volatile无法保证原子性
class VolatileDemo {
    @Volatile
    var count = 0  // volatile标记
    
    fun increment() {
        count++  // ⚠️ 这不是原子操作！
    }
}

// 为什么？count++分解为三步：
// 1. 读取count值（从主存读取最新值）✅
// 2. count + 1（CPU计算）✅
// 3. 写回count（写入主存）✅
// 
// 问题：步骤1和步骤3之间有间隙
// 线程A: 读count=5 → 计算=6 → 写6
// 线程B: 读count=5 → 计算=6 → 写6 (与A重复！)
// 结果：count=6 而不是7
```

#### 4️⃣ **代码实现对比**

```kotlin
// ❌ 错误方案：volatile无法解决并发递增
@Volatile var count = 0
repeat(1000) {
    thread {
        repeat(1000) {
            count++  // 竞态条件！最后count < 1000000
        }
    }
}

// ✅ 方案1：使用synchronized
var count = 0
repeat(1000) {
    thread {
        repeat(1000) {
            synchronized(this) {
                count++  // 互斥保护，最终count = 1000000
            }
        }
    }
}

// ✅ 方案2：使用AtomicInteger（最优）
val count = AtomicInteger(0)
repeat(1000) {
    thread {
        repeat(1000) {
            count.incrementAndGet()  // CAS原子操作
        }
    }
}

// ✅ 方案3：使用ReentrantLock
val lock = ReentrantLock()
var count = 0
repeat(1000) {
    thread {
        repeat(1000) {
            lock.lock()
            try {
                count++
            } finally {
                lock.unlock()
            }
        }
    }
}
```

#### 5️⃣ **volatile的正确用法**

```kotlin
// ✅ 正确场景1：简单的标志位
@Volatile
var shutdown = false

fun shutdown() {
    shutdown = true
}

fun run() {
    while (!shutdown) {  // 每次都从主存读取
        doWork()
    }
}

// ✅ 正确场景2：引用的可见性（不是值的原子性）
@Volatile
var config: Config? = null

fun loadConfig() {
    val newConfig = Config(...)
    config = newConfig  // 引用替换，整体可见
}

// ✅ 正确场景3：双重检查锁（虽然复杂，但是经典模式）
class Singleton {
    companion object {
        @Volatile
        private var instance: Singleton? = null
        
        fun getInstance(): Singleton {
            if (instance == null) {  // 第一次检查，无锁
                synchronized(Singleton::class) {  // 持有锁
                    if (instance == null) {  // 第二次检查
                        instance = Singleton()  // volatile保证可见性
                    }
                }
            }
            return instance!!
        }
    }
}
```

#### 6️⃣ **常见面试追问和答案**

```
Q: "为什么volatile的写操作会立即同步到主存？"
A: 通过CPU的MESI协议（缓存一致性协议）
   写入volatile变量 → 发送总线信号 → 其他核心的缓存失效
   读取volatile变量 → 缓存中没有 → 从主存读取最新值

Q: "volatile能替代锁吗？"
A: 不能。volatile只能保证可见性，不能保证原子性和互斥性
   如果需要互斥（多步操作），必须用锁

Q: "AtomicInteger和volatile count++哪个好？"
A: AtomicInteger更好
   - AtomicInteger使用CAS原子操作（无锁）
   - 性能更高（不需要阻塞等待）
   - 代码更清晰（意图明确）

Q: "JVM怎么实现volatile的可见性？"
A: 通过插入内存屏障指令：
   - 写入前：StoreStore屏障（前面的写完成）
   - 写入后：StoreLoad屏障（保证对其他核心可见）
   - 读取前：LoadLoad屏障
   - 读取后：LoadStore屏障
```

---

### Q2: ThreadLocal的实现原理和应用场景？如何避免内存泄漏？

#### 1️⃣ **原理解释**

```
ThreadLocal设计架构：

┌─────────────────────────────────────────────┐
│ ThreadLocal<T>                              │
│ - get()                                     │
│ - set(T value)                              │
│ - remove()                                  │
└─────────────────────────────────────────────┘
                ↓
┌─────────────────────────────────────────────┐
│ Thread (每个线程一个)                        │
│ - threadLocals: ThreadLocalMap              │
│ - inheritableThreadLocals                   │
└─────────────────────────────────────────────┘
                ↓
┌─────────────────────────────────────────────┐
│ ThreadLocalMap (内部类，哈希表)              │
│ - Entry[] table                             │
│ - Entry = WeakReference<ThreadLocal> + 值   │
│                                              │
│ 特殊性：                                    │
│ 1. Key用WeakReference包装                   │
│ 2. Value直接强引用                          │
│ 3. Hash冲突采用线性探测                     │
└─────────────────────────────────────────────┘
```

**工作流程**：

```
set(value) 时：
1. 获取当前线程 → Thread.currentThread()
2. 获取线程的ThreadLocalMap → thread.threadLocals
3. 计算哈希值：threadLocal.hashCode() & (table.length - 1)
4. 存入Map：table[index] = Entry(threadLocal, value)

get() 时：
1. 获取当前线程 → Thread.currentThread()
2. 获取线程的ThreadLocalMap
3. 按照相同的哈希规则查找Entry
4. 返回Entry.value
5. 若未找到则返回initialValue()或null
```

#### 2️⃣ **为什么这样设计**

```
Q: 为什么Key用WeakReference？
A: 防止ThreadLocal对象无法被GC
   - 如果Key是强引用，ThreadLocal对象就不会被回收
   - 使用WeakReference，当外部不再引用时，GC可以回收
   
Q: 为什么Value是强引用？
A: Value的生命周期应该由开发者控制
   - 如果Value也是弱引用，很容易被GC（不符合语义）
   - 开发者明确调用remove()才应该清理

Q: 那为什么还会内存泄漏？
A: 关键是线程不结束！
   - ThreadLocal本身没有问题
   - 问题在于：线程活着 → threadLocals.table活着 → Entry活着 → Value活着
   - 典型场景：线程池中的工作线程一直存活（永不停止）
```

#### 3️⃣ **内存泄漏场景分析**

```kotlin
// ⚠️ 泄漏场景：线程池 + ThreadLocal
class ApiClient {
    companion object {
        val userToken = ThreadLocal<String>()  // ThreadLocal变量
    }
    
    fun getUserInfo(): User {
        return apiCall(userToken.get())  // 某个请求中设置了token
    }
}

// 在线程池中使用
val executor = Executors.newFixedThreadPool(10)

repeat(100) {
    executor.submit {
        ApiClient.userToken.set("token_${Thread.currentThread().id}")
        val user = ApiClient.getUserInfo()
        // ⚠️ 问题：没有调用remove()！
        // threadLocals.table中的Entry会一直占用内存
    }
}

// 内存流失原因：
// 1. 线程池线程一直活着（不销毁）
// 2. threadLocals.table指向Entry
// 3. Entry虽然Key被回收（WeakReference），但Value仍被强引用
// 4. 如果Value是大对象（如大String、Context），会泄漏
// 总结：线程不死→threadLocals不死→Entry不死→Value不死
```

#### 4️⃣ **代码实现（安全用法）**

```kotlin
// ✅ 方案1：使用try-finally确保清理
class ApiClient {
    companion object {
        val userToken = ThreadLocal<String>()
        val requestContext = ThreadLocal<Context>()
    }
    
    fun handleRequest(context: Context, token: String): Response {
        userToken.set(token)
        requestContext.set(context)
        try {
            return doRequest()
        } finally {
            // 关键：清理ThreadLocal
            userToken.remove()
            requestContext.remove()
        }
    }
}

// ✅ 方案2：自定义wrapper简化操作
class ThreadLocalScope<T>(private val initializer: () -> T) {
    private val threadLocal = ThreadLocal<T>()
    
    inline fun <R> use(block: (T) -> R): R {
        val value = threadLocal.getOrSet { initializer() }
        try {
            return block(value)
        } finally {
            threadLocal.remove()
        }
    }
    
    private fun ThreadLocal<T>.getOrSet(init: () -> T): T {
        var value = get()
        if (value == null) {
            value = init()
            set(value)
        }
        return value
    }
}

// 使用
val userScope = ThreadLocalScope { User() }
userScope.use { user ->
    user.process()  // 自动清理
}

// ✅ 方案3：在线程池中自动清理（最佳实践）
class ThreadPoolTaskWrapper<T>(
    private val task: () -> T,
    private val cleanups: List<ThreadLocal<*>>
) {
    operator fun invoke(): T {
        try {
            return task()
        } finally {
            cleanups.forEach { it.remove() }
        }
    }
}

val executor = Executors.newFixedThreadPool(10)
fun <T> submit(
    task: () -> T,
    vararg locals: ThreadLocal<*>
): Future<T> {
    return executor.submit { ThreadPoolTaskWrapper(task, locals.toList())() }
}

// 使用
submit(
    { ApiClient.getUserInfo() },
    ApiClient.userToken,
    ApiClient.requestContext
)

// ✅ 方案4：与Kotlin协程结合（现代方案）
// 协程的上下文不依赖ThreadLocal，更安全
class RequestScope {
    val token: String by lazy { extractToken() }
    val context: Context by lazy { extractContext() }
}

suspend fun handleRequest() {
    val scope = RequestScope()  // 不需要ThreadLocal
    withContext(Dispatchers.IO) {
        doRequest(scope.token, scope.context)
    }
    // 函数结束，scope自动清理（无泄漏）
}
```

#### 5️⃣ **Android中的ThreadLocal应用**

```kotlin
// 1️⃣ Looper（最著名的使用）
public final class Looper {
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
    
    public static void prepare() {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(false));  // 每个线程一个Looper
    }
    
    public static @Nullable Looper myLooper() {
        return sThreadLocal.get();  // 获取当前线程的Looper
    }
}

// 2️⃣ Choreographer（掌控帧绘制时序）
public final class Choreographer {
    private static final ThreadLocal<Choreographer> sThreadInstance =
            new ThreadLocal<Choreographer>() {
        @Override
        protected Choreographer initialValue() {
            return new Choreographer(looper, vsyncSource);
        }
    };
}

// 3️⃣ Handler使用示例（本质上依赖ThreadLocal的Looper）
class MessageProcessor {
    fun processMessage(msg: Message) {
        // Handler内部获取Looper
        // Looper.myLooper() 本质上就是 Looper.sThreadLocal.get()
        val handler = Handler(Looper.myLooper()!!)
        handler.post { updateUI() }
    }
}
```

#### 6️⃣ **常见面试追问**

```
Q: "ThreadLocal的哈希冲突怎么处理？"
A: 线性探测法
   1. 计算初始位置：hash & (length - 1)
   2. 如果已占用，尝试下一位：(hash + 1) & (length - 1)
   3. 继续探测直到找到空位或Entry
   缺点：容易形成聚集，效率下降

Q: "ThreadLocalMap的长度为什么是2的幂次？"
A: 加速哈希计算
   - hash & (length - 1) 等价于 hash % length（但更快）
   - 只在length是2的幂次时成立

Q: "Entry的Key为什么一定要清理？"
A: 否则会形成"浮动Entry"
   - ThreadLocal被GC（Key为null）
   - 但Entry仍存在于table中
   - 浪费空间，且可能影响后续查询
   ThreadLocalMap.cleanSomeSlots()会定期清理这些浮动Entry

Q: "InheritableThreadLocal是什么？"
A: 允许子线程继承父线程的ThreadLocal值
   - new Thread() 时，会复制parent的inheritableThreadLocals
   - 线程池中注意：由于线程复用，继承可能有问题
   - 现代解决方案：TransmittableThreadLocal或使用上下文API
```

---

### Q3: ReentrantLock vs synchronized的优劣势

#### 1️⃣ **原理对比**

```
synchronized：
- JVM级别的语言关键字
- 隐式获取和释放（自动管理）
- 使用ObjectMonitor实现
- 偏向锁 → 轻量级锁 → 重量级锁（三阶段优化）

ReentrantLock：
- API级别的显式锁
- 需要手动获取和释放（try-finally）
- 使用AbstractQueuedSynchronizer(AQS)实现
- 基于CAS原子操作
```

#### 2️⃣ **功能对比表**

|特性|synchronized|ReentrantLock|
|---|---|---|
|**可重入**|✅|✅|
|**公平性**|❌ 非公平|✅/❌ 可配置|
|**中断**|❌ 无法中断|✅ lockInterruptibly()|
|**超时**|❌|✅ tryLock(timeout)|
|**条件变量**|❌ wait/notify|✅ Condition|
|**性能**|✅ JVM优化|⚠️ 次于synchronized|
|**易用性**|✅ 隐式管理|⚠️ 需手动管理|
|**GC友好**|✅|❌ 容易泄漏|

#### 3️⃣ **代码实现对比**

```kotlin
// synchronized: 简单场景
class Counter {
    private var count = 0
    
    @Synchronized
    fun increment() {
        count++  // 简洁，自动管理
    }
}

// ReentrantLock: 精细控制
class AdvancedCounter {
    private var count = 0
    private val lock = ReentrantLock()
    private val notEmpty = lock.newCondition()
    
    // 需要手动管理，但功能更强
    fun incrementInterruptibly() {
        lock.lockInterruptibly()  // 支持中断
        try {
            count++
            notEmpty.signalAll()  // 支持条件通知
        } finally {
            lock.unlock()
        }
    }
    
    // tryLock支持超时
    fun incrementWithTimeout(): Boolean {
        return if (lock.tryLock(100, TimeUnit.MILLISECONDS)) {
            try {
                count++
                true
            } finally {
                lock.unlock()
            }
        } else {
            false  // 超时失败
        }
    }
}

// ✅ 场景1：简单互斥 → 用synchronized（更快）
class SimpleCache {
    private val data = mutableMapOf<String, Any>()
    
    @Synchronized
    fun get(key: String) = data[key]
    
    @Synchronized
    fun put(key: String, value: Any) {
        data[key] = value
    }
}

// ✅ 场景2：复杂场景（需要中断、超时、条件变量）→ 用ReentrantLock
class ProducerConsumerQueue<T> {
    private val queue = LinkedList<T>()
    private val lock = ReentrantLock()
    private val notEmpty = lock.newCondition()
    private val notFull = lock.newCondition()
    private val maxSize = 100
    
    fun put(item: T) {
        lock.lock()
        try {
            while (queue.size >= maxSize) {
                notFull.await()  // 等待消费
            }
            queue.add(item)
            notEmpty.signal()  // 通知消费者
        } finally {
            lock.unlock()
        }
    }
    
    fun take(): T {
        lock.lockInterruptibly()  // 支持中断
        try {
            while (queue.isEmpty()) {
                notEmpty.awaitInterruptibly()  // 支持被中断
            }
            val item = queue.remove()
            notFull.signal()
            return item
        } finally {
            lock.unlock()
        }
    }
}

// ❌ 常见错误：忘记释放锁
class BadLockUsage {
    private val lock = ReentrantLock()
    private var count = 0
    
    fun badIncrement() {
        lock.lock()
        // ⚠️ 如果这里抛异常，永远不会执行unlock()！
        count++
        Thread.sleep(100)
        lock.unlock()  // 可能不会执行
    }
    
    // ✅ 正确方式
    fun goodIncrement() {
        lock.lock()
        try {
            count++
            Thread.sleep(100)
        } finally {
            lock.unlock()  // 保证执行
        }
    }
}

// ✅ 场景3：需要超时的场景
class TimeoutLockDemo {
    private val lock = ReentrantLock()
    
    fun tryAcquireWithTimeout(timeoutMillis: Long): Boolean {
        return try {
            if (lock.tryLock(timeoutMillis, TimeUnit.MILLISECONDS)) {
                try {
                    doWork()
                    true
                } finally {
                    lock.unlock()
                }
            } else {
                println("获取锁超时，可能是资源竞争激烈")
                false
            }
        } catch (e: InterruptedException) {
            println("获取锁时被中断")
            false
        }
    }
    
    private fun doWork() {
        // 实际工作
    }
}
```

#### 4️⃣ **性能对比**

```
测试场景：10个线程，每个线程执行100000次递增操作

结果对比（单位：毫秒）：
┌──────────────────┬─────────────┬──────────────┐
│ 并发程度         │ synchronized │ ReentrantLock │
├──────────────────┼─────────────┼──────────────┤
│ 低竞争(1线程)     │ ~10ms       │ ~15ms        │
│ 中竞争(5线程)     │ ~200ms      │ ~250ms       │
│ 高竞争(10线程)    │ ~800ms      │ ~900ms       │
└──────────────────┴─────────────┴──────────────┘

结论：
1. synchronized在现代JVM中通过偏向锁优化，性能接近
2. 低竞争场景下，synchronized略快（因为偏向锁）
3. 高竞争场景下，二者性能接近（都降级到重量级锁）
4. ReentrantLock的优势是功能性，不是性能
```

#### 5️⃣ **常见面试追问**

```
Q: "为什么synchronized比ReentrantLock快？"
A: 偏向锁优化
   - 第一次访问：CAS设置偏向标记
   - 后续访问：无需CAS，直接比较ThreadID
   - ReentrantLock每次都需要CAS原子操作

Q: "什么时候用synchronized，什么时候用ReentrantLock？"
A: 
   synchronized: 90%的场景
   - 简单互斥
   - JVM能做优化
   - 代码简洁
   
   ReentrantLock: 特殊场景
   - 需要中断锁等待（lockInterruptibly）
   - 需要尝试超时获取（tryLock）
   - 需要多个条件变量（Condition）
   - 需要公平性保证

Q: "ReentrantLock一定比synchronized功能强吗？"
A: 功能更多，不一定更强
   synchronized的优势：
   - wait/notify可以处理简单条件
   - JVM可以做激进优化（偏向锁、自适应自旋）
   
   ReentrantLock的优势：
   - 多个Condition，更细粒度
   - 超时/中断，更灵活
```

---

## 第二部分：Kotlin协程（★★★★★）

### Q4: suspend函数是什么？如何编译成状态机？

#### 1️⃣ **原理解释**

```
suspend函数本质：
  编译时的代码转换机制
  
┌─────────────────────────────┐
│ 源代码 (suspend函数)         │
│                             │
│ suspend fun fetchUser(): User │
└─────────────────────────────┘
            ↓ Kotlin编译器转换
┌─────────────────────────────┐
│ 字节码 (CPS转换)             │
│                             │
│ fun fetchUser(                │
│   continuation: Continuation<User> │
│ ): Any                        │
└─────────────────────────────┘
```

**关键概念**：

```
Continuation（延续）：
- 代表异步计算的剩余部分
- 包含：状态、变量、恢复点

Continuation接口：
interface Continuation<T> {
    val context: CoroutineContext
    fun resumeWith(result: Result<T>)  // 恢复执行
}
```

#### 2️⃣ **编译过程详解**

```kotlin
// 原始代码
suspend fun fetchUser(id: String): User {
    val response = apiService.getUser(id)  // 挂起点1
    val profile = apiService.getProfile(id)  // 挂起点2
    return User(response, profile)
}

// 编译后的样子（伪代码）
class FetchUserContinuation(
    val id: String,
    val completion: Continuation<User>
) : Continuation<User> {
    
    var label = 0  // 状态机状态
    var response: UserResponse? = null  // 保存中间变量
    var profile: UserProfile? = null
    
    override val context: CoroutineContext 
        = completion.context
    
    override fun resumeWith(result: Result<User>) {
        try {
            when (label) {
                0 -> {
                    // 初始状态：调用第一个挂起函数
                    label = 1
                    apiService.getUser(id, this)  // this为continuation
                    return  // 暂停等待
                }
                
                1 -> {
                    // 恢复点1：第一个挂起函数返回
                    response = result.getOrThrow() as UserResponse
                    
                    // 调用第二个挂起函数
                    label = 2
                    apiService.getProfile(id, this)
                    return  // 再次暂停
                }
                
                2 -> {
                    // 恢复点2：第二个挂起函数返回
                    profile = result.getOrThrow() as UserProfile
                    
                    // 执行最后的逻辑
                    val finalResult = User(response!!, profile!!)
                    
                    // 通知completion（调用者）
                    completion.resumeWith(Result.success(finalResult))
                    return
                }
            }
        } catch (e: Throwable) {
            completion.resumeWith(Result.failure(e))
        }
    }
}

// 调用时
fun fetchUser(
    id: String, 
    completion: Continuation<User>
): Any {  // 返回Any而非User！
    // 创建Continuation包装器
    return FetchUserContinuation(id, completion).resumeWith(Result.success(Unit))
}
```

#### 3️⃣ **为什么这样设计**

```
Q: 为什么返回值是Any而不是User？
A: 因为函数可能：
   1. 同步完成（直接返回User）
   2. 异步暂停（返回COROUTINE_SUSPENDED魔法值）
   
   调用者需要区分这两种情况
   
   Any的可能值：
   - User实例 → 同步完成
   - COROUTINE_SUSPENDED → 异步暂停，等待恢复
   - Throwable → 错误

Q: 为什么需要label（状态机）？
A: 每次恢复执行时需要知道从哪里继续
   label标记断点位置，不需要保存整个栈帧
   大大节省内存

Q: 为什么要保存中间变量？
A: suspend函数局部变量会被转换为Continuation的字段
   当协程暂停时，这些字段保存状态
   当协程恢复时，这些字段恢复上下文
```

#### 4️⃣ **代码实现演示**

```kotlin
// 基础suspend函数
suspend fun fetchUser(id: String): User {
    return apiService.getUser(id)
}

// 模拟编译后的代码
interface ApiService {
    fun getUser(id: String, continuation: Continuation<User>): Any
}

// 调用suspend函数
launch {
    val user = fetchUser("123")  // 编译器转换为：
    // val user = fetchUser("123", object : Continuation<User> {
    //     override fun resumeWith(result: Result<User>) {
    //         // 处理结果
    //     }
    // })
}

// ✅ 实际代码中的suspend函数
class UserRepository(private val apiService: ApiService) {
    
    // 单个挂起点
    suspend fun getUser(id: String): User = apiService.getUser(id)
    
    // 多个挂起点（编译为状态机）
    suspend fun getUserWithDetails(id: String): UserWithDetails {
        val user = apiService.getUser(id)        // 挂起点1
        val details = apiService.getDetails(id)  // 挂起点2
        return UserWithDetails(user, details)
    }
    
    // 条件分支挂起
    suspend fun getUserIfExists(id: String): User? {
        return try {
            apiService.getUser(id)  // 挂起点
        } catch (e: HttpException) {
            null
        }
    }
    
    // 循环中的挂起
    suspend fun getAllUsers(ids: List<String>): List<User> {
        return ids.map { id ->
            apiService.getUser(id)  // 每次循环都是挂起点
        }
    }
}

// 使用案例
viewModelScope.launch {
    try {
        // 这些suspend函数调用会自动转换为continuation
        val user = userRepository.getUser("123")
        val details = userRepository.getUserWithDetails("456")
        val optional = userRepository.getUserIfExists("789")
        val users = userRepository.getAllUsers(listOf("1", "2", "3"))
        
        _uiState.value = UiState.Success(user)
    } catch (e: Exception) {
        _uiState.value = UiState.Error(e.message)
    }
}

// ⚠️ 常见错误
suspend fun wrongSuspend() {
    // ❌ 启动协程（这是阻塞的）
    GlobalScope.launch {  // 不应该在suspend函数中这样做
        delay(1000)
    }
}

// ✅ 正确的suspend函数
suspend fun correctSuspend() {
    delay(1000)  // 直接使用suspend函数
}
```

#### 5️⃣ **Continuation API的使用**

```kotlin
// 底层API（一般不用）
suspend fun lowLevelSuspend(): String = suspendCancellableCoroutine { continuation ->
    // continuation是Continuation<String>接口
    // 异步操作完成后调用resume
    
    apiService.fetchData(object : Callback<String> {
        override fun onSuccess(data: String) {
            continuation.resume(data)  // 恢复协程
        }
        
        override fun onFailure(error: Throwable) {
            continuation.resumeWithException(error)  // 异常恢复
        }
    })
}

// 现代写法（使用suspendCancellableCoroutine）
suspend fun modernSuspend(): String = suspendCancellableCoroutine { continuation ->
    val job = apiService.fetch(
        onSuccess = { continuation.resume(it) },
        onFailure = { continuation.resumeWithException(it) }
    )
    
    // 支持取消
    continuation.invokeOnCancellation {
        job.cancel()
    }
}

// 最现代的写法（使用Flow或async）
suspend fun latestSuspend(): String {
    return withContext(Dispatchers.IO) {
        apiService.fetch()  // 自动处理Continuation
    }
}
```

#### 6️⃣ **常见面试追问**

```
Q: "suspend函数和普通函数的区别？"
A: 
   编译级别：
   - suspend函数编译后多一个Continuation参数
   - 普通函数编译后是普通的JVM方法
   
   功能：
   - suspend函数可以被挂起和恢复
   - 普通函数不能（执行到底）

Q: "为什么suspend函数返回值是Any？"
A: 因为可能返回两种值：
   1. 实际结果（T类型）→ 同步完成
   2. COROUTINE_SUSPENDED → 异步暂停
   
   Any是它们的父类型

Q: "label状态机最多有多少个状态？"
A: 理论上无限
   实际上受限于：
   1. suspend点的数量（一个suspend点一个状态）
   2. 条件分支复杂度
   3. 编译器的优化

Q: "能在非suspend函数中调用suspend函数吗？"
A: 不能
   非suspend函数没有Continuation参数
   编译会报错：Suspend function can only be called from coroutine
```

---

### Q5: LiveData vs StateFlow vs SharedFlow

#### 1️⃣ **原理对比**

```
LiveData：
- 生命周期感知的可观察数据
- 通过Lifecycle组件在作用域内生效
- 主线程回调
- 数据粘性（订阅者会收到最后一个值）

StateFlow：
- 热流（Flow的特化）
- 值类容器，始终有值
- 支持背压
- 数据粘性

SharedFlow：
- 热流
- 可配置重放策略
- 灵活的背压处理
- 可选数据粘性
```

#### 2️⃣ **代码实现对比**

```kotlin
// ======================== LiveData ========================
class UserViewModel : ViewModel() {
    private val _user = MutableLiveData<User>()
    val user: LiveData<User> = _user
    
    fun loadUser(id: String) {
        viewModelScope.launch {
            try {
                val user = repository.getUser(id)
                _user.value = user  // 主线程更新
            } catch (e: Exception) {
                _user.value = User()  // 错误处理
            }
        }
    }
}

// 订阅
class UserActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        viewModel.user.observe(this) { user ->
            // 自动生命周期感知
            // Activity销毁时自动取消订阅
            updateUI(user)
        }
    }
}

// ======================== StateFlow ========================
class UserViewModel : ViewModel() {
    private val _userState = MutableStateFlow<User?>(null)
    val userState: StateFlow<User?> = _userState.asStateFlow()
    
    fun loadUser(id: String) {
        viewModelScope.launch {
            try {
                val user = repository.getUser(id)
                _userState.value = user  // 任意线程可以设置
            } catch (e: Exception) {
                // 错误处理更复杂
            }
        }
    }
}

// 订阅（需要手动管理生命周期）
class UserActivity : AppCompatActivity() {
    private val viewModel: UserViewModel by viewModels()
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                // 在STARTED ~ STOPPED之间收集
                viewModel.userState.collect { user ->
                    updateUI(user)
                }
            }
        }
    }
}

// ======================== SharedFlow ========================
class EventBus {
    private val _events = MutableSharedFlow<Event>(
        replay = 1,           // 缓存最后1个事件
        extraBufferCapacity = 10,  // 额外缓冲
        onBufferOverflow = BufferOverflow.DROP_OLDEST  // 缓冲满时丢弃旧数据
    )
    val events = _events.asSharedFlow()
    
    suspend fun emit(event: Event) {
        _events.emit(event)
    }
}

// 使用
class EventListener(private val eventBus: EventBus) {
    fun listenEvents() {
        viewModelScope.launch {
            eventBus.events.collect { event ->
                handleEvent(event)
            }
        }
    }
}

// ======================== 对比表 ========================
/*
                LiveData    StateFlow   SharedFlow
生命周期感知      ✅          ❌          ❌
粘性数据          ✅          ✅          ✅（可配）
背压支持          ❌          ✅          ✅
重放次数          1           1           可配置
线程安全          ✅          ✅          ✅
主线程更新        ✅          ❌          ❌
多个订阅者        ✅          ✅          ✅
冷流/热流         -           热          热
*/
```

#### 3️⃣ **场景选择指南**

```kotlin
// ✅ 场景1：ViewModel向Activity发送单个数据 → LiveData
class LoginViewModel : ViewModel() {
    private val _loginResult = MutableLiveData<LoginResult>()
    val loginResult: LiveData<LoginResult> = _loginResult
    
    fun login(email: String, password: String) {
        viewModelScope.launch {
            val result = authRepository.login(email, password)
            _loginResult.value = result
        }
    }
}

class LoginActivity : AppCompatActivity() {
    private val viewModel: LoginViewModel by viewModels()
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // LiveData自动处理生命周期
        viewModel.loginResult.observe(this) { result ->
            navigateToHome()
        }
    }
}

// ✅ 场景2：管理可变状态（MVVM+单向数据流） → StateFlow
class UserListViewModel : ViewModel() {
    private val _userListState = MutableStateFlow<UserListState>(
        UserListState.Loading
    )
    val userListState: StateFlow<UserListState> = _userListState
    
    fun loadUsers() {
        viewModelScope.launch {
            _userListState.value = UserListState.Loading
            try {
                val users = repository.getUsers()
                _userListState.value = UserListState.Success(users)
            } catch (e: Exception) {
                _userListState.value = UserListState.Error(e)
            }
        }
    }
}

// ✅ 场景3：事件总线（一对多通知） → SharedFlow
class NotificationCenter {
    private val _notifications = MutableSharedFlow<Notification>(
        replay = 0,  // 不缓存历史
        extraBufferCapacity = 100  // 缓冲突发
    )
    val notifications = _notifications.asSharedFlow()
    
    suspend fun post(notification: Notification) {
        _notifications.emit(notification)
    }
}

// ✅ 场景4：数据缓存，支持多个订阅者 → StateFlow + 数据缓存
class CacheRepository {
    private val _cachedData = MutableStateFlow<Data?>(null)
    val cachedData: StateFlow<Data?> = _cachedData
    
    suspend fun getData(id: String): Data {
        // 先检查缓存
        cachedData.value?.let { return it }
        
        // 从网络获取
        val data = apiService.fetchData(id)
        _cachedData.value = data
        return data
    }
}

// ✅ 场景5：高频事件流，支持背压 → SharedFlow with BackPressure
class SensorDataCollector {
    private val _sensorData = MutableSharedFlow<SensorReading>(
        replay = 0,
        extraBufferCapacity = 0,  // 无缓冲，完全背压
        onBufferOverflow = BufferOverflow.SUSPEND  // 订阅者慢时挂起生产者
    )
    val sensorData = _sensorData.asSharedFlow()
    
    suspend fun emit(reading: SensorReading) {
        _sensorData.emit(reading)  // 背压会自动调节
    }
}

class SensorListener {
    fun listen(collector: SensorDataCollector) {
        viewModelScope.launch {
            collector.sensorData
                .buffer(10)  // 缓冲10个，然后背压
                .collect { reading ->
                    slowProcessing(reading)
                }
        }
    }
}
```

#### 4️⃣ **数据粘性的影响**

```kotlin
// 粘性数据演示
val state = MutableStateFlow(0)

// 发出1
state.value = 1
delay(100)

// 发出2
state.value = 2
delay(100)

// 现在订阅（100ms后）
state.collect { value ->
    println(value)  // 立即收到2（粘性！）
    // 然后等待下一个值
}

// 为什么粘性？
// StateFlow设计用于保存状态
// 订阅者应该知道当前状态是什么

// 如果不想要粘性 → 使用SharedFlow(replay = 0)
val events = MutableSharedFlow<Event>(replay = 0)
events.emit(Event1)
delay(100)
events.emit(Event2)

events.collect { event ->
    println(event)  // 只收到Event2（订阅后的）
    // Event1已错过
}
```

#### 5️⃣ **常见错误和正确做法**

```kotlin
// ❌ 错误：在LiveData中做长时间计算
class BadViewModel : ViewModel() {
    private val _result = MutableLiveData<Result>()
    val result: LiveData<Result> = _result
    
    fun compute() {
        // ❌ 在observe回调中做计算会阻塞主线程
        result.observe(this) { result ->
            val heavy = heavyComputation(result)  // 主线程！
        }
    }
}

// ✅ 正确：在协程中计算，然后post结果
class GoodViewModel : ViewModel() {
    private val _result = MutableLiveData<Result>()
    val result: LiveData<Result> = _result
    
    fun compute() {
        viewModelScope.launch(Dispatchers.Default) {
            val computed = heavyComputation()
            _result.postValue(computed)  // 线程安全地设置
        }
    }
}

// ❌ 错误：StateFlow中做线程转换错误
class BadStateViewModel : ViewModel() {
    private val _state = MutableStateFlow<Data?>(null)
    val state = _state.asStateFlow()
    
    fun load() {
        viewModelScope.launch {
            val data = withContext(Dispatchers.IO) {
                fetchData()
            }
            _state.value = data  // ✅ 现在在Main线程（正确）
        }
    }
}

// ❌ 错误：忘记取消SharedFlow订阅
class BadEventListener {
    fun listen(eventBus: EventBus) {
        GlobalScope.launch {  // ❌ GlobalScope会泄漏！
            eventBus.events.collect { event ->
                // 永远不会停止
            }
        }
    }
}

// ✅ 正确：使用viewModelScope
class GoodEventListener(private val viewModel: ViewModel) {
    fun listen(eventBus: EventBus) {
        viewModel.viewModelScope.launch {  // ✅ ViewModel销毁时自动取消
            eventBus.events.collect { event ->
                handleEvent(event)
            }
        }
    }
}

// ❌ 错误：混淆LiveData和StateFlow的生命周期
class ConfusedViewModel : ViewModel() {
    val data = MutableStateFlow<Data?>(null)
    
    fun bindToActivity(activity: AppCompatActivity) {
        // ❌ StateFlow不会自动停止订阅
        // Activity销毁后仍在收集！
        activity.lifecycleScope.launch {
            data.collect { /*...*/ }
        }
    }
}

// ✅ 正确：使用repeatOnLifecycle
class CorrectViewModel : ViewModel() {
    val data = MutableStateFlow<Data?>(null)
    
    fun bindToActivity(activity: AppCompatActivity) {
        activity.lifecycleScope.launch {
            activity.repeatOnLifecycle(Lifecycle.State.STARTED) {
                // STARTED ~ STOPPED期间收集
                // 自动处理暂停/恢复
                data.collect { /*...*/ }
            }
        }
    }
}
```

#### 6️⃣ **常见面试追问**

```
Q: "为什么LiveData要绑定Lifecycle？"
A: 
   1. 自动取消订阅（Activity销毁时）
   2. 避免内存泄漏
   3. 不向后台的观察者发送更新
   4. 不会在收集数据时更新销毁的Activity

Q: "StateFlow和LiveData为什么都有粘性？"
A: 都代表"状态"而非"事件"
   - 状态应该一直存在
   - 新订阅者应该知道当前状态
   - 这是设计选择，不是bug

Q: "怎样在StateFlow中实现类似LiveData的生命周期感知？"
A: 使用repeatOnLifecycle
```

lifecycle.repeatOnLifecycle(Lifecycle.State.STARTED) { viewModel.state.collect { ... } }

```

Q: "SharedFlow(replay=0)和Channel有什么区别？"
A: 
SharedFlow：
- 多订阅者
- Flow的热流实现
- 背压和缓冲灵活

Channel：
- 类似Queue
- 订阅者端枚举
- 强制背压

Q: "怎样从LiveData迁移到StateFlow？"
A: 
1. 替换MutableLiveData → MutableStateFlow
2. 替换observe() → lifecycleScope.launch + repeatOnLifecycle + collect
3. 注意：StateFlow不自动处理生命周期
```

---

## 第三部分：内存管理（★★★★★）

### Q6: LeakCanary的原理？如何检测内存泄漏？

#### 1️⃣ **原理解释**

```
LeakCanary检测流程：

1️⃣ 对象销毁检测
   └─ AppWatcher监听：
      • Activity销毁（Lifecycle回调）
      • Fragment销毁
      • ViewModel销毁
      • 自定义对象

2️⃣ WeakReference + GC检测
   └─ 对销毁对象包装WeakReference
   └─ GC后检查是否被回收
      • 若被回收：✅ 正常
      • 若未回收：⚠️ 可能泄漏

3️⃣ 堆转储(Heap Dump)
   └─ Dump内存堆
   └─ 分析对象引用链
   └─ 找出最短保留路径(Shortest Path)

4️⃣ 报告生成
   └─ 追踪泄漏对象
   └─ 标记可疑引用
   └─ 显示在通知栏
```

#### 2️⃣ **为什么这样设计**

```
Q: 为什么用WeakReference检测？
A: WeakReference的特性
   - GC会回收WeakReference的引用对象
   - 若对象未被回收，说明还有强引用持有它
   - 这正是内存泄漏的表现

Q: 为什么要触发GC？
A: 确保不可达对象真的被回收
   - 没有触发GC，对象可能仍在内存中
   - 触发GC后，不可达对象必然被回收
   - 此时还存在 → 确定有强引用

Q: 为什么要分析引用链？
A: 找到泄漏的根本原因
   - WeakReference.get()==null：对象被回收 ✅
   - WeakReference.get()!=null：对象未回收，且能找到引用链
   - 引用链指向：GC Root → 泄漏对象
   - 通过分析引用链，定位哪个对象持有了它
```

#### 3️⃣ **代码实现原理**

```kotlin
// LeakCanary的简化实现原理

// 1️⃣ Activity销毁检测
class ActivityDestructionWatcher {
    fun install(application: Application) {
        application.registerActivityLifecycleCallbacks(
            object : Application.ActivityLifecycleCallbacks {
                override fun onActivityDestroyed(activity: Activity) {
                    checkLeak(activity)
                }
                
                override fun onActivityCreated(a: Activity, s: Bundle?) {}
                override fun onActivityStarted(activity: Activity) {}
                override fun onActivityResumed(activity: Activity) {}
                override fun onActivityPaused(activity: Activity) {}
                override fun onActivityStopped(activity: Activity) {}
                override fun onActivitySaveInstanceState(a: Activity, s: Bundle) {}
            }
        )
    }
    
    private fun checkLeak(activity: Activity) {
        // 2️⃣ 包装WeakReference
        val weakRef = WeakReference(activity)
        
        // 3️⃣ 延迟检查（给GC时间）
        mainHandler.postDelayed({
            checkIfReclaimed(weakRef)
        }, 5000)  // 5秒后检查
    }
    
    private fun checkIfReclaimed(weakRef: WeakReference<Activity>) {
        // 4️⃣ 尝试触发GC
        System.gc()
        System.runFinalization()
        
        // 5️⃣ 检查是否被回收
        if (weakRef.get() != null) {
            // ⚠️ 对象还在！可能泄漏
            dumpHeapAndAnalyze(weakRef)
        } else {
            // ✅ 对象被正常回收
        }
    }
    
    private fun dumpHeapAndAnalyze(weakRef: WeakReference<Activity>) {
        // 6️⃣ 导出Heap Dump文件
        val hprofFile = dumpHeap()
        
        // 7️⃣ 使用Shark分析
        val analysis = SharkInstanceLeakFinder.findLeaks(
            hprofFile,
            weakRef
        )
        
        // 8️⃣ 报告泄漏
        if (analysis != null) {
            showLeakNotification(analysis)
        }
    }
}

// 实际使用
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        LeakCanary.install(this)  // 自动监听所有Activity
    }
}
```

#### 4️⃣ **常见泄漏场景和解决方案**

```kotlin
// ❌ 泄漏场景1：Handler持有Activity引用
class BadActivity : AppCompatActivity() {
    private val handler = Handler(Looper.getMainLooper())
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // ❌ 匿名内部类持有this引用
        handler.postDelayed({
            println("延迟执行")
        }, 30000)
        
        // Activity在延迟任务执行前销毁
        // → Handler仍持有Activity引用
        // → LeakCanary会检测到：
        //    Activity销毁 → 但WeakReference仍可访问
    }
}

// ✅ 解决方案1：使用弱引用
class GoodActivity : AppCompatActivity() {
    private class MyHandler(activity: Activity) : Handler(Looper.getMainLooper()) {
        private val weakActivity = WeakReference(activity)
        
        override fun handleMessage(msg: Message) {
            weakActivity.get()?.let { activity ->
                // 安全地使用activity
            }
        }
    }
    
    private val handler = MyHandler(this)
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        handler.postDelayed({ /*...*/ }, 30000)
    }
}

// ✅ 解决方案2：使用静态内部类
class BetterActivity : AppCompatActivity() {
    private static class MyHandler extends Handler {
        private WeakReference<Activity> activityRef;
        
        MyHandler(Activity activity) {
            this.activityRef = new WeakReference<>(activity);
        }
        
        @Override
        public void handleMessage(Message msg) {
            Activity activity = activityRef.get();
            if (activity != null) {
                // use activity
            }
        }
    }
}

// ❌ 泄漏场景2：单例持有Context
class BadSingleton {
    companion object {
        val instance = BadSingleton()
    }
    
    private var context: Context? = null  // ⚠️ 强引用！
    
    fun init(context: Context) {
        this.context = context  // Activity Context泄漏！
    }
}

// ✅ 解决方案：使用Application Context
class GoodSingleton {
    companion object {
        val instance = GoodSingleton()
    }
    
    private var appContext: Context? = null
    
    fun init(context: Context) {
        this.appContext = context.applicationContext  // 应用级Context
    }
}

// ❌ 泄漏场景3：静态变量持有View引用
class BadUtil {
    companion object {
        var staticView: View? = null  // ⚠️ 静态强引用
        
        fun saveView(view: View) {
            staticView = view  // View所属Activity无法回收！
        }
    }
}

// ✅ 解决方案：不持有View，只传递数据
class GoodUtil {
    companion object {
        fun processView(view: View) {
            // 使用后立即释放，不保存引用
        }
    }
}

// ❌ 泄漏场景4：未注销监听器
class BadActivity : AppCompatActivity(), 
    BroadcastReceiver.Listener {
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        eventBus.register(this)  // 注册监听
        // ⚠️ 忘记注销！
    }
    
    override fun onEvent(event: Event) {
        updateUI(event)
    }
}

// ✅ 解决方案1：在onDestroy中注销
class GoodActivity1 : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        eventBus.register(this)
    }
    
    override fun onDestroy() {
        super.onDestroy()
        eventBus.unregister(this)  // 清理
    }
}

// ✅ 解决方案2：使用生命周期感知
class GoodActivity2 : AppCompatActivity() {
    private val listener = object : EventListener {
        override fun onEvent(event: Event) {
            updateUI(event)
        }
    }
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        lifecycle.addObserver(object : LifecycleObserver {
            @OnLifecycleEvent(Lifecycle.Event.ON_CREATE)
            fun onCreated() {
                eventBus.register(listener)
            }
            
            @OnLifecycleEvent(Lifecycle.Event.ON_DESTROY)
            fun onDestroyed() {
                eventBus.unregister(listener)
            }
        })
    }
}

// ❌ 泄漏场景5：协程泄漏
class BadCoroutineActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        GlobalScope.launch {  // ❌ GlobalScope不受生命周期控制
            delay(10000)
            updateUI()  // Activity销毁前不会执行，但协程还活着
        }
    }
}

// ✅ 解决方案：使用viewModelScope或lifecycleScope
class GoodCoroutineActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        lifecycleScope.launch {  // ✅ Activity销毁时自动取消
            delay(10000)
            updateUI()
        }
    }
}
```

#### 5️⃣ **Heap Dump分析**

```
LeakCanary检测到泄漏后会显示：

引用链示例：
┌─ GC Root
│  └─ MainActivity (retained 2.3MB)
│     └─ MyHandler (可疑)
│        └─ anonymous Runnable$1
│           └─ LoginFragment
│              └─ mContext = MainActivity ← 泄漏！

解读：
- MainActivity销毁后仍被Handler引用
- Handler通过Runnable引用
- Runnable引用LoginFragment
- LoginFragment通过mContext引用回MainActivity（循环）

解决：
- 使用弱引用包装Handler
- 或在onDestroy时移除待处理消息
- handler.removeCallbacksAndMessages(null)
```

#### 6️⃣ **常见面试追问**

```
Q: "WeakReference被回收后，get()返回什么？"
A: null
   WeakReference包装的对象被GC后，get()返回null

Q: "为什么LeakCanary默认只监听Activity？"
A: Activity销毁容易被遗忘
   - Activity的生命周期明确
   - 销毁时资源应该全部释放
   - 如果销毁后还有引用 → 确定泄漏

Q: "能在主进程之外监听吗？"
A: LeakCanary工作在独立进程
   - 可以监听多个App进程
   - 在settings中配置

Q: "LeakCanary对性能的影响？"
A: 
   开发阶段：中等
   - 每个Activity销毁时检查
   - 5秒后进行GC和Heap Dump
   
   生产环境：
   - 不应该在生产环境使用
   - debugImplementation引入（不包含在release中）

Q: "怎样自定义LeakCanary的检测对象？"
A:
   ```kotlin
   LeakCanary.trackObject(myObject)
```

LeakCanary会监视这个对象的销毁

```

---

[继续下一部分...]
```