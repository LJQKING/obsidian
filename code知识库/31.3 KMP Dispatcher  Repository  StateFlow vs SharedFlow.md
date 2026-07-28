## 三大支柱的核心逻辑速记

### 🎯 **第一部分：Repository  隔离 = 线程安全 + 性能**

**一句话原理**：

```
不同的操作类型竞争同一个线程 → 饥荒、卡顿、内存泄露
```

**关键场景**：

- ❌ 网络请求在 IO 线程跑，图片解码也在 IO 线程 → 线程饥荒
- ❌ 数据库修改没有 Mutex 保护 → 竞态条件，集合损坏
- ❌ 所有操作都在 Main 线程 → ANR 和 GC 暂停

**正确思路**：

```
Main (UI) → IO (网络/数据库) → Computation (JSON/加密)
  不阻塞       高并发             CPU 密集
```

---

### 🔀 **第二部分：Repository 模式 = 解耦 + 缓存 + 测试**

**一句话原理**：

```
ViewModel 不应该知道 HTTP 库的存在
```

**三个地狱**：

1. **紧耦合**：改网络库 → 改 ViewModel 代码
2. **无缓存**：快速点击导致重复请求
3. **难测试**：必须依赖真实网络

**Repository 解决**：

```
ViewModel → Repository(缓存+业务逻辑) → DataSource(网络/DB)
           ▲                          ▲
         单一职责                    易于 mock
```

**缓存三层**：L1(内存) → L2(本地 DB) → L3(网络)

---

### 🔁 **第三部分：StateFlow vs SharedFlow = 建模正确**

**快速判断法则**：

|问题|选择|理由|
|---|---|---|
|"我需要表示**当前**的 UI 状态"|StateFlow|新订阅者立即得到当前值|
|"我需要**通知**所有人一个事件"|SharedFlow|事件一发就完，不需要保留|
|"屏幕旋转后状态应该保留"|StateFlow|✅ 自动保留，不重放事件|
|"屏幕旋转后事件不应该重复触发"|SharedFlow|✅ 不保留历史，旋转后新订阅收不到|

---

## 面试必问的三道题

### 题 1：为什么 Dispatcher.IO 最多 64 个线程？

**标准答案流程**：

1. **描述问题**：网络操作需要高并发，CPU 操作应该隔离
2. **资源限制**：
    - 每个线程 ~1MB 堆栈
    - Linux socket 文件描述符上限 ~1024
    - 64 个线程 × 1MB = 64MB（可接受）
3. **最优实践**：超过线程数 → context switch 开销 > 收益
4. **Android 差异**：低端机 4 核，高端机 8 核 → Dispatchers.Default 自动适配

---

### 题 2：Repository 和 DataSource 如何分工？

**标准答案**：

- **Repository**（平台无关）：
    - 定义业务接口
    - 实现缓存策略
    - 协调多个数据源
- **DataSource**（平台差异）：
    - 网络请求（Ktor 客户端）
    - 本地存储（SQLDelight）
    - 只负责 CRUD，不做业务逻辑

**实例**：

```kotlin
UserRepository.getUser() {
  // 1. 查内存缓存（Repository 职责）
  // 2. 查本地 DB（调用 LocalDataSource）
  // 3. 发网络请求（调用 RemoteDataSource）
  // 4. 更新缓存（Repository 职责）
}
```

---

### 题 3：屏幕旋转时，StateFlow 和 SharedFlow 分别会发生什么？

**标准答案**：

|场景|StateFlow|SharedFlow|
|---|---|---|
|屏幕旋转前|UI 订阅了，接收状态值 `"已加载"`|UI 订阅了，接收事件 `LoginSuccess`|
|屏幕旋转|ViewModel 保留，StateFlow 保留值|ViewModel 保留，但新订阅无法收到旧事件|
|屏幕旋转后|✅ 新 UI 立即收到 `"已加载"`|✅ 不重复发送 `LoginSuccess`，用户不会重复导航|

**正确模式**：

```kotlin
// 状态（屏幕旋转后要重放给新 UI）
val userProfile: StateFlow<User> = ...

// 事件（屏幕旋转后不要重放）
val navigationEvent: SharedFlow<Screen> = ...
```

---

## 推荐的深度学习路线

根据你的面试准备：

### Week 1-2：理论 + 反面例子

- 熟悉三个错误做法的具体后果
- 理解竞态条件、线程饥荒、内存泄露的原理
- 能讲清楚"为什么不能这样做"

### Week 3：设计题实战

- **购物车计算**：Repository 做折扣逻辑
- **聊天 App**：StateFlow 状态 + SharedFlow 事件
- **图片加载**：多 Dispatcher + 缓存协调

### Week 4：源码深钻

- Kotlin coroutine CPS 变换与 Dispatcher 调度
- Flow 的背压处理
- StateFlow vs SharedFlow 的内部实现

# KMP 核心架构三大支柱：面试级深度解析

---

## 第一部分：Dispatcher 隔离的意义

### 问题根源：为什么需要 Dispatcher 隔离？

#### ❌ 错误做法 1：所有操作跑在 Main Dispatcher

```kotlin
// 反面例子：直接在 Main 上做网络和 CPU 计算
class BadUserRepository {
    suspend fun fetchUserProfile(userId: String): UserProfile {
        // ⚠️ 问题 1：阻塞 Main 线程，UI 卡顿（ANR 风险）
        val response = httpClient.get("https://api.example.com/users/$userId")
        
        // ⚠️ 问题 2：JSON 解析在 Main 上，大对象时 OOM 风险
        val profile = Json.decodeFromString<UserProfile>(response.body())
        
        // ⚠️ 问题 3：数据库查询在 Main 上，违反单线程规则
        database.saveUser(profile)
        
        return profile
    }
}

// 调用端
viewModelScope.launch {
    // 默认在 Main Dispatcher 上
    val profile = repository.fetchUserProfile("123")
    // UI 冻结 3-5 秒 → ANR
}
```

**后果**：

- 📱 UI 卡顿，用户体验差
- 💣 ANR（Application Not Responding）
- 🗑️ 垃圾回收时 Main 线程停止，导致帧率掉落

---

#### ❌ 错误做法 2：所有操作跑在一个 IO Dispatcher

```kotlin
// 反面例子：所有操作共用同一个 Dispatcher
class SingleDispatcherRepository(private val io: CoroutineDispatcher = Dispatchers.IO) {
    
    private val imageCache = mutableMapOf<String, Bitmap>()
    
    suspend fun loadImage(url: String): Bitmap {
        return withContext(io) {
            // ⚠️ 问题 1：网络和图片解码都在 IO pool，竞争有限资源
            val bitmap = loadFromNetwork(url)
            
            // ⚠️ 问题 2：图片解码（CPU 密集）与网络 I/O 争夺线程
            // CPU 密集操作阻止 I/O 线程去处理其他网络请求
            val decoded = decodeBitmap(bitmap)
            
            // ⚠️ 问题 3：集合修改竞态条件（多线程访问）
            imageCache[url] = decoded
            
            decoded
        }
    }
    
    suspend fun fetchUserData(): List<User> {
        return withContext(io) {
            // 此时 IO pool 被 loadImage 的 CPU 计算占用
            // 导致这个网络请求被延迟
            apiClient.fetchUsers()
        }
    }
}

// 后果：竞态条件 + 资源争夺 + 不可预测的性能
```

**后果**：

- 🔗 线程饥荒（IO 线程被 CPU 操作占用，其他 I/O 等待）
- 🚨 竞态条件（多线程修改集合）
- 📊 性能不可预测（时快时慢）

---

### ✅ 正确做法：多 Dispatcher 隔离模型

#### 理论模型

```
┌─────────────────────────────────────────────────────────┐
│                  Coroutine Dispatcher Pool               │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │     Main     │  │      IO      │  │ Computation  │  │
│  │  (1 thread)  │  │ (8-10 threads)  │  (2-4 threads) │
│  ├──────────────┤  ├──────────────┤  ├──────────────┤  │
│  │ • UI 更新    │  │ • 网络请求   │  │ • JSON 解析  │  │
│  │ • 事件处理   │  │ • 数据库 I/O │  │ • 加密解密   │  │
│  │ • 状态绑定   │  │ • 文件操作   │  │ • 图片处理   │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

#### 深度实现：内存与线程安全的考量

```kotlin
// commonMain/kotlin/platform/CoroutineDispatchers.kt

expect class AppDispatchers {
    val main: CoroutineDispatcher
    val io: CoroutineDispatcher
    val computation: CoroutineDispatcher
}

// ============ Android 实现 ============
// androidMain/kotlin/platform/CoroutineDispatchers.kt

import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.newFixedThreadPoolDispatcher
import android.os.Looper
import android.os.Debug

actual class AppDispatchers {
    actual val main: CoroutineDispatcher = MainDispatcherChecker()
    
    /**
     * IO Dispatcher 为什么最多 64 个线程？
     * 
     * - 网络 socket：每个 HTTP 连接占 2-3KB 堆栈
     * - 文件描述符：Linux 系统默认 1024 限制
     * - 操作系统调度开销：太多线程导致 context switch
     * - Android 内存限制：低端机只有 512MB 堆
     * 
     * 计算：64 个线程 × 1MB/thread = 64MB（可接受）
     */
    actual val io: CoroutineDispatcher = Dispatchers.IO
    
    /**
     * Computation Dispatcher 为什么是 CPU 核心数？
     * 
     * - CPU 密集操作：解析 JSON、加密、图像处理
     * - 核心数 = 最优并行度（e.g., 骁龙 8 Gen2 = 8 核）
     * - 超过核心数的线程会导致时间片轮转，性能反而下降
     * 
     * 例子：8 核 CPU
     * ✅ 正确：8 个 coroutine 并行计算
     * ❌ 错误：16 个 coroutine 竞争 8 核，上下文切换开销大
     */
    actual val computation: CoroutineDispatcher = Dispatchers.Default
}

/**
 * Main Dispatcher 的内存安全包装
 * 
 * 核心问题：在 Main 线程上做同步操作会卡 UI
 * 解决方案：检测并拒绝长操作
 */
class MainDispatcherChecker : CoroutineDispatcher() {
    private val mainDispatcher = Dispatchers.Main
    
    override fun dispatch(context: CoroutineContext, block: Runnable) {
        if (Looper.getMainLooper().thread == Thread.currentThread()) {
            // 已经在 Main 线程上，直接执行
            block.run()
        } else {
            // 不在 Main 线程，切换到 Main
            mainDispatcher.dispatch(context, block)
        }
    }
}

// ============ iOS 实现 ============
// iosMain/kotlin/platform/CoroutineDispatchers.kt

import kotlinx.coroutines.newFixedThreadPoolDispatcher

actual class AppDispatchers {
    /**
     * iOS 没有 Looper，Main 线程由 Cocoa EventLoop 管理
     * 
     * - NSMainRunLoop：处理 UI 事件和触摸
     * - 必须用 DispatchQueue.main 切换
     * - Kotlin coroutine 会自动映射到 Grand Central Dispatch (GCD)
     */
    actual val main: CoroutineDispatcher = Dispatchers.Main
    
    /**
     * iOS 的 I/O 和计算分离
     * 
     * 网络请求：URLSession 用后台队列
     * 数据库：SQLite 有自己的串行队列（避免 WAL 冲突）
     */
    actual val io: CoroutineDispatcher = newFixedThreadPoolDispatcher(
        nThreads = 4,  // iOS 一般 4-6 核
        name = "ios-io"
    )
    
    actual val computation: CoroutineDispatcher = newFixedThreadPoolDispatcher(
        nThreads = 2,
        name = "ios-compute"
    )
}
```

---

### 🔍 内存安全的深层原理

#### 问题 1：竞态条件（Race Condition）

```kotlin
// ❌ 竞态条件示例
class BadImageCache {
    private val cache = mutableMapOf<String, Bitmap>()
    
    suspend fun getImage(url: String): Bitmap {
        return withContext(Dispatchers.IO) {
            // 线程 A 检查
            if (!cache.containsKey(url)) {  // Thread A 看到 key 不存在
                // ... 此时线程 B 也进来，执行相同的网络请求
                val bitmap = downloadImage(url)
                cache[url] = bitmap  // Thread B 也写入，重复下载！
            }
            cache[url]!!
        }
    }
}

// ✅ 正确做法：双检锁 + Mutex
class GoodImageCache {
    private val cache = mutableMapOf<String, Bitmap>()
    private val mutex = Mutex()
    
    suspend fun getImage(url: String): Bitmap {
        // 第一次检查：快速路径（无锁）
        cache[url]?.let { return it }
        
        // 加锁检查
        return mutex.withLock {
            // 第二次检查：防止竞态
            cache[url]?.let { return it }
            
            // 网络请求
            val bitmap = downloadImage(url)
            cache[url] = bitmap
            bitmap
        }
    }
}
```

**为什么这很关键**：

- 不同 Dispatcher 上的协程可能真正并行运行（多核）
- 多个线程同时修改 `mutableMap` 会导致数据结构损坏
- 解决方案：用 `Mutex` 或 `Actor` 保护共享状态

---

#### 问题 2：内存泄露与 GC 压力

```kotlin
// ❌ 错误的内存模型
class BadViewModel {
    private val scope = CoroutineScope(Dispatchers.Main)
    
    fun loadData() {
        // ⚠️ 问题 1：Scope 没有绑定生命周期，Activity 销毁后仍继续运行
        scope.launch {
            val data = repository.fetchData()  // 网络请求还在进行
            updateUI(data)  // 此时 Activity 已销毁！崩溃
        }
    }
    
    // onDestroy 时没有 cancel scope
}

// ✅ 正确的内存模型
class GoodViewModel(private val dispatchers: AppDispatchers) : ViewModel() {
    /**
     * viewModelScope 的原理：
     * 1. 自动绑定到 ViewModel 生命周期
     * 2. ViewModel 销毁时自动 cancel 所有 Job
     * 3. 取消传播：Job 的子协程也会被取消
     */
    private val scope = viewModelScope  // 自动绑定
    
    fun loadData() {
        scope.launch(dispatchers.io) {
            // 网络请求在 IO 线程
            val data = repository.fetchData()
            
            // 自动切换回 Main 线程更新 UI
            withContext(dispatchers.main) {
                updateUI(data)
            }
            
            // Activity 销毁时，scope.cancel() 自动调用
            // 上面的 launch 会被中断，不会有悬挂的请求
        }
    }
}
```

**内存模型的关键**：

- IO 线程池有有限资源（Socket、文件描述符）
- 长期占用 IO 线程会导致其他操作卡顿
- Dispatcher 隔离 + 正确的生命周期绑定 = 内存和资源的正确释放

---

#### 问题 3：线程局部变量与上下文传播

```kotlin
// 理解 Dispatcher 与 ThreadLocal 的交互
class DispatcherThreadLocalExample {
    private val threadLocal = ThreadLocal<String>()
    
    suspend fun demonstrateContextSwitch() {
        // 1. Main 线程设置值
        threadLocal.set("Main-Value")
        println("Main thread: ${threadLocal.get()}")  // ✅ Main-Value
        
        withContext(Dispatchers.IO) {
            // 2. 切换到 IO 线程池中的某个线程
            println("IO thread: ${threadLocal.get()}")  // ❌ null！
            // 原因：ThreadLocal 是线程隔离的，IO 线程是不同的物理线程
            
            threadLocal.set("IO-Value")
        }
        
        // 3. 回到 Main 线程
        println("Back to Main: ${threadLocal.get()}")  // ✅ Main-Value
        // 原因：Kotlin coroutine 会恢复原线程的 ThreadLocal
    }
}

// ✅ 跨 Dispatcher 传递上下文的正确方法
class ContextPropagation {
    suspend fun properContextPassing() {
        // 用 CoroutineContext 而不是 ThreadLocal
        val userId = "user-123"  // 普通变量
        
        withContext(Dispatchers.IO) {
            // userId 通过闭包捕获，不依赖 ThreadLocal
            val userData = fetchUserData(userId)
            
            withContext(Dispatchers.Computation) {
                // userId 仍然可用，因为闭包保持了引用
                processUserData(userId, userData)
            }
        }
    }
}
```

**线程安全的关键**：

- Dispatcher 切换 = 物理线程切换
- ThreadLocal 值不传播
- 用闭包捕获变量（不用 ThreadLocal）传递上下文

---

### 📊 性能影响数据

|场景|正确做法|错误做法|性能差异|
|---|---|---|---|
|1000 个网络请求 + JSON 解析|Dispatcher 隔离（IO + Computation）|全用 Main|10-100x 更快|
|Android 低端机（2GB RAM）|控制 IO 线程数 ≤ 8|无限制|不 OOM vs OOM|
|UI 响应性|网络/计算脱离 Main|阻塞 Main|60fps vs ANR|
|GC 暂停|Dispatcher 隔离减少竞争|所有操作竞争|<50ms vs >200ms|

---

## 第二部分：Repository 模式为什么必须

### 问题根源：为什么需要 Repository？

#### ❌ 错误做法 1：ViewModel 直接调用 API

```kotlin
// 反面例子：ViewModel 与网络层紧耦合
class BadUserViewModel : ViewModel() {
    private val _userData = MutableStateFlow<User?>(null)
    val userData = _userData.asStateFlow()
    
    fun loadUser(userId: String) {
        viewModelScope.launch(Dispatchers.IO) {
            try {
                // ⚠️ 问题 1：ViewModel 依赖具体的 HTTP 实现
                val httpClient = HttpClient(Android)
                val response = httpClient.get("https://api.example.com/users/$userId")
                
                // ⚠️ 问题 2：ViewModel 处理 JSON 解析
                val user = Json.decodeFromString<User>(response.body())
                
                // ⚠️ 问题 3：无缓存，每次都新建 HTTP 连接
                _userData.value = user
                
            } catch (e: Exception) {
                // ⚠️ 问题 4：业务逻辑和错误处理混在一起
                e.printStackTrace()
            }
        }
    }
}

// 测试时的噩梦
class BadUserViewModelTest {
    @Test
    fun testLoadUser() {
        // ❌ 如何 mock HTTP 客户端？需要修改 ViewModel 代码
        // ❌ 测试依赖网络环境
        // ❌ 测试不稳定（flaky）
    }
}
```

**问题清单**：

- 🔗 紧耦合：ViewModel 依赖 HTTP、JSON 库
- 🔄 无缓存：重复网络请求
- 🧪 不可测：必须用真实网络
- 📝 职责混乱：ViewModel 同时处理 UI 逻辑、网络、数据

---

#### ❌ 错误做法 2：ViewModel 中有业务逻辑

```kotlin
// 反面例子：ViewModel 包含业务规则
class BadShoppingViewModel : ViewModel() {
    
    fun checkout(cartItems: List<Item>, promoCode: String?): Boolean {
        // ⚠️ 问题 1：价格计算在 ViewModel 中
        val subtotal = cartItems.sumOf { it.price * it.quantity }
        
        // ⚠️ 问题 2：促销逻辑在 ViewModel 中
        val discount = if (promoCode == "SUMMER2024") {
            subtotal * 0.2  // 20% 折扣
        } else {
            0.0
        }
        
        // ⚠️ 问题 3：税率计算在 ViewModel 中（中国还要考虑增值税）
        val tax = (subtotal - discount) * 0.13
        
        val total = subtotal - discount + tax
        
        // ⚠️ 问题 4：支付逻辑在 ViewModel 中
        return processPayment(total)
    }
}

// 后果：
// 1. 促销规则改变需要修改 ViewModel（UI 逻辑层）
// 2. 多个 Screen/Fragment 需要相同逻辑时，代码重复
// 3. 单元测试必须测试 ViewModel，难以隔离业务逻辑
```

---

### ✅ 正确做法：Repository 模式的分层

#### 理论模型

```
┌────────────────────────────────────────────────────────────┐
│                     Presentation Layer                      │
│                   (UI / ViewModel / State)                   │
│  ✅ 职责：状态管理、UI 逻辑、事件处理                         │
│  ❌ 不做：业务规则、网络、数据库                             │
└────────────────────────────────────────────────────────────┘
                              ▲
                              │ (Flow/StateFlow)
                              │
┌────────────────────────────────────────────────────────────┐
│                    Domain / Repository Layer                │
│                  (业务逻辑 / Repository)                     │
│  ✅ 职责：业务规则、缓存策略、数据协调                       │
│  ❌ 不做：UI、HTTP 实现细节                                  │
└────────────────────────────────────────────────────────────┘
                              ▲
                              │ (接口调用)
                              │
┌────────────────────────────────────────────────────────────┐
│                    Data Source Layer                         │
│             (Remote / Local / Cache)                         │
│  ✅ 职责：网络请求、数据库操作、序列化                       │
│  ❌ 不做：业务规则、UI 更新                                  │
└────────────────────────────────────────────────────────────┘
```

#### 实例 1：电商购物车计算

```kotlin
// ============ Domain 层：业务规则 ============
// commonMain/kotlin/domain/entity/Price.kt
data class Money(val amount: Double, val currency: String)

// ============ Domain 层：业务接口 ============
// commonMain/kotlin/domain/repository/PricingRepository.kt
interface PricingRepository {
    /**
     * 计算最终价格
     * 
     * 输入：购物车项、促销码、地区
     * 输出：{ 小计、折扣、税金、总计 }
     * 
     * 业务规则集中在这里，与 UI 无关
     */
    suspend fun calculatePrice(
        items: List<CartItem>,
        promoCode: String?,
        region: String
    ): PriceBreakdown
}

// ============ Domain 层：业务实现 ============
// commonMain/kotlin/domain/repository/PricingRepositoryImpl.kt
class PricingRepositoryImpl(
    private val promotionDataSource: PromotionDataSource,
    private val taxDataSource: TaxDataSource
) : PricingRepository {
    
    override suspend fun calculatePrice(
        items: List<CartItem>,
        promoCode: String?,
        region: String
    ): PriceBreakdown {
        // 业务规则被隔离在 Repository 中
        
        // 1. 计算小计
        val subtotal = items.sumOf { it.price * it.quantity }
        
        // 2. 查询促销规则（从数据源获取，而不是硬编码）
        val discount = if (promoCode != null) {
            promotionDataSource.getDiscount(promoCode, items, region)
        } else {
            0.0
        }
        
        // 3. 查询税率（因地区而异）
        val taxRate = taxDataSource.getTaxRate(region)
        val tax = (subtotal - discount) * taxRate
        
        val total = subtotal - discount + tax
        
        return PriceBreakdown(
            subtotal = Money(subtotal, "CNY"),
            discount = Money(discount, "CNY"),
            tax = Money(tax, "CNY"),
            total = Money(total, "CNY")
        )
    }
}

// ============ ViewModel：只负责状态管理 ============
// commonMain/kotlin/ui/CartViewModel.kt
class CartViewModel(
    private val pricingRepository: PricingRepository
) : ViewModel() {
    
    private val _priceBreakdown = MutableStateFlow<PriceBreakdown?>(null)
    val priceBreakdown = _priceBreakdown.asStateFlow()
    
    fun updatePrice(items: List<CartItem>, promoCode: String?, region: String) {
        viewModelScope.launch {
            // ✅ ViewModel 只调用 Repository，不知道内部细节
            val breakdown = pricingRepository.calculatePrice(items, promoCode, region)
            _priceBreakdown.value = breakdown
        }
    }
}

// ============ 单元测试：隔离业务逻辑 ============
class PricingRepositoryTest {
    
    private val mockPromoDataSource = MockPromotionDataSource()
    private val mockTaxDataSource = MockTaxDataSource()
    private val repository = PricingRepositoryImpl(mockPromoDataSource, mockTaxDataSource)
    
    @Test
    fun testSummerPromoAppliesCorrectDiscount() = runBlocking {
        // 不需要 UI 环境，直接测试业务逻辑
        mockPromoDataSource.setDiscount("SUMMER2024", 0.2)  // 20% 折扣
        mockTaxDataSource.setTaxRate(0.13)  // 13% 税
        
        val items = listOf(CartItem("Item1", 100.0, 2))
        val breakdown = repository.calculatePrice(items, "SUMMER2024", "CN")
        
        assertEquals(200.0, breakdown.subtotal.amount)  // 100 * 2
        assertEquals(40.0, breakdown.discount.amount)   // 200 * 0.2
        assertEquals(20.8, breakdown.tax.amount)        // (200 - 40) * 0.13
        assertEquals(180.8, breakdown.total.amount)
    }
    
    @Test
    fun testNoPromoCodeMeansNoDiscount() = runBlocking {
        val items = listOf(CartItem("Item1", 100.0, 1))
        val breakdown = repository.calculatePrice(items, null, "CN")
        
        assertEquals(0.0, breakdown.discount.amount)
    }
}

// ============ ViewModel 测试：验证 UI 行为 ============
class CartViewModelTest {
    
    private val mockRepository = MockPricingRepository()
    private val viewModel = CartViewModel(mockRepository)
    
    @Test
    fun testPriceUpdatesWhenPromoCodeChanges() = runBlocking {
        val items = listOf(CartItem("Item1", 100.0, 1))
        
        mockRepository.setBreakdown(
            PriceBreakdown(Money(100.0, "CNY"), Money(0.0, "CNY"), ...)
        )
        
        viewModel.updatePrice(items, null, "CN")
        
        // ✅ 只验证 ViewModel 的状态管理职责，不测试计算逻辑
        assertEquals(100.0, viewModel.priceBreakdown.value?.total?.amount)
    }
}
```

**对比**：

- ❌ 错误做法：ViewModel 中硬编码促销规则 → 需要改 UI 层代码
- ✅ 正确做法：业务规则在 Repository → 改一个地方就行

---

#### 实例 2：缓存策略与数据协调

```kotlin
// ============ ViewModel 直接调用 API 的问题 ============
class BadRefreshViewModel : ViewModel() {
    fun refreshUserData(userId: String) {
        viewModelScope.launch(Dispatchers.IO) {
            // ❌ 问题 1：每次都新建 HTTP 连接
            val user = apiClient.getUser(userId)
            
            // ❌ 问题 2：重复的网络请求
            // 如果用户快速点击"刷新"2 次，会发 2 个网络请求
            
            // ❌ 问题 3：没有本地缓存降级
            // 网络失败时，UI 一片空白
        }
    }
}

// ============ Repository 的缓存管理 ============
// commonMain/kotlin/domain/repository/UserRepository.kt
class UserRepositoryImpl(
    private val remoteDataSource: UserRemoteDataSource,
    private val localDataSource: UserLocalDataSource,
    private val dispatchers: AppDispatchers
) : UserRepository {
    
    /**
     * 缓存策略的三层模型：
     * 
     * L1: 内存 StateFlow（热缓存，速度最快）
     * L2: 本地数据库（温缓存，恢复速度快）
     * L3: 远程 API（冷缓存，网络请求）
     */
    private val _userCache = MutableStateFlow<Map<String, User>>(emptyMap())
    
    override fun observeUser(userId: String): Flow<ApiResult<User>> {
        return _userCache
            .map { cache ->
                cache[userId]?.let { ApiResult.Success(it) }
                    ?: ApiResult.Loading(null)
            }
            .distinctUntilChanged()
            .flowOn(dispatchers.main)
    }
    
    override suspend fun refreshUser(userId: String): ApiResult<User> {
        return withContext(dispatchers.io) {
            try {
                // 1. 先检查内存缓存（完全不阻塞 UI）
                _userCache.value[userId]?.let {
                    // 后台更新，不阻塞返回
                    refreshInBackground(userId)
                    return@withContext ApiResult.Success(it)
                }
                
                // 2. 再查本地数据库
                val cached = localDataSource.getUser(userId)
                if (cached != null) {
                    _userCache.update { it + (userId to cached) }
                    // 后台拉最新数据
                    refreshInBackground(userId)
                    return@withContext ApiResult.Success(cached)
                }
                
                // 3. 最后才网络请求
                val remote = remoteDataSource.getUser(userId)
                _userCache.update { it + (userId to remote) }
                localDataSource.saveUser(remote)
                
                ApiResult.Success(remote)
                
            } catch (e: Exception) {
                // 网络失败时尝试返回本地缓存
                val cached = localDataSource.getUser(userId)
                if (cached != null) {
                    ApiResult.Error(e, retryable = true, cachedData = cached)
                } else {
                    ApiResult.Error(e, retryable = true)
                }
            }
        }
    }
    
    private suspend fun refreshInBackground(userId: String) {
        try {
            val remote = remoteDataSource.getUser(userId)
            _userCache.update { it + (userId to remote) }
            localDataSource.saveUser(remote)
        } catch (e: Exception) {
            // 静默失败，已有缓存就行
            Log.w("UserRepository", "Background refresh failed", e)
        }
    }
}

// ============ ViewModel 使用 Repository ============
class GoodRefreshViewModel(
    private val userRepository: UserRepository
) : ViewModel() {
    
    val userState = userRepository.observeUser("user-123")
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(),
            initialValue = ApiResult.Loading()
        )
    
    fun refresh() {
        viewModelScope.launch {
            userRepository.refreshUser("user-123")
            // ✅ Repository 已处理缓存协调
            // ✅ 重复刷新时能利用缓存
            // ✅ 网络失败时有本地数据
        }
    }
}
```

**收益**：

- 📱 用户快速点击"刷新" → 只发 1 个网络请求（去重）
- 🚀 首次进入 → 显示本地缓存（快速），后台更新
- 📡 网络失败 → 显示旧数据（降级），而不是空白
- 🧪 完全可测试 → mock 数据源即可

---

### 📋 Repository 模式的核心职责

|职责|为什么重要|示例|
|---|---|---|
|**数据协调**|选择数据源优先级|内存 > 本地 > 远程|
|**缓存管理**|减少网络请求|去重、过期时间|
|**业务规则**|逻辑集中管理|促销折扣、税率计算|
|**错误恢复**|提升容错性|降级到本地缓存|
|**单一职责**|便于测试|mock 数据源，测试业务逻辑|

---

## 第三部分：StateFlow vs SharedFlow 的选型

### 问题根源：为什么有两个 Flow？

```kotlin
// 理解两者的本质区别
// StateFlow: 有状态的值流（当前值可查）
// SharedFlow: 无状态的事件流（纯粹的事件广播）

// ❌ 错误用法 1：用 SharedFlow 存储状态
class BadCounterViewModel {
    private val _counterEvents = MutableSharedFlow<Int>()
    val counterEvents = _counterEvents.asSharedFlow()
    
    fun increment() {
        // 问题 1：新订阅者收不到当前计数值
        // 问题 2：需要手动管理"最后一个值"
        viewModelScope.launch {
            _counterEvents.emit(counter + 1)
        }
    }
    
    // 订阅端
    val currentCounter = counterEvents
        .scan(0) { acc, _ -> acc + 1 }  // 手动管理状态，容易出错
}

// ❌ 错误用法 2：用 StateFlow 做事件通知
class BadLoginViewModel {
    private val _loginEvent = MutableStateFlow<LoginEvent?>(null)
    val loginEvent = _loginEvent.asStateFlow()
    
    fun login() {
        viewModelScope.launch {
            _loginEvent.value = LoginEvent.Success("token")
        }
    }
    
    // 订阅端
    val eventObserver = loginEvent
        .filterNotNull()
        .collect { event ->
            when (event) {
                is LoginEvent.Success -> navigateToHome()
                // ❌ 问题 1：旋转屏幕，LoginEvent 被重新发送（重复导航）
                // ❌ 问题 2：必须手动清空值（设为 null），容易忘记
            }
        }
}
```

---

### ✅ 正确选型指南

#### 场景 1：UI 状态（用 StateFlow）

```kotlin
// ✅ StateFlow 的最佳用途：表示当前状态
class UserProfileViewModel(
    private val userRepository: UserRepository
) : ViewModel() {
    
    /**
     * StateFlow 特性：
     * - 有当前值 (value 属性)
     * - 新订阅者立即收到当前值
     * - 值相同时不发射（智能去重）
     */
    val userProfile: StateFlow<ApiResult<UserProfile>> = 
        userRepository.observeUserProfile("user-123")
            .stateIn(
                scope = viewModelScope,
                started = SharingStarted.WhileSubscribed(5000),
                initialValue = ApiResult.Loading()
            )
    
    val isLoading: StateFlow<Boolean> = userProfile
        .map { it is ApiResult.Loading }
        .distinctUntilChanged()
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.Lazily,
            initialValue = false
        )
}

// UI 层
@Composable
fun UserProfileScreen(viewModel: UserProfileViewModel) {
    val profile by viewModel.userProfile.collectAsState()
    val isLoading by viewModel.isLoading.collectAsState()
    
    // ✅ 每次屏幕重组都能立即得到当前状态
    // ✅ 不需要 viewModel 来维护状态
}
```

**为什么 StateFlow 适合状态**：

- 订阅者需要立即得到当前值（UI 初始化）
- 值相同时不重复发射（避免不必要的重组）
- 支持 `.value` 直接读取（某些场景需要）

---

#### 场景 2：一次性事件（用 SharedFlow）

```kotlin
// ✅ SharedFlow 的最佳用途：事件通知
class AuthViewModel : ViewModel() {
    
    /**
     * SharedFlow 特性：
     * - 没有当前值（纯事件）
     * - 新订阅者收不到之前的事件
     * - 每个订阅者各自接收事件
     */
    private val _loginEvent = MutableSharedFlow<LoginEvent>()
    val loginEvent: SharedFlow<LoginEvent> = _loginEvent.asSharedFlow()
    
    fun login(username: String, password: String) {
        viewModelScope.launch {
            try {
                val token = authRepository.login(username, password)
                // 发送事件，不修改状态
                _loginEvent.emit(LoginEvent.Success(token))
            } catch (e: Exception) {
                _loginEvent.emit(LoginEvent.Error(e.message ?: "Unknown error"))
            }
        }
    }
}

// UI 层：使用事件
class LoginFragment : Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        // ✅ 每个订阅只处理一次事件
        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.loginEvent.collect { event ->
                    when (event) {
                        is LoginEvent.Success -> {
                            // ✅ 屏幕旋转后不会重复导航
                            navigateToHome()
                        }
                        is LoginEvent.Error -> {
                            showErrorDialog(event.message)
                        }
                    }
                }
            }
        }
    }
}
```

**为什么 SharedFlow 适合事件**：

- 事件一般不需要重放（不需要当前值）
- 多个订阅者都要收到通知
- 屏幕旋转不应该重复触发事件

---

#### 场景 3：混合模型（StateFlow + SharedFlow）

```kotlin
// 真实场景：既需要状态，也需要事件
class UserListViewModel(
    private val userRepository: UserRepository
) : ViewModel() {
    
    // 状态：列表当前数据
    val users: StateFlow<ApiResult<List<User>>> = 
        userRepository.observeUsers()
            .stateIn(
                scope = viewModelScope,
                started = SharingStarted.WhileSubscribed(),
                initialValue = ApiResult.Loading()
            )
    
    // 事件：用户被删除
    private val _userDeletedEvent = MutableSharedFlow<User>()
    val userDeletedEvent: SharedFlow<User> = _userDeletedEvent.asSharedFlow()
    
    fun deleteUser(userId: String) {
        viewModelScope.launch {
            try {
                val user = userRepository.deleteUser(userId)
                // 发送事件（一次性通知）
                _userDeletedEvent.emit(user)
                
                // 列表状态会通过 userRepository 自动更新
                // （Repository 中的 StateFlow 会发射新列表）
            } catch (e: Exception) {
                // 错误处理...
            }
        }
    }
}

// UI 层
@Composable
fun UserListScreen(viewModel: UserListViewModel) {
    val users by viewModel.users.collectAsState()
    
    LaunchedEffect(Unit) {
        // 监听删除事件（一次性，用于 Toast/Snackbar）
        viewModel.userDeletedEvent.collect { deletedUser ->
            showSnackbar("删除用户 ${deletedUser.name} 成功")
        }
    }
    
    // 显示用户列表（响应式状态）
    LazyColumn {
        items(
            items = (users as? ApiResult.Success)?.data ?: emptyList(),
            key = { it.id }
        ) { user ->
            UserItem(user, onDelete = { viewModel.deleteUser(it.id) })
        }
    }
}
```

---

### 📊 完整对比表

|特性|StateFlow|SharedFlow|
|---|---|---|
|**有当前值**|✅ 是|❌ 否|
|**新订阅者收到当前值**|✅ 是|❌ 否|
|**值不变时重新发射**|❌ 否|✅ 是|
|**多播到多个订阅者**|✅ 是|✅ 是|
|**典型使用**|UI 状态、ViewModel 数据|一次性事件、通知|
|**屏幕旋转安全**|✅ 自动保留状态|⚠️ 需小心不重复触发|
|**初始值**|必须提供|可选|
|**内存占用**|低（只存当前值）|低（可配置 buffer）|

---

### 🔧 高级用法：SharedFlow 的 replay

```kotlin
// SharedFlow 也可以重放历史事件（类似 StateFlow）
class AnalyticsViewModel : ViewModel() {
    
    /**
     * replay = 10：保留最后 10 个事件
     * 新订阅者可以立即收到这 10 个事件
     * 
     * 场景：用户分析
     */
    private val _analyticsEvents = MutableSharedFlow<AnalyticsEvent>(
        replay = 10,
        onBufferOverflow = BufferOverflow.DROP_OLDEST
    )
    val analyticsEvents = _analyticsEvents.asSharedFlow()
    
    fun trackEvent(event: AnalyticsEvent) {
        viewModelScope.launch {
            _analyticsEvents.emit(event)
        }
    }
}

// 对比：什么时候用 replay？
// ❌ 不要：用 replay 模仿 StateFlow，应该直接用 StateFlow
// ✅ 应该：追踪最近的 N 个事件，用于分析或调试
```

---

### 🎯 面试题：现场设计题

**题目**：设计一个实时聊天应用的 ViewModel，支持：

1. 显示消息列表（当前状态）
2. 显示 3 秒后自动隐藏的消息提示（一次性事件）
3. 对方正在输入的状态（实时更新）

```kotlin
// ✅ 标准答案
class ChatViewModel(private val chatRepository: ChatRepository) : ViewModel() {
    
    // 1. 消息列表状态（StateFlow）
    val messages: StateFlow<List<Message>> = chatRepository.observeMessages()
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(),
            initialValue = emptyList()
        )
    
    // 2. 消息提示事件（SharedFlow 无 replay）
    private val _messageToast = MutableSharedFlow<String>()
    val messageToast: SharedFlow<String> = _messageToast.asSharedFlow()
    
    // 3. 对方输入状态（StateFlow）
    val otherUserTyping: StateFlow<Boolean> = chatRepository.observeTypingStatus()
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(),
            initialValue = false
        )
    
    fun sendMessage(content: String) {
        viewModelScope.launch {
            try {
                chatRepository.sendMessage(content)
                _messageToast.emit("消息已发送")
            } catch (e: Exception) {
                _messageToast.emit("发送失败: ${e.message}")
            }
        }
    }
}
```

**回答关键点**：

- ✅ 状态用 StateFlow（消息列表、输入指示）
- ✅ 事件用 SharedFlow（Toast 通知）
- ✅ 解释为什么这样设计（状态需要当前值，事件不需要）
- ✅ 提到屏幕旋转安全性

---

## 总结对比表

|概念|核心目标|关键点|
|---|---|---|
|**Dispatcher 隔离**|性能 + 内存安全|Main 处理 UI，IO 处理网络，Computation 处理计算|
|**Repository 模式**|解耦 + 可测试|隐藏实现细节，集中业务逻辑，支持缓存|
|**StateFlow vs SharedFlow**|正确建模|状态用 SF，事件用 SHF|