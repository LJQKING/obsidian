
## 第四部分：网络通信（★★★★☆）

### Q7: 如何整合Retrofit + RxJava + Dagger2？实现token刷新机制？

#### 1️⃣ **架构设计**

```
┌─────────────────────────────────┐
│ 业务层 (Repository)             │
│ - 提供业务接口                   │
│ - 处理数据转换                   │
└──────────────┬──────────────────┘
               ↓
┌─────────────────────────────────┐
│ 网络层 (ApiService + Interceptor) │
│ - HTTP请求                      │
│ - token管理                      │
│ - 错误处理                       │
└──────────────┬──────────────────┘
               ↓
┌─────────────────────────────────┐
│ 依赖注入层 (Hilt/Dagger2)        │
│ - 生命周期管理                   │
│ - 单例控制                       │
└─────────────────────────────────┘
```

#### 2️⃣ **完整代码实现**

```kotlin
// ============ 1️⃣ API接口定义 ============
interface ApiService {
    @GET("/users/{id}")
    suspend fun getUser(@Path("id") id: String): Response<User>
    
    @POST("/login")
    suspend fun login(@Body request: LoginRequest): Response<LoginResponse>
    
    @POST("/refresh")
    suspend fun refreshToken(@Body request: RefreshRequest): Response<TokenResponse>
}

// ============ 2️⃣ 请求/响应数据类 ============
data class LoginRequest(
    val email: String,
    val password: String
)

data class LoginResponse(
    val accessToken: String,
    val refreshToken: String,
    val expiresIn: Long
)

data class TokenResponse(
    val accessToken: String,
    val expiresIn: Long
)

data class User(
    val id: String,
    val name: String,
    val email: String
)

// ============ 3️⃣ Token管理 ============
class TokenManager @Inject constructor() {
    private var accessToken: String? = null
    private var refreshToken: String? = null
    private var expiresAt: Long = 0
    
    @Synchronized
    fun setTokens(access: String, refresh: String, expiresIn: Long) {
        this.accessToken = access
        this.refreshToken = refresh
        // 提前5分钟过期
        this.expiresAt = System.currentTimeMillis() + (expiresIn - 300) * 1000
    }
    
    @Synchronized
    fun getAccessToken(): String? = accessToken
    
    @Synchronized
    fun getRefreshToken(): String? = refreshToken
    
    @Synchronized
    fun isTokenExpired(): Boolean {
        return System.currentTimeMillis() > expiresAt
    }
    
    @Synchronized
    fun clearTokens() {
        accessToken = null
        refreshToken = null
        expiresAt = 0
    }
}

// ============ 4️⃣ 网络拦截器链 ============

// 拦截器1：添加Token
class AuthInterceptor @Inject constructor(
    private val tokenManager: TokenManager
) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val original = chain.request()
        
        // 添加Authorization头
        val token = tokenManager.getAccessToken()
        val requestBuilder = original.newBuilder()
        if (token != null) {
            requestBuilder.addHeader("Authorization", "Bearer $token")
        }
        
        return chain.proceed(requestBuilder.build())
    }
}

// 拦截器2：处理401错误（token过期）
class TokenRefreshInterceptor @Inject constructor(
    private val apiService: ApiService,
    private val tokenManager: TokenManager
) : Interceptor {
    
    override fun intercept(chain: Interceptor.Chain): Response {
        var response = chain.proceed(chain.request())
        
        // 如果401（未授权）
        if (response.code == 401) {
            synchronized(this) {  // 同步，防止多个线程同时刷新
                // 再检查一次（double-check）
                val refreshToken = tokenManager.getRefreshToken()
                if (refreshToken != null && !tokenManager.isTokenExpired()) {
                    // 尝试刷新token
                    val newToken = refreshAccessToken(refreshToken)
                    if (newToken != null) {
                        // 成功刷新，重试原请求
                        response.close()
                        
                        val retryRequest = chain.request()
                            .newBuilder()
                            .header("Authorization", "Bearer $newToken")
                            .build()
                        return chain.proceed(retryRequest)
                    }
                }
            }
            // 刷新失败，清除token，返回401
            tokenManager.clearTokens()
        }
        
        return response
    }
    
    private fun refreshAccessToken(refreshToken: String): String? {
        return try {
            val request = RefreshRequest(refreshToken)
            val response = apiService.refreshToken(request)  // 同步调用
            
            if (response.isSuccessful) {
                val tokenResponse = response.body()?.data
                if (tokenResponse != null) {
                    tokenManager.setTokens(
                        tokenResponse.accessToken,
                        refreshToken,
                        tokenResponse.expiresIn
                    )
                    return tokenResponse.accessToken
                }
            }
            null
        } catch (e: Exception) {
            null
        }
    }
}

// 拦截器3：日志记录
class LoggingInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request()
        val startTime = System.currentTimeMillis()
        
        Log.d("API", "→ 请求: ${request.method} ${request.url}")
        request.headers.forEach { (name, value) ->
            Log.d("API", "  $name: $value")
        }
        
        val response = chain.proceed(request)
        val duration = System.currentTimeMillis() - startTime
        
        Log.d("API", "← 响应: ${response.code} (耗时: ${duration}ms)")
        return response
    }
}

// ============ 5️⃣ Dagger2 Module配置 ============
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    
    @Provides
    @Singleton
    fun provideTokenManager(): TokenManager = TokenManager()
    
    @Provides
    @Singleton
    fun provideOkHttpClient(
        authInterceptor: AuthInterceptor,
        tokenRefreshInterceptor: TokenRefreshInterceptor,
        loggingInterceptor: LoggingInterceptor
    ): OkHttpClient {
        return OkHttpClient.Builder()
            .addInterceptor(loggingInterceptor)      // 最先执行
            .addInterceptor(authInterceptor)         // 添加token
            .addInterceptor(tokenRefreshInterceptor) // 处理401
            .connectTimeout(30, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .writeTimeout(30, TimeUnit.SECONDS)
            .build()
    }
    
    @Provides
    @Singleton
    fun provideRetrofit(okHttpClient: OkHttpClient): Retrofit {
        return Retrofit.Builder()
            .baseUrl("https://api.example.com/")
            .client(okHttpClient)
            .addConverterFactory(GsonConverterFactory.create())
            .addCallAdapterFactory(RxJava3CallAdapterFactory.create())  // RxJava适配
            .build()
    }
    
    @Provides
    @Singleton
    fun provideApiService(retrofit: Retrofit): ApiService {
        return retrofit.create(ApiService::class.java)
    }
}

// ============ 6️⃣ 业务层 Repository ============
@Singleton
class UserRepository @Inject constructor(
    private val apiService: ApiService,
    private val tokenManager: TokenManager
) {
    
    // 使用suspend函数（现代方案）
    suspend fun getUser(id: String): User {
        return try {
            val response = apiService.getUser(id)
            if (response.isSuccessful) {
                response.body()?.data ?: throw Exception("Empty response")
            } else {
                throw Exception("Error: ${response.code()}")
            }
        } catch (e: Exception) {
            throw e
        }
    }
    
    // 使用RxJava（备选方案）
    fun getUserRx(id: String): Observable<User> {
        return apiService.getUser(id)
            .toObservable()
            .map { response ->
                if (response.isSuccessful) {
                    response.body()?.data ?: throw Exception("Empty")
                } else {
                    throw Exception("Error")
                }
            }
            .retryWhen { errors ->
                errors.flatMap { error ->
                    if (error is HttpException && error.code() == 401) {
                        // 401时重试
                        Observable.timer(1, TimeUnit.SECONDS)
                    } else {
                        Observable.error(error)
                    }
                }
            }
    }
    
    suspend fun login(email: String, password: String) {
        val request = LoginRequest(email, password)
        val response = apiService.login(request)
        
        if (response.isSuccessful) {
            val loginData = response.body()?.data
            if (loginData != null) {
                tokenManager.setTokens(
                    loginData.accessToken,
                    loginData.refreshToken,
                    loginData.expiresIn
                )
            }
        } else {
            throw Exception("Login failed: ${response.code()}")
        }
    }
}

// ============ 7️⃣ ViewModel 使用 ============
@HiltViewModel
class UserViewModel @Inject constructor(
    private val userRepository: UserRepository
) : ViewModel() {
    
    private val _userState = MutableStateFlow<UiState<User>>(UiState.Idle)
    val userState: StateFlow<UiState<User>> = _userState.asStateFlow()
    
    fun loadUser(id: String) {
        viewModelScope.launch {
            _userState.value = UiState.Loading
            try {
                val user = userRepository.getUser(id)
                _userState.value = UiState.Success(user)
            } catch (e: Exception) {
                _userState.value = UiState.Error(e.message ?: "Unknown error")
            }
        }
    }
}

// UiState 类
sealed class UiState<T> {
    object Idle : UiState<Nothing>()
    object Loading : UiState<Nothing>()
    data class Success<T>(val data: T) : UiState<T>()
    data class Error<T>(val message: String) : UiState<T>()
}
```

#### 3️⃣ **token刷新的双重检查锁**

```kotlin
// 关键代码解析
synchronized(this) {  // 第一道锁
    // 再检查一次，防止多个线程同时刷新
    if (shouldRefresh()) {  // 第二重检查
        refreshToken()
    }
}

// 为什么需要双重检查？
// 假设线程A和B同时发出401请求：
//
// 不使用双重检查：
// 线程A: 获得锁 → 刷新token1 → 释放锁
// 线程B: 等待... → 获得锁 → 刷新token2 → 释放锁 ❌ 重复刷新
//
// 使用双重检查：
// 线程A: 获得锁 → 刷新token1 → 释放锁
// 线程B: 等待... → 获得锁 → 检查（已刷新）→ 释放锁 ✅ 只刷新一次
```

#### 4️⃣ **两层缓存设计**

```kotlin
class AdvancedUserRepository @Inject constructor(
    private val apiService: ApiService
) {
    // 内存缓存（热数据）
    private val memoryCache = mutableMapOf<String, CacheEntry<User>>()
    
    // 磁盘缓存（冷数据）
    @Inject
    lateinit var diskCache: RoomDatabase
    
    data class CacheEntry<T>(
        val data: T,
        val timestamp: Long,
        val ttl: Long = 5 * 60 * 1000  // 5分钟过期
    ) {
        fun isExpired() = System.currentTimeMillis() - timestamp > ttl
    }
    
    suspend fun getUser(id: String): User {
        // 一层缓存：内存
        memoryCache[id]?.let { entry ->
            if (!entry.isExpired()) {
                return entry.data
            } else {
                memoryCache.remove(id)
            }
        }
        
        // 二层缓存：磁盘（数据库）
        diskCache.userDao().getUser(id)?.let { cachedUser ->
            return cachedUser
        }
        
        // 三层：网络
        val user = apiService.getUser(id)
        
        // 写入两层缓存
        memoryCache[id] = CacheEntry(user, System.currentTimeMillis())
        diskCache.userDao().insert(user)
        
        return user
    }
}
```

#### 5️⃣ **常见面试追问**

```
Q: "为什么Token要单独管理？"
A: 
   - 集中控制token的生命周期
   - 方便刷新和失效处理
   - 支持多个API调用共享token
   - 便于单元测试（Mock token）

Q: "401后为什么要同步刷新？"
A:
   - Interceptor是同步执行
   - 需要阻塞等待新token
   - 然后重试原请求
   
Q: "怎样处理token过期发生在业务层？"
A: 
   - 业务层不应该处理token过期
   - 这是网络层的职责
   - 业务层只需处理业务异常

Q: "RxJava的retryWhen怎么配合token刷新？"
A:
   ```kotlin
   apiCall()
       .retryWhen { errors ->
           errors.flatMap { error ->
               if (is401(error)) {
                   refreshToken()  // 同步刷新
                   Observable.just(Unit)  // 重试
               } else {
                   Observable.error(error)
               }
           }
       }
```

```

---

### Q8: OkHttp的拦截器链机制？连接池原理？

#### 1️⃣ **拦截器链工作流程**

```

请求流程（从上到下）： ┌────────────────────────────────────┐ │ Application Interceptors 应用层 │ │ (logging, request modification) │ └──────────┬───────────────────────┘ ↓ ┌────────────────────────────────────┐ │ Network Interceptors 网络层 │ │ (protocol, compression, auth) │ └──────────┬───────────────────────┘ ↓ ┌────────────────────────────────────┐ │ Real Request 真实HTTP请求 │ └──────────┬───────────────────────┘ ↓ 响应流程（从下到上）： ┌────────────────────────────────────┐ │ Network Interceptors 解析响应 │ │ (decompression, caching) │ └──────────┬───────────────────────┘ ↓ ┌────────────────────────────────────┐ │ Application Interceptors │ │ (response logging) │ └────────────────────────────────────┘

````

#### 2️⃣ **完整拦截器实现**

```kotlin
// 自定义应用层拦截器
class RequestLoggingInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request()
        
        val startTime = System.nanoTime()
        println("→ 发送请求：${request.method} ${request.url}")
        
        // 打印请求头
        request.headers.forEach { (name, value) ->
            println("  $name: $value")
        }
        
        // 执行请求（传递给下一个拦截器）
        val response = try {
            chain.proceed(request)
        } catch (e: Exception) {
            println("✗ 请求失败：${e.message}")
            throw e
        }
        
        val elapsedTime = (System.nanoTime() - startTime) / 1_000_000
        println("← 响应：${response.code} (${elapsedTime}ms)")
        
        return response
    }
}

// 自定义网络层拦截器（处理重试、超时等）
class RetryInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        var request = chain.request()
        var response: Response? = null
        var lastException: Exception? = null
        
        // 最多重试3次
        for (attempt in 0..2) {
            try {
                response = chain.proceed(request)
                
                // 5xx错误也重试
                if (response.code >= 500) {
                    response.close()
                    if (attempt < 2) {
                        Thread.sleep((attempt + 1) * 1000L)  // 指数退避
                        continue
                    }
                }
                
                return response
            } catch (e: IOException) {
                lastException = e
                if (attempt < 2) {
                    Thread.sleep((attempt + 1) * 1000L)
                }
            }
        }
        
        throw lastException ?: IOException("Max retries exceeded")
    }
}

// 数据压缩拦截器
class GzipInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request()
        
        // 检查是否需要gzip压缩
        if (request.body != null && request.header("Content-Encoding") == null) {
            val originalBody = request.body!!
            
            val compressed = Buffer()
            GzipSink(compressed).use { gzipSink ->
                originalBody.writeTo(gzipSink)
            }
            
            val newRequest = request.newBuilder()
                .header("Content-Encoding", "gzip")
                .post(object : RequestBody() {
                    override fun contentType() = originalBody.contentType()
                    override fun writeTo(sink: BufferedSink) {
                        sink.write(compressed.snapshot())
                    }
                })
                .build()
            
            return chain.proceed(newRequest)
        }
        
        return chain.proceed(request)
    }
}

// 响应缓存拦截器
class CacheInterceptor : Interceptor {
    private val cache = mutableMapOf<String, CacheResponse>()
    
    data class CacheResponse(
        val response: Response,
        val timestamp: Long
    )
    
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request()
        val cacheKey = request.url.toString()
        
        // 检查内存缓存
        cache[cacheKey]?.let { cached ->
            if (System.currentTimeMillis() - cached.timestamp < 60000) {  // 1分钟
                return cached.response
            } else {
                cache.remove(cacheKey)
            }
        }
        
        val response = chain.proceed(request)
        
        // 缓存可缓存的响应
        if (response.code == 200) {
            cache[cacheKey] = CacheResponse(response, System.currentTimeMillis())
        }
        
        return response
    }
}
````

#### 3️⃣ **连接池原理**

```
连接池架构：
┌──────────────────────────────┐
│ ConnectionPool               │
│ - 连接队列 (RealConnection) │
│ - 清理器线程                │
│ - 最大空闲连接数             │
│ - 连接最大保活时间           │
└──────────────────────────────┘
         ↓
    保持HTTP连接的复用
    
优势：
1. 减少握手次数（TCP三次握手）
2. 减少SSL/TLS握手
3. 提高请求速度
```

```kotlin
// OkHttp连接池使用

class NetworkModule {
    companion object {
        @Singleton
        @Provides
        fun provideOkHttpClient(): OkHttpClient {
            val connectionPool = ConnectionPool(
                maxIdleConnections = 5,      // 最多5条空闲连接
                keepAliveDuration = 5,       // 保活5分钟
                timeUnit = TimeUnit.MINUTES
            )
            
            return OkHttpClient.Builder()
                .connectionPool(connectionPool)
                .connectTimeout(30, TimeUnit.SECONDS)
                .readTimeout(30, TimeUnit.SECONDS)
                .writeTimeout(30, TimeUnit.SECONDS)
                .build()
        }
    }
}

// 连接复用示例
fun demonstrateConnectionReuse() {
    val client = OkHttpClient()
    
    // 第一个请求 → 创建新连接
    val request1 = Request.Builder().url("https://api.example.com/user/1").build()
    val response1 = client.newCall(request1).execute()
    
    // 第二个请求 → 复用前一个连接
    val request2 = Request.Builder().url("https://api.example.com/user/2").build()
    val response2 = client.newCall(request2).execute()
    
    // 两个请求使用同一个TCP连接（HTTP/1.1 Keep-Alive）
}

// 连接池内部工作原理（伪代码）
class ConnectionPool {
    private val connections = ConcurrentLinkedQueue<RealConnection>()
    private val cleanupExecutor = ScheduledThreadPoolExecutor(1)
    
    init {
        // 启动定时清理任务
        cleanupExecutor.scheduleAtFixedRate(
            { evictConnections() },
            5, 5, TimeUnit.MINUTES
        )
    }
    
    fun get(address: Address): RealConnection? {
        // 查找可复用的连接
        return connections.find { connection ->
            connection.isEligible(address) &&  // 匹配地址
            !connection.isExpired()             // 未过期
        }?.also {
            it.incrementUseCount()  // 增加使用计数
        }
    }
    
    fun put(connection: RealConnection) {
        connections.add(connection)
    }
    
    private fun evictConnections() {
        // 清理过期或空闲太久的连接
        connections.removeIf { connection ->
            connection.isExpired() ||
            connection.isIdle() && System.currentTimeMillis() - connection.lastUseTime > 5 * 60 * 1000
        }
    }
}

// HTTP/2 SPDY优化（自动多路复用）
class Http2Example {
    fun setupHttp2() {
        val client = OkHttpClient.Builder()
            .protocols(listOf(Protocol.HTTP_2, Protocol.HTTP_1_1))
            .build()
        
        // OkHttp会自动使用HTTP/2
        // 一个TCP连接上可以发送多个请求（不需要等待响应）
    }
}
```

---

## 第五部分：应用架构（★★★★☆）

### Q9: MVVM + Hilt架构的最佳实践？

#### 1️⃣ **架构分层**

```
┌─────────────────────────────────┐
│ UI Layer (Activity/Fragment)     │
│ - 生命周期管理                   │
│ - UI渲染                         │
└──────────┬──────────────────────┘
           ↓
┌─────────────────────────────────┐
│ ViewModel Layer                 │
│ - 状态管理                       │
│ - 业务逻辑协调                   │
└──────────┬──────────────────────┘
           ↓
┌─────────────────────────────────┐
│ Repository Layer (数据仓库)      │
│ - 数据聚合                       │
│ - 缓存策略                       │
│ - 事务处理                       │
└──────────┬──────────────────────┘
           ↓
┌─────────────────────────────────┐
│ Data Sources                    │
│ - Network (OkHttp/Retrofit)     │
│ - Database (Room)               │
│ - Local Storage                 │
└─────────────────────────────────┘
```

#### 2️⃣ **完整MVVM+Hilt实现**

```kotlin
// ============ 1️⃣ 数据模型 ============
data class User(
    val id: String,
    val name: String,
    val email: String,
    val avatar: String
)

sealed class UserListUiState {
    object Idle : UserListUiState()
    object Loading : UserListUiState()
    data class Success(val users: List<User>) : UserListUiState()
    data class Error(val message: String) : UserListUiState()
}

// ============ 2️⃣ 业务事件 ============
sealed class UserListEvent {
    object LoadUsers : UserListEvent()
    data class SelectUser(val userId: String) : UserListEvent()
    object Retry : UserListEvent()
}

// ============ 3️⃣ ViewModel（核心） ============
@HiltViewModel
class UserListViewModel @Inject constructor(
    private val userRepository: UserRepository,
    private val analytics: AnalyticsService,
    savedStateHandle: SavedStateHandle
) : ViewModel() {
    
    // 状态管理
    private val _uiState = MutableStateFlow<UserListUiState>(UserListUiState.Idle)
    val uiState: StateFlow<UserListUiState> = _uiState.asStateFlow()
    
    // 单向事件（页面导航等）
    private val _navigationEvent = MutableSharedFlow<NavigationEvent>()
    val navigationEvent = _navigationEvent.asSharedFlow()
    
    // 恢复保存的状态
    private val savedUserId: String? = savedStateHandle.get("userId")
    
    // 处理用户事件
    fun onEvent(event: UserListEvent) {
        when (event) {
            is UserListEvent.LoadUsers -> loadUsers()
            is UserListEvent.SelectUser -> selectUser(event.userId)
            is UserListEvent.Retry -> loadUsers()
        }
    }
    
    private fun loadUsers() {
        viewModelScope.launch {
            _uiState.value = UserListUiState.Loading
            
            try {
                val users = userRepository.getUsers()
                _uiState.value = UserListUiState.Success(users)
                analytics.logEvent("users_loaded", mapOf("count" to users.size))
            } catch (e: Exception) {
                _uiState.value = UserListUiState.Error(e.message ?: "Unknown error")
                analytics.logError("load_users_failed", e)
            }
        }
    }
    
    private fun selectUser(userId: String) {
        viewModelScope.launch {
            _navigationEvent.emit(
                NavigationEvent.UserDetail(userId)
            )
        }
    }
    
    init {
        // 自动加载
        loadUsers()
    }
}

// 导航事件
sealed class NavigationEvent {
    data class UserDetail(val userId: String) : NavigationEvent()
    object Back : NavigationEvent()
}

// ============ 4️⃣ Repository（数据聚合） ============
@Singleton
class UserRepository @Inject constructor(
    private val userApiService: UserApiService,
    private val userDao: UserDao,
    private val userCache: UserCache
) {
    
    // 单一数据源：getUsers返回最新数据
    suspend fun getUsers(): List<User> {
        return try {
            // 1. 尝试网络请求
            val users = userApiService.getUsers()
            
            // 2. 保存到数据库
            userDao.insertUsers(users)
            userCache.setUsers(users)
            
            users
        } catch (e: Exception) {
            // 3. 网络失败，从数据库读取
            userDao.getAllUsers().ifEmpty {
                throw e  // 都没有就抛出异常
            }
        }
    }
    
    suspend fun getUserById(id: String): User {
        // 先查缓存
        userCache.getUser(id)?.let { return it }
        
        return try {
            val user = userApiService.getUserById(id)
            userDao.insertUser(user)
            userCache.setUser(user)
            user
        } catch (e: Exception) {
            userDao.getUserById(id) ?: throw e
        }
    }
}

// ============ 5️⃣ 缓存层 ============
@Singleton
class UserCache @Inject constructor() {
    private val memoryCache = mutableMapOf<String, User>()
    private val cacheLock = Mutex()
    
    suspend fun getUser(id: String): User? {
        return cacheLock.withLock {
            memoryCache[id]
        }
    }
    
    suspend fun setUser(user: User) {
        cacheLock.withLock {
            memoryCache[user.id] = user
        }
    }
    
    suspend fun setUsers(users: List<User>) {
        cacheLock.withLock {
            users.forEach { memoryCache[it.id] = it }
        }
    }
}

// ============ 6️⃣ API服务 ============
interface UserApiService {
    @GET("/users")
    suspend fun getUsers(): List<User>
    
    @GET("/users/{id}")
    suspend fun getUserById(@Path("id") id: String): User
}

// ============ 7️⃣ Room数据库 ============
@Entity(tableName = "users")
data class UserEntity(
    @PrimaryKey val id: String,
    val name: String,
    val email: String,
    val avatar: String
)

@Dao
interface UserDao {
    @Query("SELECT * FROM users")
    suspend fun getAllUsers(): List<UserEntity>
    
    @Query("SELECT * FROM users WHERE id = :id")
    suspend fun getUserById(id: String): UserEntity?
    
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertUser(user: UserEntity)
    
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertUsers(users: List<UserEntity>)
}

@Database(entities = [UserEntity::class], version = 1)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao
}

// ============ 8️⃣ Hilt Module (依赖注入) ============
@Module
@InstallIn(SingletonComponent::class)
object DataModule {
    
    @Provides
    @Singleton
    fun provideDatabase(context: Context): AppDatabase {
        return Room.databaseBuilder(
            context,
            AppDatabase::class.java,
            "app_db"
        ).build()
    }
    
    @Provides
    @Singleton
    fun provideUserDao(database: AppDatabase): UserDao {
        return database.userDao()
    }
    
    @Provides
    @Singleton
    fun provideUserApiService(retrofit: Retrofit): UserApiService {
        return retrofit.create(UserApiService::class.java)
    }
}

@Module
@InstallIn(SingletonComponent::class)
object RepositoryModule {
    
    @Provides
    @Singleton
    fun provideUserRepository(
        apiService: UserApiService,
        dao: UserDao,
        cache: UserCache
    ): UserRepository {
        return UserRepository(apiService, dao, cache)
    }
}

// ============ 9️⃣ UI层 (Activity) ============
@AndroidEntryPoint
class UserListActivity : AppCompatActivity() {
    
    private val viewModel: UserListViewModel by viewModels()
    private lateinit var adapter: UserAdapter
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_user_list)
        
        setupUI()
        observeState()
        observeNavigation()
    }
    
    private fun setupUI() {
        adapter = UserAdapter { userId ->
            viewModel.onEvent(UserListEvent.SelectUser(userId))
        }
        
        binding.recyclerView.adapter = adapter
        
        binding.retryButton.setOnClickListener {
            viewModel.onEvent(UserListEvent.Retry)
        }
    }
    
    private fun observeState() {
        lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.uiState.collect { state ->
                    when (state) {
                        is UserListUiState.Idle -> showIdle()
                        is UserListUiState.Loading -> showLoading()
                        is UserListUiState.Success -> showUsers(state.users)
                        is UserListUiState.Error -> showError(state.message)
                    }
                }
            }
        }
    }
    
    private fun observeNavigation() {
        lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.navigationEvent.collect { event ->
                    when (event) {
                        is NavigationEvent.UserDetail -> {
                            startActivity(
                                Intent(this@UserListActivity, UserDetailActivity::class.java)
                                    .putExtra("userId", event.userId)
                            )
                        }
                        NavigationEvent.Back -> finish()
                    }
                }
            }
        }
    }
    
    private fun showUsers(users: List<User>) {
        binding.loadingView.gone()
        binding.errorView.gone()
        adapter.submitList(users)
    }
    
    private fun showLoading() {
        binding.loadingView.visible()
        binding.errorView.gone()
    }
    
    private fun showError(message: String) {
        binding.loadingView.gone()
        binding.errorView.visible()
        binding.errorText.text = message
    }
    
    private fun showIdle() {
        binding.loadingView.gone()
        binding.errorView.gone()
    }
}

// ============ 十 UI适配器 ============
class UserAdapter(
    private val onUserClick: (String) -> Unit
) : ListAdapter<User, UserViewHolder>(UserDiffCallback()) {
    
    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): UserViewHolder {
        return UserViewHolder(
            UserItemBinding.inflate(LayoutInflater.from(parent.context), parent, false),
            onUserClick
        )
    }
    
    override fun onBindViewHolder(holder: UserViewHolder, position: Int) {
        holder.bind(getItem(position))
    }
}

class UserViewHolder(
    private val binding: UserItemBinding,
    private val onUserClick: (String) -> Unit
) : RecyclerView.ViewHolder(binding.root) {
    
    fun bind(user: User) {
        binding.apply {
            nameText.text = user.name
            emailText.text = user.email
            
            root.setOnClickListener {
                onUserClick(user.id)
            }
        }
    }
}

class UserDiffCallback : DiffUtil.ItemCallback<User>() {
    override fun areItemsTheSame(oldItem: User, newItem: User) = oldItem.id == newItem.id
    override fun areContentsTheSame(oldItem: User, newItem: User) = oldItem == newItem
}
```

#### 3️⃣ **MVVM最佳实践**

```kotlin
// ✅ 最佳实践1：单向数据流(UDF)
/*
    用户交互 → ViewModel.onEvent() 
         ↓
    修改状态 → _state.value = newState
         ↓
    观察者收到更新 → UI重新渲染
*/

// ✅ 最佳实践2：SavedStateHandle处理进程销毁
@HiltViewModel
class DetailViewModel @Inject constructor(
    savedStateHandle: SavedStateHandle
) : ViewModel() {
    private val userId: String = savedStateHandle.get("userId")!!
    
    // 进程被杀死时，savedInstanceState会被保存
    // 恢复时，ViewModel会用相同的savedStateHandle重建
}

// ✅ 最佳实践3：异常处理
private fun loadData() {
    viewModelScope.launch {
        try {
            val data = repository.getData()
            _state.value = State.Success(data)
        } catch (e: CancellationException) {
            // 协程被取消，不处理（正常行为）
            throw e
        } catch (e: NetworkException) {
            _state.value = State.Error("网络错误")
        } catch (e: Exception) {
            _state.value = State.Error("未知错误")
        }
    }
}

// ✅ 最佳实践4：避免重复订阅
class BadActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // ❌ 多次触发collect（每次屏幕旋转都会重复）
        lifecycleScope.launch {
            viewModel.state.collect { updateUI(it) }
        }
    }
}

class GoodActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // ✅ 使用repeatOnLifecycle
        lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.state.collect { updateUI(it) }
            }
        }
    }
}
```

---

## 第六部分：Android Framework（★★★★☆）

### Q10: Handler/Looper/MessageQueue的工作原理？为什么主线程不会阻塞？

#### 1️⃣ **完整工作流程**

```
主线程启动流程：
┌─────────────────────────────────┐
│ ActivityThread.main()           │
│ - 入口点                         │
└──────────┬──────────────────────┘
           ↓
┌─────────────────────────────────┐
│ Looper.prepareMainLooper()      │
│ - 创建主线程Looper              │
│ - 创建MainThread.sThreadLocal   │
└──────────┬──────────────────────┘
           ↓
┌─────────────────────────────────┐
│ Looper.loop()                   │
│ - 进入无限循环                   │
│ - 等待消息                       │
└──────────┬──────────────────────┘
           ↓
┌─────────────────────────────────┐
│ MessageQueue                     │
│ - 维护消息队列                   │
│ - 支持优先级和延迟               │
└──────────┬──────────────────────┘
           ↓
┌─────────────────────────────────┐
│ Handler                         │
│ - 发送消息                       │
│ - 处理消息                       │
└─────────────────────────────────┘
```

#### 2️⃣ **完整代码实现**

```kotlin
// ============ Handler/Looper基础用法 ============
class HandlerExamples {
    
    // 1️⃣ 基础Handler
    fun basicHandlerUsage() {
        // 主线程中使用
        val handler = Handler(Looper.getMainLooper())
        
        handler.post {
            // 这个代码会在主线程执行
            updateUI()
        }
        
        handler.postDelayed({
            // 延迟2秒执行
            showToast()
        }, 2000)
        
        handler.postAtTime({
            // 在指定时间执行
        }, SystemClock.uptimeMillis() + 5000)
    }
    
    // 2️⃣ 子线程创建Handler
    fun handlerInThread() {
        thread {
            // 子线程中需要先prepare Looper
            Looper.prepare()
            
            val handler = Handler(Looper.myLooper()!!)
            handler.post { /* 在子线程执行 */ }
            
            // 进入消息循环
            Looper.loop()  // 这会阻塞线程
        }
    }
    
    // 3️⃣ 安全的Handler（避免内存泄漏）
    class SafeHandler(looper: Looper) : Handler(looper) {
        // 静态内部类 + WeakReference
        private var callback: WeakReference<Callback>? = null
        
        interface Callback {
            fun onMessage(msg: Message)
        }
        
        fun setCallback(callback: Callback) {
            this.callback = WeakReference(callback)
        }
        
        override fun handleMessage(msg: Message) {
            callback?.get()?.onMessage(msg)  // 安全调用
        }
    }
}

// ============ MessageQueue内部原理 ============
class MessageQueueExplanation {
    /*
    MessageQueue的数据结构：
    
    单链表：
    Message1 (time=100) → Message2 (time=200) → Message3 (time=300) → null
    
    特点：
    1. 按时间排序（优先级队列）
    2. 支持延迟消息
    3. 使用native epoll监听文件描述符
    */
    
    // 简化实现
    class SimpleMessageQueue {
        private var mMessages: Message? = null
        
        fun enqueueMessage(msg: Message, uptimeMillis: Long): Boolean {
            msg.when = uptimeMillis
            
            // 按时间插入（有序）
            if (mMessages == null || uptimeMillis == 0L || uptimeMillis < mMessages!!.when) {
                // 插入队列头
                msg.next = mMessages
                mMessages = msg
            } else {
                // 按顺序插入
                var prev = mMessages
                var cur = prev?.next
                while (cur != null && cur.when <= uptimeMillis) {
                    prev = cur
                    cur = cur.next
                }
                msg.next = cur
                prev?.next = msg
            }
            
            return true
        }
        
        fun next(): Message? {
            while (true) {
                val nextPollTimeoutMillis = computeDelayMillis()  // 计算等待时间
                
                if (nextPollTimeoutMillis > 0) {
                    // 使用epoll等待
                    nativePollOnce(nextPollTimeoutMillis)
                }
                
                val msg = mMessages ?: continue
                
                if (msg.when <= SystemClock.uptimeMillis()) {
                    // 消息时间已到，返回
                    mMessages = msg.next
                    return msg
                } else {
                    // 消息未到，继续等待
                    continue
                }
            }
        }
        
        private fun computeDelayMillis(): Long {
            val now = SystemClock.uptimeMillis()
            val nextMsg = mMessages ?: return -1
            val msgTime = nextMsg.when
            return if (msgTime <= now) 0 else msgTime - now
        }
        
        // Native方法（实际由C++实现）
        private external fun nativePollOnce(timeoutMillis: Long)
    }
}

// ============ Looper工作流程 ============
class LooperExplanation {
    /*
    Looper.loop()的核心逻辑：
    
    public static void loop() {
        final Looper me = myLooper();
        final MessageQueue queue = me.mQueue;
        
        while (true) {
            Message msg = queue.next();  // 阻塞等待消息
            
            if (msg == null) {
                // 队列退出
                return;
            }
            
            msg.target.dispatchMessage(msg);  // 分发消息
        }
    }
    
    关键点：
    1. 无限循环
    2. queue.next()会阻塞等待
    3. 收到消息后dispatch
    4. 继续循环
    */
    
    // 演示无限循环
    fun demonstrateLoop() {
        while (true) {
            val msg = MessageQueue.next()  // 这里会阻塞
            if (msg == null) break  // 退出信号
            
            println("处理消息: $msg")
            // 继续循环，处理下一条消息
        }
    }
}

// ============ 为什么主线程不会阻塞 ============
class WhyMainThreadDoesntBlock {
    /*
    误区：
    "Looper.loop()是无限循环，为什么不卡死？"
    
    真相：
    1. loop()确实是无限循环
    2. 但queue.next()会让线程进入休眠（epoll）
    3. 当没有消息时，线程不消耗CPU
    4. 当有消息到达时，线程被唤醒
    
    类比：
    主线程像一个服务员：
    - 无消息时：坐下休息（epoll休眠）
    - 有消息来：起身处理
    - 处理完：继续休息
    
    CPU不会一直被占用
    */
    
    // 演示epoll的作用
    class EpollDemo {
        external fun epollWait(timeoutMillis: Int): Int
        
        fun demonstrateMainLoop() {
            while (true) {
                // 1. 计算需要等待多长时间
                val timeToWait = getNextMessageDelayMillis()
                
                // 2. 线程进入休眠，等待事件或超时
                epollWait(timeToWait)  // 这里线程不消耗CPU
                
                // 3. 被唤醒后继续处理消息
                val msg = getNextMessage()
                if (msg != null) {
                    processMessage(msg)
                }
            }
        }
        
        private fun getNextMessageDelayMillis(): Int {
            // 计算下一条消息的延迟时间
            val nextMsg = peekMessage()
            return if (nextMsg != null) {
                Math.max(0, (nextMsg.when - SystemClock.uptimeMillis()).toInt())
            } else {
                -1  // 无限等待
            }
        }
        
        private fun peekMessage(): Message? {
            // 返回队列中最近的消息
            return null
        }
        
        private fun getNextMessage(): Message? {
            return null
        }
        
        private fun processMessage(msg: Message) {
            // 处理消息
        }
    }
}

// ============ Handler发送消息的机制 ============
class HandlerMessageSending {
    
    class DetailedHandler(looper: Looper) : Handler(looper) {
        
        // post → sendMessage的转换
        fun demonstratePost() {
            // 这个：
            post {
                doWork()
            }
            
            // 等同于：
            val msg = Message.obtain()
            msg.callback = Runnable { doWork() }
            sendMessage(msg)
        }
        
        // sendMessage发送过程
        fun demonstrateSendMessage() {
            val msg = Message.obtain()
            msg.what = 1
            msg.arg1 = 100
            
            // 1. 设置目标Handler
            msg.target = this
            
            // 2. 计算执行时间
            val delayMillis = 0L
            val uptimeMillis = SystemClock.uptimeMillis() + delayMillis
            msg.when = uptimeMillis
            
            // 3. 入队到MessageQueue
            val queue = looper.queue
            queue.enqueueMessage(msg, uptimeMillis)
            
            // 4. 唤醒等待的线程（如果在sleep）
            queue.nativeWake()
        }
        
        override fun handleMessage(msg: Message) {
            when (msg.what) {
                1 -> println("收到消息，arg1=${msg.arg1}")
            }
        }
    }
}

// ============ 常见使用模式 ============
class CommonPatterns {
    
    // 模式1：线程间通信
    fun threadCommunication() {
        val handler = Handler(Looper.getMainLooper())
        
        thread {
            // 子线程做耗时操作
            val result = heavyComputation()
            
            // 切换回主线程
            handler.post {
                updateUIOnMainThread(result)
            }
        }
    }
    
    // 模式2：延迟任务
    fun delayedTask() {
        val handler = Handler(Looper.getMainLooper())
        
        handler.postDelayed({
            // 2秒后执行
            showToast("延迟弹窗")
        }, 2000)
    }
    
    // 模式3：轮询任务
    fun periodicTask() {
        val handler = Handler(Looper.getMainLooper())
        
        fun startPolling() {
            handler.postDelayed({
                doCheckWork()
                startPolling()  // 递归继续
            }, 5000)
        }
        
        startPolling()
    }
    
    // 模式4：移除待处理任务
    fun removeCallbacks() {
        val handler = Handler(Looper.getMainLooper())
        
        val runnable = Runnable { doWork() }
        
        handler.postDelayed(runnable, 1000)
        
        // 如果想取消这个任务
        handler.removeCallbacks(runnable)
    }
    
    private fun heavyComputation(): String = ""
    private fun updateUIOnMainThread(result: String) {}
    private fun showToast(msg: String) {}
    private fun doCheckWork() {}
    private fun doWork() {}
}
```

#### 3️⃣ **常见面试追问**

```
Q: "为什么要调用Looper.prepare()？"
A: Looper通过ThreadLocal存储
   - prepare()方法会创建新Looper并放入ThreadLocal
   - 不调用就没有Looper
   - 主线程已经自动prepare过，不需要再调

Q: "loop()无限循环会泄漏吗？"
A: 不会
   - loop()是线程级的，存活时间 = 线程存活时间
   - 线程销毁时，Looper也销毁
   - 只有Message引用泄漏才是问题

Q: "为什么不用wait/notify而用epoll？"
A: 性能考虑
   - epoll是Linux系统调用，更高效
   - wait/notify是Java级别
   - epoll可以等待多种事件（不仅是消息队列）

Q: "MessageQueue的next()为什么返回Message而不是null？"
A: 返回null代表退出信号
   - 调用Looper.quit()会发送null
   - 其他情况下always返回Message
```

---

## 面试高频深度问题补充

### Q11: 线程池(ThreadPoolExecutor)的核心参数配置和执行流程

```kotlin
// 核心参数
class ThreadPoolParams {
    // corePoolSize: 核心线程数（始终活跃）
    // maximumPoolSize: 最大线程数
    // keepAliveTime: 超过core的线程空闲多久被回收
    // blockingQueue: 任务队列
    // rejectedExecutionHandler: 拒绝策略
}

// ✅ CPU密集型配置
val cpuIntensivePool = ThreadPoolExecutor(
    corePoolSize = Runtime.getRuntime().availableProcessors(),
    maximumPoolSize = Runtime.getRuntime().availableProcessors() + 1,
    keepAliveTime = 0,
    timeUnit = TimeUnit.SECONDS,
    workQueue = SynchronousQueue()  // 无队列，快速处理
)

// ✅ IO密集型配置
val ioIntensivePool = ThreadPoolExecutor(
    corePoolSize = Runtime.getRuntime().availableProcessors() * 2,
    maximumPoolSize = Runtime.getRuntime().availableProcessors() * 4,
    keepAliveTime = 60,
    timeUnit = TimeUnit.SECONDS,
    workQueue = LinkedBlockingQueue(1000)  // 较大队列
)

// 执行流程
val executor = ThreadPoolExecutor(2, 4, 60, TimeUnit.SECONDS, LinkedBlockingQueue(2))

// 1. 前2个任务 → 直接创建核心线程执行
executor.execute { task1() }  // thread1创建
executor.execute { task2() }  // thread2创建

// 2. 第3、4个任务 → 入队等待
executor.execute { task3() }  // 入队
executor.execute { task4() }  // 入队

// 3. 第5、6个任务 → 队列满，创建非核心线程
executor.execute { task5() }  // thread3创建
executor.execute { task6() }  // thread4创建

// 4. 第7个任务 → 队列满，线程满 → 拒绝
executor.execute { task7() }  // 触发拒绝策略

// 拒绝策略
executor.setRejectedExecutionHandler { runnable, executor ->
    // AbortPolicy（默认）：抛异常
    // CallerRunsPolicy：主线程执行
    // DiscardPolicy：丢弃
    // DiscardOldestPolicy：丢弃最老的任务
}
```

---

## 总结：面试回答框架

### 任何技术问题都用这个框架：

```
问题 → 原理 → 为什么 → 代码 → 场景 → 常见错误 → 追问预案
  
例如：
Q: "synchronized和volatile区别？"

原理：
- synchronized: Monitor锁机制
- volatile: 内存屏障指令

为什么：
- synchronized提供互斥和可见性
- volatile只提供可见性（不保证原子性）

代码：展示两者无法替代关系

场景：什么时候用哪个

常见错误：volatile count++为什么错

追问：prepare for"为什么volatile写入立即同步？"
```

---

这份指南可以直接用来： ✅ 面试前深度复习 ✅ 遇到追问时随时翻看 ✅ 模拟面试的答题参考 ✅ 代码写在本地运行验证