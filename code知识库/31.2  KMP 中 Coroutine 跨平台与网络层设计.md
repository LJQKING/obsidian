
## 1. 核心问题与设计思想

### 跨平台 Coroutine 的挑战

- **Main Dispatcher 差异**：Android 有 Looper，iOS 没有
- **线程池配置**：不同平台的最优线程数不同
- **性能监控**：需要平台差异化的采样方案
- **取消传播**：Job 在不同平台的 GC 表现不同

### 网络层设计目标

1. **解耦**：业务逻辑和 HTTP 实现分离
2. **可测试性**：支持 mock 和 fake 实现
3. **恢复力**：重试、超时、断路器
4. **可观测性**：请求链路跟踪

---

## 2. Dispatcher 跨平台设计（Expect/Actual）

### 2.1 定义 expect interface

```kotlin
// commonMain/kotlin/platform/CoroutineDispatchers.kt
expect class AppDispatchers {
    val main: CoroutineDispatcher
    val io: CoroutineDispatcher
    val computation: CoroutineDispatcher
    val background: CoroutineDispatcher
}

expect fun createAppDispatchers(): AppDispatchers
```

### 2.2 Android 实现

```kotlin
// androidMain/kotlin/platform/CoroutineDispatchers.kt
import androidx.lifecycle.DefaultLifecycleObserver
import androidx.lifecycle.ProcessLifecycleOwner
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.MainCoroutineDispatcher
import kotlinx.coroutines.newFixedThreadPoolDispatcher

actual class AppDispatchers(
    actual val main: CoroutineDispatcher = Dispatchers.Main,
    actual val io: CoroutineDispatcher = Dispatchers.IO,
    // 计算密集型：CPU 核心数相关
    actual val computation: CoroutineDispatcher = Dispatchers.Default,
    // 后台低优先级任务
    actual val background: CoroutineDispatcher = newFixedThreadPoolDispatcher(
        nThreads = 2,
        name = "app-background"
    )
)

actual fun createAppDispatchers(): AppDispatchers = AppDispatchers()

// 内存管理感知的 dispatcher 适配器
class MemoryAwareDispatcher(
    private val primary: CoroutineDispatcher,
    private val fallback: CoroutineDispatcher
) : CoroutineDispatcher() {
    override fun dispatch(context: CoroutineContext, block: Runnable) {
        val runtime = Runtime.getRuntime()
        val freeMemory = runtime.freeMemory()
        val maxMemory = runtime.maxMemory()
        val usagePercent = (maxMemory - freeMemory) * 100 / maxMemory
        
        // 内存压力 > 80% 时降级到 fallback
        val dispatcher = if (usagePercent > 80) fallback else primary
        dispatcher.dispatch(context, block)
    }
}
```

### 2.3 iOS 实现

```kotlin
// iosMain/kotlin/platform/CoroutineDispatchers.kt
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.newFixedThreadPoolDispatcher
import platform.Foundation.NSOperationQueue
import platform.Foundation.dispatch_get_main_queue
import platform.Foundation.dispatch_queue_create

actual class AppDispatchers(
    // iOS Main 不能用 Dispatchers.Main（需要 UI 线程）
    actual val main: CoroutineDispatcher = Dispatchers.Main,
    // iOS 的 GCD 默认优先级队列
    actual val io: CoroutineDispatcher = newFixedThreadPoolDispatcher(
        nThreads = 4,
        name = "ios-io-queue"
    ),
    actual val computation: CoroutineDispatcher = newFixedThreadPoolDispatcher(
        nThreads = 2,
        name = "ios-compute-queue"
    ),
    actual val background: CoroutineDispatcher = newFixedThreadPoolDispatcher(
        nThreads = 1,
        name = "ios-background-queue"
    )
)

actual fun createAppDispatchers(): AppDispatchers = AppDispatchers()
```

---

## 3. 网络层设计（分层架构）

### 3.1 Domain 层（平台无关）

```kotlin
// commonMain/kotlin/domain/model/ApiResponse.kt
sealed class ApiResult<out T> {
    data class Success<T>(val data: T) : ApiResult<T>()
    data class Error(val exception: Exception, val retryable: Boolean) : ApiResult<Nothing>()
    data class Loading<T>(val cached: T? = null) : ApiResult<T>()
}

// 业务模型
data class UserProfile(
    val id: String,
    val name: String,
    val avatar: String
)

// 通用的 API 返回包装
data class ApiWrapper<T>(
    val code: Int,
    val message: String,
    val data: T
)
```

### 3.2 Repository 层（定义 expect contract）

```kotlin
// commonMain/kotlin/domain/repository/UserRepository.kt
interface UserRepository {
    suspend fun fetchUserProfile(userId: String): ApiResult<UserProfile>
    fun observeUserProfile(userId: String): Flow<ApiResult<UserProfile>>
}

// Platform-independent implementation with caching
class UserRepositoryImpl(
    private val remoteDataSource: UserRemoteDataSource,
    private val localDataSource: UserLocalDataSource,
    private val dispatchers: AppDispatchers
) : UserRepository {
    
    // 使用 StateFlow 管理跨平台状态
    private val _profileCache = MutableStateFlow<Map<String, UserProfile>>(emptyMap())
    
    override suspend fun fetchUserProfile(userId: String): ApiResult<UserProfile> {
        return withContext(dispatchers.io) {
            try {
                // 1. 尝试本地缓存
                _profileCache.value[userId]?.let {
                    return@withContext ApiResult.Success(it)
                }
                
                // 2. 从远程获取
                val remote = remoteDataSource.getUserProfile(userId)
                
                // 3. 更新缓存
                _profileCache.update { map ->
                    map + (userId to remote)
                }
                
                // 4. 写入本地数据源
                localDataSource.saveUserProfile(remote)
                
                ApiResult.Success(remote)
            } catch (e: Exception) {
                // 重试逻辑由 DataSource 层处理
                ApiResult.Error(e, retryable = isRetryable(e))
            }
        }
    }
    
    override fun observeUserProfile(userId: String): Flow<ApiResult<UserProfile>> {
        return _profileCache
            .map { cache ->
                cache[userId]?.let { ApiResult.Success(it) } 
                    ?: ApiResult.Loading(null)
            }
            .distinctUntilChanged()
            .flowOn(dispatchers.io)
    }
    
    private fun isRetryable(e: Exception): Boolean {
        return e is IOException || // 网络错误
               (e is HttpException && e.code in 500..599) // 服务器错误
    }
}
```

### 3.3 Data Source 层（平台差异化）

```kotlin
// commonMain/kotlin/data/remote/UserRemoteDataSource.kt
interface UserRemoteDataSource {
    suspend fun getUserProfile(userId: String): UserProfile
}

expect class HttpClientFactory {
    fun createClient(): HttpClient
}

class UserRemoteDataSourceImpl(
    private val httpClient: HttpClient,
    private val dispatchers: AppDispatchers
) : UserRemoteDataSource {
    
    override suspend fun getUserProfile(userId: String): UserProfile {
        return withContext(dispatchers.io) {
            httpClient.get("https://api.example.com/users/$userId") {
                contentType(ContentType.Application.Json)
                timeout {
                    requestTimeoutMillis = 15000
                    connectTimeoutMillis = 10000
                    socketTimeoutMillis = 10000
                }
            }.body<ApiWrapper<UserProfile>>().data
        }
    }
}
```

---

## 4. 重试与断路器策略

### 4.1 指数退避重试

```kotlin
// commonMain/kotlin/data/network/RetryPolicy.kt
class ExponentialBackoffRetry(
    private val maxRetries: Int = 3,
    private val initialDelayMs: Long = 100,
    private val maxDelayMs: Long = 30000
) {
    suspend fun <T> execute(block: suspend () -> T): T {
        var lastException: Exception? = null
        
        repeat(maxRetries) { attempt ->
            try {
                return block()
            } catch (e: Exception) {
                lastException = e
                
                if (attempt < maxRetries - 1) {
                    val delayMs = (initialDelayMs * (1 shl attempt))
                        .coerceAtMost(maxDelayMs)
                    delay(delayMs)
                }
            }
        }
        
        throw lastException ?: Exception("Unknown error")
    }
}

// 使用示例
suspend fun getUserWithRetry(userId: String): UserProfile {
    return ExponentialBackoffRetry().execute {
        userRemoteDataSource.getUserProfile(userId)
    }
}
```

### 4.2 断路器模式

```kotlin
// commonMain/kotlin/data/network/CircuitBreaker.kt
enum class CircuitBreakerState {
    CLOSED,      // 正常
    OPEN,        // 熔断（拒绝请求）
    HALF_OPEN    // 半开（允许一个请求测试恢复）
}

class CircuitBreaker(
    private val failureThreshold: Int = 5,
    private val successThreshold: Int = 2,
    private val timeoutMs: Long = 60000
) {
    private var state = CircuitBreakerState.CLOSED
    private var failureCount = 0
    private var successCount = 0
    private var lastFailureTime = 0L
    
    suspend fun <T> execute(block: suspend () -> T): T {
        when (state) {
            CircuitBreakerState.OPEN -> {
                if (System.currentTimeMillis() - lastFailureTime > timeoutMs) {
                    state = CircuitBreakerState.HALF_OPEN
                    successCount = 0
                } else {
                    throw CircuitBreakerOpenException()
                }
            }
            else -> {}
        }
        
        return try {
            block().also {
                onSuccess()
            }
        } catch (e: Exception) {
            onFailure()
            throw e
        }
    }
    
    private fun onSuccess() {
        failureCount = 0
        
        if (state == CircuitBreakerState.HALF_OPEN) {
            successCount++
            if (successCount >= successThreshold) {
                state = CircuitBreakerState.CLOSED
            }
        }
    }
    
    private fun onFailure() {
        lastFailureTime = System.currentTimeMillis()
        failureCount++
        
        if (failureCount >= failureThreshold) {
            state = CircuitBreakerState.OPEN
        }
    }
}

class CircuitBreakerOpenException : Exception("Circuit breaker is open")
```

---

## 5. Platform-Specific HTTP Client Factory

### 5.1 Android HttpClient

```kotlin
// androidMain/kotlin/data/network/HttpClientFactory.kt
import io.ktor.client.*
import io.ktor.client.engine.okhttp.*
import okhttp3.OkHttpClient
import java.util.concurrent.TimeUnit

actual class HttpClientFactory {
    actual fun createClient(): HttpClient = HttpClient(OkHttp) {
        engine {
            // 配置 OkHttp 连接池
            preconfigured = OkHttpClient.Builder()
                .connectTimeout(10, TimeUnit.SECONDS)
                .readTimeout(10, TimeUnit.SECONDS)
                .writeTimeout(10, TimeUnit.SECONDS)
                .connectionPool(okhttp3.ConnectionPool(8, 5, TimeUnit.MINUTES))
                .build()
        }
        
        install(HttpTimeout) {
            requestTimeoutMillis = 15000
            connectTimeoutMillis = 10000
            socketTimeoutMillis = 10000
        }
        
        // Android 特定：添加证书钉钉（certificate pinning）
        install(HttpClientDefaultRequest) {
            header("User-Agent", "MyApp-Android/1.0")
        }
    }
}
```

### 5.2 iOS HttpClient

```kotlin
// iosMain/kotlin/data/network/HttpClientFactory.kt
import io.ktor.client.*
import io.ktor.client.engine.darwin.*

actual class HttpClientFactory {
    actual fun createClient(): HttpClient = HttpClient(Darwin) {
        engine {
            // iOS 使用 NSURLSession
            configureSession {
                sessionConfiguration = NSURLSessionConfiguration.defaultSessionConfiguration()
                sessionConfiguration?.timeoutIntervalForRequest = 15.0
                sessionConfiguration?.waitsForConnectivity = true
            }
        }
        
        // iOS 特定：支持后台传输
        install(HttpTimeout) {
            requestTimeoutMillis = 15000
            connectTimeoutMillis = 10000
            socketTimeoutMillis = 10000
        }
    }
}
```

---

## 6. 完整使用流程

### 6.1 ViewModel 层（业务逻辑）

```kotlin
// commonMain/kotlin/ui/UserProfileViewModel.kt
class UserProfileViewModel(
    private val userRepository: UserRepository,
    private val dispatchers: AppDispatchers
) : ViewModel() {
    
    private val _userId = MutableStateFlow<String?>(null)
    
    // 暴露给 UI 的 State
    val userProfile: Flow<ApiResult<UserProfile>> = _userId
        .filterNotNull()
        .flatMapLatest { userId ->
            userRepository.observeUserProfile(userId)
        }
        .flowOn(dispatchers.main)
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = ApiResult.Loading()
        )
    
    fun loadUserProfile(userId: String) {
        _userId.value = userId
        
        viewModelScope.launch(dispatchers.io) {
            userRepository.fetchUserProfile(userId)
        }
    }
    
    fun retry() {
        _userId.value?.let { loadUserProfile(it) }
    }
}
```

### 6.2 UI 层（Compose/SwiftUI）

```kotlin
// commonMain/kotlin/ui/UserProfileScreen.kt
@Composable
fun UserProfileScreen(viewModel: UserProfileViewModel) {
    val profile by viewModel.userProfile.collectAsState(ApiResult.Loading())
    
    Box(modifier = Modifier.fillMaxSize()) {
        when (profile) {
            is ApiResult.Loading -> {
                CircularProgressIndicator(modifier = Modifier.align(Alignment.Center))
            }
            is ApiResult.Success -> {
                val data = (profile as ApiResult.Success).data
                Column(modifier = Modifier.fillMaxSize().padding(16.dp)) {
                    Text("用户: ${data.name}")
                    AsyncImage(model = data.avatar, contentDescription = null)
                }
            }
            is ApiResult.Error -> {
                val error = profile as ApiResult.Error
                Column(modifier = Modifier.align(Alignment.Center)) {
                    Text("加载失败: ${error.exception.message}")
                    if (error.retryable) {
                        Button(onClick = { viewModel.retry() }) {
                            Text("重试")
                        }
                    }
                }
            }
        }
    }
}
```

---

## 7. 关键设计要点总结

### Dispatcher 隔离

- `main`：UI 更新
- `io`：网络 I/O
- `computation`：CPU 密集
- `background`：低优先级后台任务

### 网络层分层

```
UI (Compose/SwiftUI)
  ↓
ViewModel (StateFlow)
  ↓
Repository (业务逻辑 + 缓存)
  ↓
RemoteDataSource (网络请求)
  ↓
HttpClient (平台实现)
```

### 跨平台协调

- **Expect/Actual** 处理平台差异（Dispatcher、HttpClient）
- **Flow/StateFlow** 统一异步状态管理
- **suspend function** 统一异步接口

### 恢复力设计

- 指数退避重试
- 断路器防雪崩
- 本地缓存降级
- 可区分重试与非重试错误