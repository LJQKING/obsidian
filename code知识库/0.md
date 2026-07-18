## 1. Kotlin 开发语言与特性

### 1.1 干什么的

Kotlin 是 JetBrains 推出的在 JVM 上运行的静态类型语言，已成为 Android 官方推荐开发语言。相比 Java，Kotlin 具有更简洁的语法、更强的类型安全性和更好的函数式编程支持。

### 1.2 怎么用的

#### 扩展函数 (Extension Functions)

```kotlin
// 为 String 类添加扩展函数
fun String.isEmailValid(): Boolean {
    return this.matches(Regex("^[A-Za-z0-9+_.-]+@(.+)$"))
}

val email = "test@example.com"
if (email.isEmailValid()) {
    println("Email is valid")
}

// 带接收者的扩展函数 (Scope Function)
val result = StringBuilder().apply {
    append("Hello")
    append(" ")
    append("World")
}.toString()
```

#### Lambda 与高阶函数

```kotlin
// 高阶函数：接收函数作为参数
inline fun <T> List<T>.myFilter(predicate: (T) -> Boolean): List<T> {
    val result = mutableListOf<T>()
    for (item in this) {
        if (predicate(item)) {
            result.add(item)
        }
    }
    return result
}

// 使用 Lambda
val numbers = listOf(1, 2, 3, 4, 5)
val evenNumbers = numbers.myFilter { it % 2 == 0 }

// 高阶函数作为返回值
fun getComparator(): (String, String) -> Int {
    return { a, b -> a.compareTo(b) }
}
```

#### 密封类 (Sealed Class)

```kotlin
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error(val exception: Exception) : Result<Nothing>()
    object Loading : Result<Nothing>()
}

// 使用密封类进行类型安全的分支判断
fun <T> handleResult(result: Result<T>) {
    when (result) {
        is Result.Success -> println("Success: ${result.data}")
        is Result.Error -> println("Error: ${result.exception.message}")
        is Result.Loading -> println("Loading...")
    }
}
```

#### 泛型 (Generics)

```kotlin
// 协变 (out) 和逆变 (in)
interface Repository<out T> {
    fun getData(): T
}

interface Consumer<in T> {
    fun accept(data: T)
}

// 泛型函数与约束
fun <T : Comparable<T>> findMax(list: List<T>): T? {
    return list.maxOrNull()
}

// 使用投影限制泛型
fun copyList(from: List<out Any>, to: MutableList<in Any>) {
    for (item in from) {
        to.add(item)
    }
}
```

#### 委托 (Delegation)

```kotlin
// 类委托
interface DataSource {
    fun fetchData(): String
}

class RealDataSource : DataSource {
    override fun fetchData() = "Real Data"
}

class CachedDataSource(dataSource: DataSource) : DataSource by dataSource {
    // 可选：覆盖特定方法
    override fun fetchData(): String {
        return "Cached: ${super.fetchData()}"
    }
}

// 属性委托
class UserViewModel {
    var name: String by Delegates.observable("") { _, old, new ->
        println("Name changed from $old to $new")
    }
}

// Lazy 属性委托
val expensiveComputation: String by lazy {
    println("Computing...")
    "Result"
}
```

### 1.3 源码原理

#### 扩展函数的编译原理

```kotlin
// Kotlin 源码
fun String.isEmailValid(): Boolean {
    return this.matches(Regex("^[A-Za-z0-9+_.-]+@(.+)$"))
}

// 编译后的 Java 字节码等价形式
public static final boolean isEmailValid(String $this) {
    return $this.matches("^[A-Za-z0-9+_.-]+@(.+)$");
}
```

**原理**：

- 扩展函数在编译时被转换为静态方法
- 接收者对象作为第一个参数传入
- 完全没有运行时开销，仅是语法糖
- `inline` 关键字会将函数体内联到调用点，进一步消除方法调用开销

#### Lambda 与内联的优化

```kotlin
// 使用 inline
inline fun <T> List<T>.myForEach(action: (T) -> Unit) {
    for (item in this) {
        action(item)  // 非虚拟调用，直接内联
    }
}

// 编译前
list.myForEach { println(it) }

// 编译后（伪代码）
for (item in list) {
    println(item)  // Lambda 被内联展开
}
```

**关键点**：

- 不使用 `inline` 时，Lambda 被编译为 `Function` 对象（产生堆栈分配）
- `inline` 会将 Lambda 代码直接展开到调用处，避免对象创建
- `noinline` 参数不会被内联，可以存储和传递

#### 密封类的内存优化

```kotlin
// 密封类成员的实现原理
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error(val exception: Exception) : Result<Nothing>()
    object Loading : Result<Nothing>()
}

// 单例对象的优化：Loading 在内存中只有一个实例
// 由 Kotlin Runtime 在类加载时创建，并由 JVM 的类加载机制保证线程安全
```

**编译优化**：

- 数据类自动生成 `equals()`、`hashCode()`、`toString()`、`copy()` 方法
- 对象单例在 JVM 级别由 `<clinit>` 方法初始化，保证线程安全

#### 委托的动态分发原理

```kotlin
// 类委托的字节码原理
class CachedDataSource(private val dataSource: DataSource) : DataSource by dataSource

// 编译后等价于：
class CachedDataSource(private val dataSource: DataSource) : DataSource {
    override fun fetchData(): String = dataSource.fetchData()
    // 所有其他 DataSource 方法都自动委托
}
```

**属性委托原理**（Lazy 为例）：

```kotlin
// Kotlin 源码
val expensiveComputation: String by lazy { "Result" }

// 编译后的伪代码
private val expensiveComputation$delegate = LazyThreadSafetyMode.PUBLICATION.lazy { "Result" }
val expensiveComputation: String
    get() = expensiveComputation$delegate.value

// LazyThreadSafetyMode.PUBLICATION 使用 volatile + compareAndSet 实现线程安全的单次初始化
```

---

## 2. MVVM 架构设计

### 2.1 干什么的

MVVM (Model-View-ViewModel) 是现代 Android 开发的标准架构，分离了 UI 表现层和业务逻辑层，提高了代码的可测试性和可维护性。

**三层结构**：

- **Model**：数据层（Repository、DataSource、Database）
- **View**：UI 层（Activity、Fragment、自定义 View）
- **ViewModel**：逻辑层（业务逻辑、状态管理、生命周期感知）

### 2.2 怎么用的

#### 基础 MVVM 框架

```kotlin
// 1. Model 层 - Repository 模式
interface UserRepository {
    suspend fun getUser(userId: String): Result<User>
    suspend fun updateUser(user: User): Result<Unit>
}

class UserRepositoryImpl(
    private val remoteDataSource: UserRemoteDataSource,
    private val localDataSource: UserLocalDataSource
) : UserRepository {
    override suspend fun getUser(userId: String): Result<User> {
        return try {
            val user = remoteDataSource.fetchUser(userId)
            localDataSource.saveUser(user)
            Result.success(user)
        } catch (e: Exception) {
            val localUser = localDataSource.getUser(userId)
            if (localUser != null) {
                Result.success(localUser)
            } else {
                Result.failure(e)
            }
        }
    }
}

// 2. ViewModel 层 - 状态管理
sealed class UserUiState {
    object Loading : UserUiState()
    data class Success(val user: User) : UserUiState()
    data class Error(val message: String) : UserUiState()
}

class UserViewModel(
    private val repository: UserRepository
) : ViewModel() {
    
    private val _uiState = MutableStateFlow<UserUiState>(UserUiState.Loading)
    val uiState: StateFlow<UserUiState> = _uiState.asStateFlow()
    
    fun loadUser(userId: String) {
        viewModelScope.launch {
            _uiState.value = UserUiState.Loading
            try {
                val result = repository.getUser(userId)
                result.onSuccess { user ->
                    _uiState.value = UserUiState.Success(user)
                }.onFailure { exception ->
                    _uiState.value = UserUiState.Error(exception.message ?: "Unknown error")
                }
            } catch (e: Exception) {
                _uiState.value = UserUiState.Error(e.message ?: "Unknown error")
            }
        }
    }
}

// 3. View 层 - UI 绑定
class UserFragment : Fragment() {
    private val viewModel: UserViewModel by viewModels()
    
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.uiState.collect { state ->
                    when (state) {
                        is UserUiState.Loading -> showLoading()
                        is UserUiState.Success -> showUser(state.user)
                        is UserUiState.Error -> showError(state.message)
                    }
                }
            }
        }
        
        viewModel.loadUser("user123")
    }
}
```

#### Repository 模式的完整实现

```kotlin
// Repository 层级设计
interface UserLocalDataSource {
    suspend fun getUser(id: String): User?
    suspend fun saveUser(user: User)
    suspend fun deleteUser(id: String)
}

interface UserRemoteDataSource {
    suspend fun fetchUser(id: String): User
    suspend fun updateUser(user: User): User
}

class UserLocalDataSourceImpl(private val userDao: UserDao) : UserLocalDataSource {
    override suspend fun getUser(id: String): User? {
        return userDao.getUserById(id)
    }
    
    override suspend fun saveUser(user: User) {
        userDao.insert(user)
    }
    
    override suspend fun deleteUser(id: String) {
        userDao.deleteById(id)
    }
}

class UserRemoteDataSourceImpl(private val apiService: ApiService) : UserRemoteDataSource {
    override suspend fun fetchUser(id: String): User {
        return apiService.getUser(id)
    }
    
    override suspend fun updateUser(user: User): User {
        return apiService.updateUser(user)
    }
}

// 缓存策略 Repository
class CachedUserRepository(
    private val remoteDataSource: UserRemoteDataSource,
    private val localDataSource: UserLocalDataSource
) : UserRepository {
    
    override suspend fun getUser(userId: String): Result<User> {
        return withContext(Dispatchers.IO) {
            // 优先读本地缓存
            localDataSource.getUser(userId)?.let {
                return@withContext Result.success(it)
            }
            
            // 本地没有则从远程获取
            try {
                val remoteUser = remoteDataSource.fetchUser(userId)
                localDataSource.saveUser(remoteUser)
                Result.success(remoteUser)
            } catch (e: Exception) {
                Result.failure(e)
            }
        }
    }
}
```

#### 模块化开发与代码解耦

```kotlin
// feature 模块的 public API 定义
// 文件：user/public/UserFeatureApi.kt
interface UserFeatureApi {
    fun getUserDetailRoute(): String
    fun createUserDetailIntent(userId: String): Intent
}

// feature 模块的实现
// 文件：user/src/main/java/UserFeatureImpl.kt
class UserFeatureImpl : UserFeatureApi {
    override fun getUserDetailRoute() = "user/detail/{userId}"
    
    override fun createUserDetailIntent(userId: String): Intent {
        return Intent().apply {
            putExtra("userId", userId)
        }
    }
}

// app 模块使用 feature API（通过 ServiceLoader 或 DI 注入）
class AppModuleSetup {
    fun setupFeatures() {
        val userFeature = ServiceLoader.load(UserFeatureApi::class.java).firstOrNull()
        userFeature?.let {
            // 注册路由、注册 ViewModel 等
        }
    }
}
```

### 2.3 源码原理

#### ViewModel 生命周期管理原理

```kotlin
// ViewModel 源码概览（com.android.arch.lifecycle.ViewModel）
public abstract class ViewModel {
    // 存储清理回调
    private final Map<String, Object> mBagOfTags = new ConcurrentHashMap<>();
    
    // 当关联的 Activity/Fragment 销毁时调用
    protected void onCleared() {
    }
}

// ViewModelStore 原理
public class ViewModelStore {
    private final HashMap<String, ViewModel> mMap = new HashMap<>();
    
    final void put(String key, ViewModel viewModel) {
        ViewModel oldViewModel = mMap.put(key, viewModel);
        if (oldViewModel != null) {
            oldViewModel.onCleared();
        }
    }
    
    final ViewModel get(String key) {
        return mMap.get(key);
    }
    
    public final void clear() {
        for (ViewModel vm : mMap.values()) {
            vm.onCleared();  // 确保 ViewModel 清理时调用回调
        }
        mMap.clear();
    }
}

// Activity 持有 ViewModelStore
public class ComponentActivity extends androidx.core.app.ComponentActivity {
    private ViewModelStore mViewModelStore;
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        // ...
        // ViewModelStore 的生命周期与 Activity 的 lastNonConfigurationInstance 关联
        // 配置变更时不会销毁，从而保证 ViewModel 实例的存活
    }
}
```

**关键机制**：

- ViewModel 在 Activity 配置变更时不会销毁（因为 ViewModelStore 通过 `onRetainNonConfigurationInstance()` 保留）
- Fragment 销毁时，会清空其 ViewModelStore，触发 `onCleared()`
- Activity 销毁时，最终会销毁 ViewModelStore

#### StateFlow 的实现原理

```kotlin
// StateFlow 核心实现（kotlinx.coroutines.flow）
public interface StateFlow<out T> : Flow<T> {
    public val value: T
}

// MutableStateFlow 实现
public class MutableStateFlow<T>(initialValue: T) : StateFlow<T> {
    private val state: AtomicReference<Any> = AtomicReference(initialValue)
    
    override var value: T
        get() = state.get() as T
        set(value) {
            // 比较更新，只有值改变时才通知订阅者
            if (state.compareAndSet(value, value).not()) {
                state.set(value)
                // 通知所有收集器
            }
        }
    
    override suspend fun collect(collector: FlowCollector<T>) {
        // 首先发送当前值
        collector.emit(value)
        // 然后订阅后续变化
        subscribeToUpdates(collector)
    }
}
```

**StateFlow 与 LiveData 的区别**：

- `StateFlow` 是 Flow 的实现，支持背压（backpressure）
- `StateFlow` 初始状态下立即发送最后一个值
- `StateFlow` 在 Dispatchers.Main 上发送收集（协程感知）
- `LiveData` 是传统的可观察对象，需要在主线程订阅

---

## 3. Jetpack 组件详解

### 3.1 干什么的

Jetpack 是 Google 推出的一套 Android 开发库，包含许多满足 Android 平台最佳实践的库。核心组件包括：ViewModel（状态管理）、LiveData（可观察数据）、Lifecycle（生命周期感知）、Navigation（导航）、Room（本地数据库）、DataStore（键值存储）等。

### 3.2 怎么用的

#### ViewModel - 状态管理

```kotlin
// 带参数的 ViewModel Factory
class UserViewModel(
    private val userId: String,
    private val repository: UserRepository
) : ViewModel() {
    
    private val _userData = MutableStateFlow<User?>(null)
    val userData: StateFlow<User?> = _userData.asStateFlow()
    
    init {
        loadUserData()
    }
    
    private fun loadUserData() {
        viewModelScope.launch {
            try {
                val user = repository.getUser(userId)
                _userData.value = user
            } catch (e: Exception) {
                // 错误处理
            }
        }
    }
}

// ViewModel Factory
class UserViewModelFactory(
    private val userId: String,
    private val repository: UserRepository
) : ViewModelProvider.Factory {
    
    override fun <T : ViewModel> create(modelClass: Class<T>): T {
        if (modelClass.isAssignableFrom(UserViewModel::class.java)) {
            return UserViewModel(userId, repository) as T
        }
        throw IllegalArgumentException("Unknown ViewModel class")
    }
}

// 使用 Factory 创建 ViewModel
val viewModel: UserViewModel by viewModels {
    UserViewModelFactory("user123", repository)
}
```

#### LiveData - 可观察数据（传统方式）

```kotlin
class UserViewModel : ViewModel() {
    
    private val _userLiveData = MutableLiveData<User>()
    val userLiveData: LiveData<User> = _userLiveData
    
    private val _loadingLiveData = MutableLiveData<Boolean>()
    val loadingLiveData: LiveData<Boolean> = _loadingLiveData
    
    // 转换 LiveData
    val userNameLiveData: LiveData<String> = userLiveData.map { user ->
        user.name
    }
    
    // 合并多个 LiveData
    val combinedLiveData: LiveData<Pair<User, Boolean>> = MediatorLiveData<Pair<User, Boolean>>().apply {
        addSource(userLiveData) { user ->
            val loading = loadingLiveData.value ?: false
            value = Pair(user, loading)
        }
        addSource(loadingLiveData) { loading ->
            val user = userLiveData.value
            if (user != null) {
                value = Pair(user, loading)
            }
        }
    }
    
    fun loadUser(userId: String) {
        _loadingLiveData.value = true
        viewModelScope.launch {
            try {
                val user = fetchUser(userId)
                _userLiveData.value = user
            } finally {
                _loadingLiveData.value = false
            }
        }
    }
}

// View 观察 LiveData
class UserFragment : Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        viewModel.userLiveData.observe(viewLifecycleOwner) { user ->
            textView.text = user.name
        }
        
        viewModel.loadingLiveData.observe(viewLifecycleOwner) { isLoading ->
            progressBar.isVisible = isLoading
        }
    }
}
```

#### Lifecycle - 生命周期感知

```kotlin
// 自定义生命周期感知组件
class LocationTracker(private val context: Context) : DefaultLifecycleObserver {
    
    override fun onResume(owner: LifecycleOwner) {
        startLocationUpdates()
    }
    
    override fun onPause(owner: LifecycleOwner) {
        stopLocationUpdates()
    }
    
    private fun startLocationUpdates() {
        // 请求位置更新
    }
    
    private fun stopLocationUpdates() {
        // 停止位置更新
    }
}

// Activity 注册生命周期观察者
class MapActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        val tracker = LocationTracker(this)
        lifecycle.addObserver(tracker)  // Activity 销毁时会自动移除
    }
}

// 编写生命周期感知的 Composable
class ProcessLifecycleFragment : Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        // 响应 STARTED 生命周期状态
        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                // 这段代码只在 STARTED 状态运行
                viewModel.uiState.collect { state ->
                    updateUI(state)
                }
            }
        }
    }
}
```

#### Navigation - 导航框架

```kotlin
// 定义导航图（nav_graph.xml）
<?xml version="1.0" encoding="utf-8"?>
<navigation xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/nav_graph">
    
    <fragment
        android:id="@+id/userListFragment"
        android:name="com.example.UserListFragment"
        android:label="User List">
        <action
            android:id="@+id/action_userList_to_userDetail"
            app:destination="@id/userDetailFragment" />
    </fragment>
    
    <fragment
        android:id="@+id/userDetailFragment"
        android:name="com.example.UserDetailFragment"
        android:label="User Detail">
        <argument
            android:name="userId"
            app:argType="string" />
    </fragment>
</navigation>

// Fragment 导航
class UserListFragment : Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        userAdapter.setOnUserClickListener { userId ->
            val action = UserListFragmentDirections.actionUserListToUserDetail(userId)
            findNavController().navigate(action)
        }
    }
}

// 接收导航参数
class UserDetailFragment : Fragment() {
    private val args: UserDetailFragmentArgs by navArgs()
    
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        val userId = args.userId
        viewModel.loadUser(userId)
    }
}
```

#### Room - 本地数据库

```kotlin
// 定义实体
@Entity(tableName = "users")
data class UserEntity(
    @PrimaryKey
    val id: String,
    val name: String,
    val email: String,
    val createdAt: Long = System.currentTimeMillis()
)

// 定义 DAO
@Dao
interface UserDao {
    
    @Query("SELECT * FROM users WHERE id = :userId LIMIT 1")
    suspend fun getUserById(userId: String): UserEntity?
    
    @Query("SELECT * FROM users ORDER BY createdAt DESC")
    fun getAllUsersFlow(): Flow<List<UserEntity>>
    
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertUser(user: UserEntity)
    
    @Update
    suspend fun updateUser(user: UserEntity)
    
    @Delete
    suspend fun deleteUser(user: UserEntity)
    
    @Query("DELETE FROM users WHERE id = :userId")
    suspend fun deleteUserById(userId: String)
}

// 定义数据库
@Database(
    entities = [UserEntity::class],
    version = 1,
    exportSchema = false
)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao
    
    companion object {
        @Volatile
        private var INSTANCE: AppDatabase? = null
        
        fun getInstance(context: Context): AppDatabase =
            INSTANCE ?: synchronized(this) {
                Room.databaseBuilder(
                    context.applicationContext,
                    AppDatabase::class.java,
                    "app_database"
                )
                .addMigrations(MIGRATION_1_2)  // 添加迁移
                .build()
                .also { INSTANCE = it }
            }
        
        // 数据库迁移
        private val MIGRATION_1_2 = object : Migration(1, 2) {
            override fun migrate(database: SupportSQLiteDatabase) {
                database.execSQL(
                    "ALTER TABLE users ADD COLUMN lastModified INTEGER DEFAULT 0"
                )
            }
        }
    }
}

// 在 Repository 中使用
class UserRepositoryImpl(private val userDao: UserDao) : UserRepository {
    
    override suspend fun getUser(userId: String): Result<User> {
        return try {
            val userEntity = userDao.getUserById(userId)
            if (userEntity != null) {
                Result.success(userEntity.toDomainModel())
            } else {
                Result.failure(Exception("User not found"))
            }
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    override fun observeUsers(): Flow<List<User>> {
        return userDao.getAllUsersFlow()
            .map { entities -> entities.map { it.toDomainModel() } }
    }
}
```

#### DataStore - 键值存储（替代 SharedPreferences）

```kotlin
// 定义 Proto DataStore schema
syntax = "proto3";

message UserPreferences {
    string user_id = 1;
    string theme = 2;
    bool notifications_enabled = 3;
}

// Kotlin 中使用 DataStore
class UserPreferencesRepository(private val context: Context) {
    
    companion object {
        private val Context.userPreferencesDataStore: DataStore<UserPreferences> by preferencesDataStore(
            name = "user_preferences",
            serializer = UserPreferencesSerializer
        )
    }
    
    val userPreferences: Flow<UserPreferences> = context.userPreferencesDataStore.data
    
    suspend fun saveUserTheme(theme: String) {
        context.userPreferencesDataStore.updateData { preferences ->
            preferences.toBuilder()
                .setTheme(theme)
                .build()
        }
    }
    
    suspend fun saveNotificationsEnabled(enabled: Boolean) {
        context.userPreferencesDataStore.updateData { preferences ->
            preferences.toBuilder()
                .setNotificationsEnabled(enabled)
                .build()
        }
    }
}

// DataStore Serializer
object UserPreferencesSerializer : Serializer<UserPreferences> {
    override val defaultValue: UserPreferences = UserPreferences.getDefaultInstance()
    
    override suspend fun readFrom(input: InputStream): UserPreferences {
        return try {
            UserPreferences.parseFrom(input)
        } catch (exception: IOException) {
            defaultValue
        }
    }
    
    override suspend fun writeTo(t: UserPreferences, output: OutputStream) {
        t.writeTo(output)
    }
}

// ViewModel 中使用
class SettingsViewModel(
    private val preferencesRepository: UserPreferencesRepository
) : ViewModel() {
    
    val userTheme: StateFlow<String> = preferencesRepository.userPreferences
        .map { it.theme }
        .stateIn(viewModelScope, SharingStarted.Lazily, "light")
    
    fun setTheme(theme: String) {
        viewModelScope.launch {
            preferencesRepository.saveUserTheme(theme)
        }
    }
}
```

### 3.3 源码原理

#### ViewModel 的生命周期管理源码

```kotlin
// ViewModelStore 的核心机制
public class ViewModelStore {
    private final HashMap<String, ViewModel> mMap = new HashMap<>();
    
    final void put(String key, ViewModel viewModel) {
        ViewModel oldViewModel = mMap.put(key, viewModel);
        if (oldViewModel != null) {
            oldViewModel.onCleared();
        }
    }
    
    final ViewModel get(String key) {
        return mMap.get(key);
    }
    
    public final void clear() {
        for (ViewModel vm : mMap.values()) {
            vm.onCleared();
        }
        mMap.clear();
    }
}

// Activity 对 ViewModelStore 的持有
public class ComponentActivity extends androidx.core.app.ComponentActivity {
    private ViewModelStore mViewModelStore;
    
    public ViewModelStore getViewModelStore() {
        if (mViewModelStore == null) {
            NonConfigurationInstances nc =
                    (NonConfigurationInstances) getLastNonConfigurationInstance();
            if (nc != null) {
                // 配置变更时恢复之前的 ViewModelStore
                mViewModelStore = nc.viewModelStore;
            }
            if (mViewModelStore == null) {
                mViewModelStore = new ViewModelStore();
            }
        }
        return mViewModelStore;
    }
    
    @Override
    public Object onRetainNonConfigurationInstance() {
        // 配置变更时保留 ViewModelStore
        NonConfigurationInstances nci = new NonConfigurationInstances();
        nci.viewModelStore = mViewModelStore;
        return nci;
    }
}
```

**关键点**：

- ViewModel 通过 `onRetainNonConfigurationInstance()` 在配置变更时被保留
- 每个 Activity/Fragment 都有一个 `ViewModelStore` 用来管理其 ViewModel 实例
- Activity 销毁时会调用 `ViewModelStore.clear()`，进而调用所有 ViewModel 的 `onCleared()`

#### LiveData 的线程安全实现

```kotlin
// LiveData 源码（简化版）
public abstract class LiveData<T> {
    private volatile Object mData;
    private int mVersion = START_VERSION;
    private SafeIterableMap<Observer<? super T>, ObserverWrapper> mObservers =
            new SafeIterableMap<>();
    
    @MainThread
    public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
        if (owner.getLifecycle().getCurrentState() == Lifecycle.State.DESTROYED) {
            return;
        }
        
        LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
        mObservers.putIfAbsent(observer, wrapper);
        owner.getLifecycle().addObserver(wrapper);
    }
    
    @MainThread
    protected void setValue(T value) {
        mVersion++;
        mData = value;
        dispatchValue(null);  // 在主线程分发
    }
    
    protected void postValue(T value) {
        // 切换到主线程
        mPostValueRunnable = () -> setValue(value);
        getMainHandler().post(mPostValueRunnable);
    }
    
    private void dispatchValue(@Nullable ObserverWrapper initiator) {
        if (mDispatchingValue) {
            mDispatchInvalidated = true;
            return;
        }
        
        mDispatchingValue = true;
        do {
            mDispatchInvalidated = false;
            for (Iterator<Map.Entry<Observer<? super T>, ObserverWrapper>> iterator =
                    mObservers.iteratorWithAdditions(); iterator.hasNext(); ) {
                considerNotify(iterator.next().getValue());
            }
        } while (mDispatchInvalidated);
        mDispatchingValue = false;
    }
}

// 生命周期感知的观察者包装
private class LifecycleBoundObserver extends ObserverWrapper implements LifecycleEventObserver {
    @Override
    public void onStateChanged(@NonNull LifecycleOwner source,
            @NonNull Lifecycle.Event event) {
        if (mOwner.getLifecycle().getCurrentState() == DESTROYED) {
            removeObserver(mObserver);
            return;
        }
        activeStateChanged(shouldBeActive());
    }
}
```

**核心机制**：

- `setValue()` 必须在主线程调用，安全分发观察者回调
- `postValue()` 可以从任意线程调用，内部会切换到主线程
- LiveData 持有生命周期所有者的引用，生命周期销毁时自动移除观察者（防止内存泄漏）

#### Room 的编译时代码生成

```kotlin
// Room 编译时生成的 DAO 实现（伪代码）
public class UserDao_Impl extends UserDao {
    private final RoomDatabase __db;
    private final SupportSQLiteStatement __insertionAdapterOfUserEntity;
    
    public UserDao_Impl(RoomDatabase __db) {
        this.__db = __db;
        this.__insertionAdapterOfUserEntity = __createInsertionAdapter();
    }
    
    @Override
    public Object insertUser(UserEntity user, Continuation<? super Unit> continuation) {
        return CoroutinesRoom.execute(__db, false, new Callable<Unit>() {
            @Override
            public Unit call() throws Exception {
                __insertionAdapterOfUserEntity.bindLong(1, user.id);
                __insertionAdapterOfUserEntity.bindString(2, user.name);
                __insertionAdapterOfUserEntity.executeInsert();
                return Unit.INSTANCE;
            }
        }, continuation);
    }
    
    @Override
    public Object getUserById(String userId, Continuation<? super UserEntity> continuation) {
        final String _sql = "SELECT * FROM users WHERE id = ? LIMIT 1";
        final RoomSQLiteQuery _statement = RoomSQLiteQuery.acquire(_sql, 1);
        
        return CoroutinesRoom.execute(__db, true, new Callable<UserEntity>() {
            @Override
            public UserEntity call() throws Exception {
                _statement.bindString(1, userId);
                final Cursor _cursor = DBUtil.query(__db, _statement, false, null);
                try {
                    final UserEntity _result;
                    if (_cursor.moveToFirst()) {
                        _result = __entityCursorConverter(cursor);
                    } else {
                        _result = null;
                    }
                    return _result;
                } finally {
                    _cursor.close();
                }
            }
        }, continuation);
    }
}
```

**关键点**：

- Room 使用 Kotlin Poet 和 KAPT（Kotlin Annotation Processing Tool）在编译时生成 DAO 实现
- 生成的代码使用 `CoroutinesRoom.execute()` 包装查询，在 IO 线程执行
- 参数绑定和结果映射都是自动生成的，确保性能最优

---

## 4. Kotlin Coroutines（协程）

### 4.1 干什么的

Kotlin Coroutines 是 JetBrains 推出的轻量级并发框架，用于简化异步编程。相比传统的回调、RxJava，协程提供了更直观的顺序式代码结构。

### 4.2 怎么用的

#### 基础协程用法

```kotlin
// 启动协程
viewModelScope.launch {
    // 在协程中执行异步操作
    val user = repository.getUser(userId)  // 挂起函数
    updateUI(user)
}

// launch：返回 Job，不返回结果
// async：返回 Deferred，可以获取结果
val userDeferred = viewModelScope.async {
    repository.getUser(userId)
}
val user = userDeferred.await()

// runBlocking：阻塞当前线程直到协程完成（仅用于测试）
val user = runBlocking {
    repository.getUser(userId)
}
```

#### 作用域与生命周期管理

```kotlin
class UserViewModel(
    private val repository: UserRepository
) : ViewModel() {
    
    // viewModelScope 会在 ViewModel.onCleared() 时自动取消
    fun loadUser(userId: String) {
        viewModelScope.launch {
            try {
                val user = repository.getUser(userId)
                // 更新 UI
            } catch (e: CancellationException) {
                // 协程被取消
                throw e
            } catch (e: Exception) {
                // 其他异常处理
            }
        }
    }
}

// lifecycleScope 会在 Lifecycle 销毁时自动取消
class UserFragment : Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        // 协程会在 Fragment 销毁时取消
        lifecycleScope.launch {
            viewModel.uiState.collect { state ->
                updateUI(state)
            }
        }
        
        // 只在 STARTED 状态执行
        lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.uiState.collect { state ->
                    updateUI(state)
                }
            }
        }
    }
}

// 自定义 CoroutineScope
class LocationTracker : DefaultLifecycleObserver {
    private val trackerScope = CoroutineScope(Dispatchers.Main.immediate + Job())
    
    override fun onResume(owner: LifecycleOwner) {
        trackerScope.launch {
            // 追踪位置
        }
    }
    
    override fun onDestroy(owner: LifecycleOwner) {
        trackerScope.cancel()  // 手动取消所有协程
    }
}
```

#### 异常处理

```kotlin
// try-catch 处理异常
viewModelScope.launch {
    try {
        val user = repository.getUser(userId)
    } catch (e: IOException) {
        // 网络异常
    } catch (e: CancellationException) {
        // 协程被取消，必须重新抛出
        throw e
    }
}

// CoroutineExceptionHandler 全局异常处理
val exceptionHandler = CoroutineExceptionHandler { _, exception ->
    println("Exception: ${exception.message}")
}

viewModelScope.launch(exceptionHandler) {
    val user = repository.getUser(userId)
}

// 在 ViewModel 中集中处理异常
class UserViewModel : ViewModel() {
    private val exceptionHandler = CoroutineExceptionHandler { _, exception ->
        _errorState.value = exception.message
    }
    
    private val _errorState = MutableStateFlow<String?>(null)
    val errorState: StateFlow<String?> = _errorState.asStateFlow()
    
    fun loadUser(userId: String) {
        viewModelScope.launch(exceptionHandler) {
            val user = repository.getUser(userId)
        }
    }
}
```

#### Flow - 数据流

```kotlin
// 创建 Flow
fun getUsers(): Flow<User> = flow {
    for (i in 1..5) {
        delay(1000)
        emit(User(id = i))
    }
}

// Flow 转换
fun getUsersFiltered(): Flow<User> = getUsers()
    .filter { it.id % 2 == 0 }  // 过滤
    .map { it.copy(name = it.name.uppercase()) }  // 转换
    .debounce(500)  // 防抖
    .distinctUntilChanged()  // 去重

// Flow 收集
viewModelScope.launch {
    getUsers()
        .catch { e ->
            println("Error: ${e.message}")
        }
        .collect { user ->
            println("User: $user")
        }
}

// Flow 与 SharedFlow
private val _userEvents = MutableSharedFlow<UserEvent>()
val userEvents: SharedFlow<UserEvent> = _userEvents.asSharedFlow()

fun triggerEvent(event: UserEvent) {
    viewModelScope.launch {
        _userEvents.emit(event)
    }
}

// StateFlow - 带状态的 Flow
private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
val uiState: StateFlow<UiState> = _uiState.asStateFlow()

// 收集 Flow 转换为 StateFlow
val userNames: StateFlow<List<String>> = repository.getUsers()
    .map { users -> users.map { it.name } }
    .stateIn(
        scope = viewModelScope,
        started = SharingStarted.Lazily,
        initialValue = emptyList()
    )
```

#### 协程上下文与调度器

```kotlin
// Dispatchers
viewModelScope.launch(Dispatchers.Main) {
    // 主线程执行
    updateUI()
}

viewModelScope.launch(Dispatchers.IO) {
    // IO 线程执行（网络请求、数据库操作）
    val data = repository.fetchData()
}

viewModelScope.launch(Dispatchers.Default) {
    // 默认线程（CPU 密集操作）
    val result = calculateData()
}

viewModelScope.launch(Dispatchers.Unconfined) {
    // 不指定线程（谨慎使用）
}

// 上下文切换
viewModelScope.launch {
    val data = withContext(Dispatchers.IO) {
        repository.fetchData()  // IO 线程
    }
    // 回到主线程
    updateUI(data)
}

// 创建自定义 Dispatcher
val customDispatcher = newSingleThreadContext("CustomThread")

viewModelScope.launch(customDispatcher) {
    // 在单独的线程执行
}

customDispatcher.close()  // 释放资源
```

### 4.3 源码原理

#### 协程挂起与恢复原理

```kotlin
// 挂起函数编译前
suspend fun getUser(userId: String): User {
    delay(1000)
    return fetchFromNetwork(userId)
}

// 编译后（伪代码）：协程被转换为状态机
fun getUser(userId: String, continuation: Continuation<User>): Any {
    when (getState()) {
        0 -> {
            // 初始状态
            delay(1000) { delayResult ->
                state = 1
                getUser(userId, continuation)  // 恢复执行
            }
            return COROUTINE_SUSPENDED
        }
        1 -> {
            // delay 完成后
            val user = fetchFromNetwork(userId)
            continuation.resume(user)
            return user
        }
    }
}

// Continuation 接口
interface Continuation<in T> {
    val context: CoroutineContext
    fun resumeWith(result: Result<T>)
}
```

**关键机制**：

- Kotlin 编译器将挂起函数转换为状态机
- 每个 `await` 或 `delay` 点都是一个状态转移
- 挂起时返回 `COROUTINE_SUSPENDED`，恢复时直接跳转到下一个状态
- 完全没有线程阻塞，效率极高

#### viewModelScope 的实现原理

```kotlin
// ViewModel 中的 viewModelScope 实现
public val ViewModel.viewModelScope: CoroutineScope
    get() {
        val scope: CoroutineScope? = this.getTag(JOB_KEY)
        if (scope != null) {
            return scope
        }
        return setTagIfAbsent(
            JOB_KEY,
            CloseableCoroutineScope(SupervisorJob() + Dispatchers.Main.immediate)
        )
    }

// ViewModel 销毁时取消 viewModelScope
override fun onCleared() {
    val job = getTag<Job>(JOB_KEY)
    job?.cancel()
}

// CloseableCoroutineScope 实现
private class CloseableCoroutineScope(
    context: CoroutineContext
) : CoroutineScope, Closeable {
    override val coroutineContext: CoroutineContext = context
    
    override fun close() {
        coroutineContext.cancel()
    }
}
```

**关键点**：

- `viewModelScope` 是 ViewModel 的扩展属性，延迟初始化
- 使用 `SupervisorJob()` 而不是 `Job()`，子协程异常不会取消其他子协程
- ViewModel 销毁时自动取消所有协程

#### Flow 的背压处理

```kotlin
// Flow 源码（简化版）
public interface Flow<out T> {
    public suspend fun collect(collector: FlowCollector<T>)
}

// Flow 实现背压
public fun <T> flow(
    block: suspend FlowCollector<T>.() -> Unit
): Flow<T> = object : Flow<T> {
    override suspend fun collect(collector: FlowCollector<T>) {
        collector.block()
    }
}

// FlowCollector 接口
public interface FlowCollector<in T> {
    public suspend fun emit(value: T)
}

// 背压演示：如果收集者处理缓慢，emit 会暂停
private val _values = flow {
    for (i in 1..100) {
        emit(i)  // 如果收集者处理缓慢，此处会挂起等待
        delay(100)
    }
}

viewModelScope.launch {
    _values
        .collect { value ->
            delay(500)  // 处理缓慢，emit 会等待
            println(value)
        }
}
```

**背压机制**：

- `emit()` 是挂起函数，当下游收集者处理缓慢时会自动暂停上游
- Flow 天然支持背压，不需要额外配置
- RxJava 需要显式配置背压策略（如 `onBackpressureBuffer()`）

---

## 5. Retrofit + OkHttp 网络请求

### 5.1 干什么的

Retrofit 是 Square 公司推出的类型安全的 HTTP 客户端库，基于 OkHttp 构建。OkHttp 是底层网络库，处理连接、超时、重试等。

### 5.2 怎么用的

#### 基础 API 定义

```kotlin
// 定义 API 接口
interface UserApiService {
    
    @GET("users/{userId}")
    suspend fun getUser(@Path("userId") userId: String): ApiResponse<User>
    
    @POST("users")
    suspend fun createUser(@Body user: User): ApiResponse<User>
    
    @PUT("users/{userId}")
    suspend fun updateUser(
        @Path("userId") userId: String,
        @Body user: User
    ): ApiResponse<User>
    
    @DELETE("users/{userId}")
    suspend fun deleteUser(@Path("userId") userId: String): ApiResponse<Unit>
    
    // Query 参数
    @GET("users")
    suspend fun getUserList(
        @Query("page") page: Int,
        @Query("limit") limit: Int,
        @Query("filter") filter: String? = null
    ): ApiResponse<List<User>>
    
    // 文件上传
    @Multipart
    @POST("upload")
    suspend fun uploadFile(
        @Part("description") description: RequestBody,
        @Part file: MultipartBody.Part
    ): ApiResponse<UploadResult>
    
    // 流式下载
    @GET("files/{fileId}")
    suspend fun downloadFile(@Path("fileId") fileId: String): ResponseBody
}

// 响应数据包装
data class ApiResponse<T>(
    val code: Int,
    val message: String,
    val data: T?
)

// 创建 Retrofit 实例
object RetrofitBuilder {
    private const val BASE_URL = "https://api.example.com/"
    
    private val okHttpClient = OkHttpClient.Builder()
        .connectTimeout(15, TimeUnit.SECONDS)
        .readTimeout(15, TimeUnit.SECONDS)
        .writeTimeout(15, TimeUnit.SECONDS)
        .addInterceptor(RequestLoggingInterceptor())  // 日志拦截器
        .addInterceptor(TokenInterceptor())  // Token 自动注入
        .addNetworkInterceptor(ResponseLoggingInterceptor())  // 响应日志
        .retryOnConnectionFailure(true)
        .build()
    
    val retrofit = Retrofit.Builder()
        .baseUrl(BASE_URL)
        .client(okHttpClient)
        .addConverterFactory(GsonConverterFactory.create())  // JSON 序列化
        .addCallAdapterFactory(RxJava2CallAdapterFactory.create())  // 支持 RxJava
        .build()
    
    val apiService = retrofit.create(UserApiService::class.java)
}
```

#### 拦截器实现

**Token 自动刷新拦截器**

```kotlin
class TokenInterceptor(
    private val tokenManager: TokenManager
) : Interceptor {
    
    override fun intercept(chain: Interceptor.Chain): Response {
        val originalRequest = chain.request()
        
        // 检查 Token 是否过期
        if (tokenManager.isTokenExpired()) {
            val newToken = tokenManager.refreshToken()
            return chain.proceed(originalRequest.addAuthHeader(newToken))
        }
        
        val request = originalRequest.addAuthHeader(tokenManager.getToken())
        val response = chain.proceed(request)
        
        // 响应 401，自动刷新 Token 重试
        if (response.code == 401) {
            synchronized(this) {
                val newToken = tokenManager.refreshToken()
                val retryRequest = originalRequest.addAuthHeader(newToken)
                response.close()
                return chain.proceed(retryRequest)
            }
        }
        
        return response
    }
    
    private fun Request.addAuthHeader(token: String): Request {
        return this.newBuilder()
            .addHeader("Authorization", "Bearer $token")
            .build()
    }
}

// Token 管理器
class TokenManager(
    private val localStorage: UserPreferencesRepository
) {
    
    suspend fun getToken(): String {
        return localStorage.getAccessToken()
    }
    
    fun isTokenExpired(): Boolean {
        val expiryTime = getTokenExpiryTime()
        return System.currentTimeMillis() > expiryTime
    }
    
    suspend fun refreshToken(): String {
        val refreshToken = localStorage.getRefreshToken()
        val newTokenResponse = apiService.refreshToken(refreshToken)
        
        localStorage.saveAccessToken(newTokenResponse.accessToken)
        localStorage.saveAccessTokenExpiry(newTokenResponse.expiresIn)
        
        return newTokenResponse.accessToken
    }
}
```

**日志拦截器**

```kotlin
class RequestLoggingInterceptor : Interceptor {
    
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request()
        
        val startTime = System.nanoTime()
        println("请求开始 --> ${request.method} ${request.url}")
        println("请求头:")
        request.headers.forEach { (name, value) ->
            println("$name: $value")
        }
        
        return try {
            val response = chain.proceed(request)
            val duration = (System.nanoTime() - startTime) / 1_000_000
            
            println("<-- 响应完成 (耗时: ${duration}ms)")
            println("响应码: ${response.code}")
            
            response
        } catch (e: Exception) {
            val duration = (System.nanoTime() - startTime) / 1_000_000
            println("<-- 请求失败 (耗时: ${duration}ms)")
            println("异常: ${e.message}")
            throw e
        }
    }
}

class ResponseLoggingInterceptor : Interceptor {
    
    override fun intercept(chain: Interceptor.Chain): Response {
        val response = chain.proceed(chain.request())
        
        val responseBody = response.body?.string() ?: ""
        println("响应体: $responseBody")
        
        // 因为 ResponseBody 只能读一次，需要重新构建
        return response.newBuilder()
            .body(ResponseBody.create(response.body?.contentType(), responseBody))
            .build()
    }
}
```

#### 网络异常处理

```kotlin
sealed class NetworkException : Exception() {
    object NoInternetConnection : NetworkException()
    object RequestTimeout : NetworkException()
    object ServerError : NetworkException()
    data class ApiError(val code: Int, val message: String) : NetworkException()
}

class ApiErrorHandler(
    private val apiResponse: ApiResponse<*>
) : NetworkException(apiResponse.message) {
    val code = apiResponse.code
}

// Repository 中的异常处理
class UserRepositoryImpl(
    private val apiService: UserApiService,
    private val localDataSource: UserLocalDataSource
) : UserRepository {
    
    override suspend fun getUser(userId: String): Result<User> {
        return withContext(Dispatchers.IO) {
            try {
                val response = apiService.getUser(userId)
                
                if (response.code == 200 && response.data != null) {
                    // 成功，保存到本地
                    localDataSource.saveUser(response.data)
                    Result.success(response.data)
                } else {
                    Result.failure(
                        ApiErrorHandler(response)
                    )
                }
            } catch (e: Exception) {
                // 网络异常时，尝试从本地读取
                handleNetworkError(e, userId)
            }
        }
    }
    
    private suspend fun handleNetworkError(
        exception: Exception,
        userId: String
    ): Result<User> {
        return when (exception) {
            is SocketTimeoutException -> {
                Result.failure(NetworkException.RequestTimeout)
            }
            is UnknownHostException -> {
                Result.failure(NetworkException.NoInternetConnection)
            }
            is HttpException -> {
                if (exception.code() == 500) {
                    Result.failure(NetworkException.ServerError)
                } else {
                    Result.failure(exception)
                }
            }
            else -> {
                // 尝试从本地读取缓存
                val localUser = localDataSource.getUser(userId)
                if (localUser != null) {
                    Result.success(localUser)
                } else {
                    Result.failure(exception)
                }
            }
        }
    }
}
```

#### 文件上传下载

```kotlin
// 文件上传
class FileUploadRepository(
    private val apiService: UserApiService
) {
    
    suspend fun uploadFile(filePath: String): Result<UploadResult> {
        return try {
            val file = File(filePath)
            val fileRequestBody = file.asRequestBody("application/octet-stream".toMediaType())
            val filePart = MultipartBody.Part.createFormData("file", file.name, fileRequestBody)
            val descriptionPart = RequestBody.create("text/plain".toMediaType(), "File Description")
            
            val response = apiService.uploadFile(descriptionPart, filePart)
            
            if (response.code == 200) {
                Result.success(response.data!!)
            } else {
                Result.failure(Exception(response.message))
            }
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
}

// 文件下载（带进度）
class FileDownloadRepository(
    private val apiService: UserApiService
) {
    
    suspend fun downloadFile(
        fileId: String,
        outputPath: String,
        progressCallback: (progress: Int) -> Unit
    ): Result<String> {
        return try {
            val responseBody = apiService.downloadFile(fileId)
            val file = File(outputPath)
            
            responseBody.byteStream().use { input ->
                file.outputStream().use { output ->
                    val totalSize = responseBody.contentLength()
                    var downloadedSize = 0L
                    val buffer = ByteArray(8192)
                    var bytesRead: Int
                    
                    while (input.read(buffer).also { bytesRead = it } != -1) {
                        output.write(buffer, 0, bytesRead)
                        downloadedSize += bytesRead
                        val progress = (downloadedSize * 100 / totalSize).toInt()
                        progressCallback(progress)
                    }
                }
            }
            
            Result.success(outputPath)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
}
```

### 5.3 源码原理

#### Retrofit 的动态代理原理

```kotlin
// Retrofit 创建 API 实例时的原理
val apiService = retrofit.create(UserApiService::class.java)

// 其中 create() 的实现
fun <T> create(service: Class<T>): T {
    val annotations = service.annotations
    
    // 使用 Java 动态代理
    return Proxy.newProxyInstance(
        service.classLoader,
        arrayOf<Class<*>>(service)
    ) { proxy, method, args ->
        // method: getUserUser(userId)
        // 在这里分析方法注解和参数注解
        val requestFactory = RequestFactory.parseAnnotations(service, method)
        val callAdapter = callAdapter(method.returnType)
        val responseConverter = responseConverter(method.genericReturnType)
        
        val call = okHttpClient.newCall(requestFactory.build(args))
        val callExecution = callAdapter.adapt(call)
        
        callExecution
    }.unsafeCast<T>()
}

// 动态代理调用流程
@GET("users/{userId}")
suspend fun getUser(@Path("userId") userId: String): ApiResponse<User>

// 代理会:
// 1. 解析 @GET, @Path 注解
// 2. 构建 Request: GET /users/user123
// 3. 执行 OkHttp 调用
// 4. 通过 GsonConverterFactory 将响应转换为 ApiResponse<User>
// 5. 返回 suspend 函数结果
```

**关键点**：

- Retrofit 使用 Java 动态代理拦截接口方法调用
- 运行时解析所有注解（@GET、@Path、@Query 等）
- 基于注解信息构建 Request 对象
- 使用 `CallAdapter` 支持不同的异步模式（Kotlin Coroutines、RxJava）

#### OkHttp 连接池原理

```kotlin
// OkHttp 连接池实现
class ConnectionPool(
    private val maxIdleConnections: Int,
    private val keepAliveDurationNs: Long
) {
    private val connections = ConcurrentLinkedDeque<RealConnection>()
    
    fun get(address: Address): RealConnection? {
        // 查找可复用的连接
        for (connection in connections) {
            if (connection.isEligible(address)) {
                return connection
            }
        }
        return null
    }
    
    fun put(connection: RealConnection) {
        // 放入连接池
        connections.add(connection)
        
        // 启动清理任务，移除超时连接
        cleanupThread.schedule(this::evictConnections, keepAliveDurationNs)
    }
    
    private fun evictConnections() {
        val now = System.nanoTime()
        for (connection in connections) {
            if (connection.idleAtNanos + keepAliveDurationNs < now) {
                connection.close()
                connections.remove(connection)
            }
        }
    }
}

// TCP 连接复用
class RealConnection : Connection {
    var socket: Socket? = null
    var source: BufferedSource? = null
    var sink: BufferedSink? = null
    
    // HTTP/2 多路复用
    var http2Connection: Http2Connection? = null
    
    fun isHealthy(doExtensiveChecks: Boolean): Boolean {
        if (socket?.isClosed == true) return false
        if (socket?.isConnected == false) return false
        
        // 检查 socket 是否可用
        return if (!doExtensiveChecks) true
        else try {
            // 尝试读取一个字节检查连接健康度
            socket!!.soTimeout == 1
            true
        } catch (e: Exception) {
            false
        }
    }
}
```

**连接池优化**：

- TCP 连接复用，避免频繁创建销毁连接
- HTTP/2 多路复用（单个 TCP 连接上运行多个 HTTP 请求）
- 自动清理超时连接，内存高效

#### OkHttp 拦截器链原理

```kotlin
// Interceptor 接口
interface Interceptor {
    fun intercept(chain: Chain): Response
    
    interface Chain {
        fun proceed(request: Request): Response
    }
}

// RealInterceptorChain 实现
class RealInterceptorChain(
    private val interceptors: List<Interceptor>,
    private val index: Int,
    private val request: Request
) : Interceptor.Chain {
    
    override fun proceed(request: Request): Response {
        if (index >= interceptors.size) {
            throw AssertionError("No more interceptors")
        }
        
        // 创建下一个链
        val next = RealInterceptorChain(interceptors, index + 1, request)
        val interceptor = interceptors[index]
        
        return interceptor.intercept(next)
    }
}

// 拦截器执行顺序
// 应用拦截器 (Application Interceptor) -> 
// 网络拦截器 (Network Interceptor) -> 
// 连接拦截器 (ConnectionInterceptor) -> 
// 真实请求

// 示例：
addInterceptor(TokenInterceptor())  // 应用拦截器
addNetworkInterceptor(ResponseLoggingInterceptor())  // 网络拦截器
```

**拦截器链的设计模式**：

- 责任链模式的经典实现
- 应用拦截器在连接前执行，可以修改请求头、超时等
- 网络拦截器在连接后执行，可以看到真实的网络交互

---

## 6. RecyclerView 与 DiffUtil

### 6.1 干什么的

RecyclerView 是 Android 提供的强大的列表组件，支持数据的高效回收和复用。DiffUtil 是用来计算列表差异的工具类，能够精确地更新列表中变化的项，避免全量刷新。

### 6.2 怎么用的

#### 基础 RecyclerView 设置

```kotlin
// 定义 ViewHolder
class UserItemViewHolder(
    private val binding: ItemUserBinding
) : RecyclerView.ViewHolder(binding.root) {
    
    fun bind(user: User, onItemClick: (User) -> Unit) {
        binding.apply {
            userName.text = user.name
            userEmail.text = user.email
            
            root.setOnClickListener {
                onItemClick(user)
            }
            
            // Glide 加载头像
            Glide.with(userAvatar)
                .load(user.avatarUrl)
                .placeholder(R.drawable.placeholder)
                .error(R.drawable.error)
                .into(userAvatar)
        }
    }
}

// 定义 Adapter
class UserAdapter(
    private val onItemClick: (User) -> Unit
) : RecyclerView.Adapter<UserItemViewHolder>() {
    
    private val items = mutableListOf<User>()
    
    override fun onCreateViewHolder(
        parent: ViewGroup,
        viewType: Int
    ): UserItemViewHolder {
        val binding = ItemUserBinding.inflate(
            LayoutInflater.from(parent.context),
            parent,
            false
        )
        return UserItemViewHolder(binding)
    }
    
    override fun getItemCount() = items.size
    
    override fun onBindViewHolder(holder: UserItemViewHolder, position: Int) {
        val user = items[position]
        holder.bind(user, onItemClick)
    }
    
    fun updateItems(newItems: List<User>) {
        items.clear()
        items.addAll(newItems)
        notifyDataSetChanged()  // 全量刷新（不高效）
    }
}

// Activity 中使用
class UserListActivity : AppCompatActivity() {
    
    private val viewModel: UserListViewModel by viewModels()
    private lateinit var adapter: UserAdapter
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        adapter = UserAdapter { user ->
            startActivity(UserDetailActivity.intent(this, user.id))
        }
        
        binding.recyclerView.apply {
            layoutManager = LinearLayoutManager(this@UserListActivity)
            adapter = this@UserListActivity.adapter
        }
        
        lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.users.collect { users ->
                    adapter.updateItems(users)
                }
            }
        }
    }
}
```

#### 使用 DiffUtil 优化更新

```kotlin
// DiffUtil.ItemCallback 定义
class UserDiffCallback : DiffUtil.ItemCallback<User>() {
    
    override fun areItemsTheSame(oldItem: User, newItem: User): Boolean {
        // 比较 ID 判断是否是同一项
        return oldItem.id == newItem.id
    }
    
    override fun areContentsTheSame(oldItem: User, newItem: User): Boolean {
        // 比较内容判断内容是否相同
        return oldItem == newItem
    }
    
    override fun getChangePayload(oldItem: User, newItem: User): Any? {
        // 返回变化的部分内容（可选）
        if (oldItem.name != newItem.name) {
            return "name"
        }
        if (oldItem.avatarUrl != newItem.avatarUrl) {
            return "avatar"
        }
        return null
    }
}

// 使用 ListAdapter（推荐）
class UserListAdapter(
    private val onItemClick: (User) -> Unit
) : ListAdapter<User, UserItemViewHolder>(UserDiffCallback()) {
    
    override fun onCreateViewHolder(
        parent: ViewGroup,
        viewType: Int
    ): UserItemViewHolder {
        val binding = ItemUserBinding.inflate(
            LayoutInflater.from(parent.context),
            parent,
            false
        )
        return UserItemViewHolder(binding)
    }
    
    override fun onBindViewHolder(holder: UserItemViewHolder, position: Int) {
        val user = getItem(position)
        holder.bind(user, onItemClick)
    }
    
    // 支持 payload 的部分更新
    override fun onBindViewHolder(
        holder: UserItemViewHolder,
        position: Int,
        payloads: List<Any>
    ) {
        if (payloads.isEmpty()) {
            super.onBindViewHolder(holder, position, payloads)
        } else {
            // 只更新变化的部分
            val user = getItem(position)
            when (payloads[0]) {
                "name" -> holder.binding.userName.text = user.name
                "avatar" -> {
                    Glide.with(holder.binding.userAvatar)
                        .load(user.avatarUrl)
                        .into(holder.binding.userAvatar)
                }
            }
        }
    }
}

// ViewModel 中
class UserListViewModel(
    private val repository: UserRepository
) : ViewModel() {
    
    val users: StateFlow<List<User>> = repository.observeUsers()
        .stateIn(viewModelScope, SharingStarted.Lazily, emptyList())
}

// Activity 中使用
class UserListActivity : AppCompatActivity() {
    
    private val viewModel: UserListViewModel by viewModels()
    private lateinit var adapter: UserListAdapter
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        adapter = UserListAdapter { user ->
            startActivity(UserDetailActivity.intent(this, user.id))
        }
        
        binding.recyclerView.adapter = adapter
        
        lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.users.collect { users ->
                    // ListAdapter 会自动计算差异并高效更新
                    adapter.submitList(users)
                }
            }
        }
    }
}
```

#### 多类型 Item

```kotlin
// 定义数据类型
sealed class ListItem {
    data class UserItem(val user: User) : ListItem()
    data class HeaderItem(val title: String) : ListItem()
    data class FooterItem(val text: String) : ListItem()
}

// 定义 ViewType
sealed class ListItemViewType {
    object User : ListItemViewType()
    object Header : ListItemViewType()
    object Footer : ListItemViewType()
}

// 多类型 Adapter
class MultiTypeListAdapter(
    private val onUserClick: (User) -> Unit
) : ListAdapter<ListItem, RecyclerView.ViewHolder>(ListItemDiffCallback()) {
    
    override fun getItemViewType(position: Int): Int {
        return when (getItem(position)) {
            is ListItem.UserItem -> 0
            is ListItem.HeaderItem -> 1
            is ListItem.FooterItem -> 2
        }
    }
    
    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): RecyclerView.ViewHolder {
        return when (viewType) {
            0 -> UserItemViewHolder(
                ItemUserBinding.inflate(LayoutInflater.from(parent.context), parent, false)
            )
            1 -> HeaderItemViewHolder(
                ItemHeaderBinding.inflate(LayoutInflater.from(parent.context), parent, false)
            )
            2 -> FooterItemViewHolder(
                ItemFooterBinding.inflate(LayoutInflater.from(parent.context), parent, false)
            )
            else -> throw IllegalArgumentException("Unknown viewType: $viewType")
        }
    }
    
    override fun onBindViewHolder(holder: RecyclerView.ViewHolder, position: Int) {
        when (holder) {
            is UserItemViewHolder -> {
                val item = getItem(position) as ListItem.UserItem
                holder.bind(item.user, onUserClick)
            }
            is HeaderItemViewHolder -> {
                val item = getItem(position) as ListItem.HeaderItem
                holder.bind(item.title)
            }
            is FooterItemViewHolder -> {
                val item = getItem(position) as ListItem.FooterItem
                holder.bind(item.text)
            }
        }
    }
}

class ListItemDiffCallback : DiffUtil.ItemCallback<ListItem>() {
    override fun areItemsTheSame(oldItem: ListItem, newItem: ListItem): Boolean {
        return when {
            oldItem is ListItem.UserItem && newItem is ListItem.UserItem ->
                oldItem.user.id == newItem.user.id
            oldItem is ListItem.HeaderItem && newItem is ListItem.HeaderItem ->
                oldItem.title == newItem.title
            oldItem is ListItem.FooterItem && newItem is ListItem.FooterItem ->
                oldItem.text == newItem.text
            else -> false
        }
    }
    
    override fun areContentsTheSame(oldItem: ListItem, newItem: ListItem): Boolean {
        return oldItem == newItem
    }
}
```

#### 列表性能优化

```kotlin
// 1. 禁用预加载
binding.recyclerView.apply {
    setItemViewCacheSize(0)  // 禁用缓存
    setHasFixedSize(true)  // 列表大小固定
}

// 2. 设置合理的滚动监听
binding.recyclerView.addOnScrollListener(object : RecyclerView.OnScrollListener() {
    override fun onScrollStateChanged(recyclerView: RecyclerView, newState: Int) {
        when (newState) {
            RecyclerView.SCROLL_STATE_DRAGGING, RecyclerView.SCROLL_STATE_SETTLING -> {
                // 停止 Glide 加载
                Glide.with(this@UserListActivity).pauseRequests()
            }
            RecyclerView.SCROLL_STATE_IDLE -> {
                // 恢复加载
                Glide.with(this@UserListActivity).resumeRequests()
            }
        }
    }
})

// 3. 优化 ViewHolder 的 bind 方法
class UserItemViewHolder(
    private val binding: ItemUserBinding
) : RecyclerView.ViewHolder(binding.root) {
    
    fun bind(user: User, onItemClick: (User) -> Unit) {
        // 复用线程局部变量避免频繁创建对象
        binding.apply {
            userName.text = user.name
            userEmail.text = user.email
            
            // 取消之前的请求
            Glide.with(userAvatar).clear(userAvatar)
            
            // 使用 thumbnail 预加载
            Glide.with(userAvatar)
                .load(user.avatarUrl)
                .placeholder(R.drawable.placeholder)
                .into(userAvatar)
            
            root.setOnClickListener {
                onItemClick(user)
            }
        }
    }
}

// 4. 分页加载
class PaginatedUserAdapter : ListAdapter<User, UserItemViewHolder>(UserDiffCallback()) {
    
    private var onLoadMore: (() -> Unit)? = null
    private val PAGE_LOAD_THRESHOLD = 5
    
    fun setOnLoadMoreListener(onLoadMore: () -> Unit) {
        this.onLoadMore = onLoadMore
    }
    
    override fun onBindViewHolder(holder: UserItemViewHolder, position: Int) {
        val user = getItem(position)
        holder.bind(user) {}
        
        // 距离底部 PAGE_LOAD_THRESHOLD 时触发加载
        if (position >= itemCount - PAGE_LOAD_THRESHOLD) {
            onLoadMore?.invoke()
        }
    }
    
    override fun getItemCount() = super.getItemCount()
}
```

### 6.3 源码原理

#### DiffUtil 算法原理

```kotlin
// DiffUtil 内部使用 Myers 差异算法
// Myers 算法是一种高效的行差异算法

class DiffUtil {
    
    companion object {
        fun calculateDiff(
            callback: ItemCallback<T>,
            detectMoves: Boolean = true
        ): DiffResult {
            // 使用 Myers 差异算法计算最小编辑距离
            val matrix = computeSnakes(
                callback.oldListSize(),
                callback.newListSize(),
                callback
            )
            
            return DiffResult(matrix, callback, detectMoves)
        }
    }
    
    private fun computeSnakes(
        oldSize: Int,
        newSize: Int,
        callback: ItemCallback<T>
    ): IntArray {
        // Myers 算法的核心：计算编辑距离矩阵
        // 时间复杂度: O(N + M + D²)，其中 D 是差异数量
        
        val MAX_STEPS = (oldSize + newSize + 1) / 2
        val diagonals = IntArray((2 * MAX_STEPS + 1) * 2)
        
        // 前向扫描
        for (d in 0..MAX_STEPS) {
            for (k in -d..d step 2) {
                val index = ((d + 1) * (2 * MAX_STEPS + 1)) + k + MAX_STEPS
                
                var x = if (k == -d || (k != d && diagonals[index - 2] < diagonals[index + 2])) {
                    diagonals[index + 2]
                } else {
                    diagonals[index - 2] + 1
                }
                
                var y = x - k
                
                while (x < oldSize && y < newSize && callback.areItemsTheSame(x, y)) {
                    x++
                    y++
                }
                
                diagonals[index] = x
                
                if (x >= oldSize && y >= newSize) {
                    return diagonals
                }
            }
        }
        
        return diagonals
    }
}

// DiffResult 应用差异
class DiffResult(
    private val snakes: IntArray,
    private val callback: ItemCallback<T>,
    private val detectMoves: Boolean
) {
    
    fun dispatchUpdatesTo(adapter: ListAdapter<T, *>) {
        // 逐个应用更新
        val updates = mutableListOf<Update>()
        
        // 遍历 snakes 矩阵提取更新操作
        for (snake in snakes) {
            when (snake) {
                is Insert -> {
                    adapter.notifyItemInserted(snake.position)
                    updates.add(snake)
                }
                is Delete -> {
                    adapter.notifyItemRemoved(snake.position)
                    updates.add(snake)
                }
                is Move -> {
                    adapter.notifyItemMoved(snake.from, snake.to)
                    updates.add(snake)
                }
                is Update -> {
                    val payload = callback.getChangePayload(snake.oldItem, snake.newItem)
                    adapter.notifyItemChanged(snake.position, payload)
                    updates.add(snake)
                }
            }
        }
    }
}
```

**Myers 算法优势**：

- 时间复杂度 O(N + M + D²)，其中 D 是最小编辑距离
- 相比 Levenshtein 算法的 O(NM) 快很多
- 能够检测插入、删除、移动、修改操作

#### RecyclerView 的 ViewHolder 缓存机制

```kotlin
// RecyclerView.Recycler 实现
class Recycler {
    // 一级缓存：最近离屏的 ViewHolder（默认 2 个）
    private val mAttachedScrap = ArrayList<ViewHolder>()
    
    // 二级缓存：完全离屏的 ViewHolder（默认 5 个）
    private val mCachedViews = ArrayList<ViewHolder>()
    
    // 三级缓存：自定义 ViewHolder 缓存池
    private val mViewCacheExtension: ViewCacheExtension? = null
    
    // 四级缓存：通用 ViewHolder 对象池
    private val mRecycledViewPool = RecycledViewPool()
    
    fun getViewForPosition(position: Int): ViewHolder {
        // 查找顺序：一级 -> 二级 -> 三级 -> 四级 -> 创建新 ViewHolder
        
        // 一级缓存（最快，仅用于离屏快要进屏的 ViewHolder）
        val scrappedView = mAttachedScrap.find { it.adapterPosition == position }
        if (scrappedView != null) return scrappedView
        
        // 二级缓存（快速，存储最近离屏的 ViewHolder）
        val cachedView = mCachedViews.find { it.adapterPosition == position }
        if (cachedView != null) {
            mCachedViews.remove(cachedView)
            return cachedView
        }
        
        // 三级缓存（自定义）
        val extensionView = mViewCacheExtension?.getViewForPositionAndType(position, type)
        if (extensionView != null) return extensionView
        
        // 四级缓存（对象池）
        val pooledView = mRecycledViewPool.getRecycledView(type)
        if (pooledView != null) {
            adapter.bindViewHolder(pooledView, position)
            return pooledView
        }
        
        // 创建新 ViewHolder
        val newHolder = adapter.onCreateViewHolder(parent, type)
        adapter.bindViewHolder(newHolder, position)
        return newHolder
    }
    
    fun recycleView(holder: ViewHolder) {
        // ViewHolder 离屏时回收
        val transient = holder.itemView.hasTransientState()
        
        if (transient) {
            // 一级缓存
            mAttachedScrap.add(holder)
        } else if (mCachedViews.size < DEFAULT_CACHE_SIZE) {
            // 二级缓存
            mCachedViews.add(holder)
        } else {
            // 放入对象池（四级）
            mRecycledViewPool.putRecycledView(holder)
        }
    }
}

// RecycledViewPool：对象池
class RecycledViewPool {
    private val mScrap = HashMap<Int, Queue<ViewHolder>>()
    
    fun getRecycledView(viewType: Int): ViewHolder? {
        val queue = mScrap[viewType] ?: return null
        return queue.poll()
    }
    
    fun putRecycledView(holder: ViewHolder) {
        val viewType = holder.itemViewType
        var queue = mScrap[viewType]
        if (queue == null) {
            queue = LinkedList()
            mScrap[viewType] = queue
        }
        
        if (queue.size < DEFAULT_MAX_SCRAP) {
            holder.reset()
            queue.offer(holder)
        }
    }
}
```

**缓存层级**：

1. **一级缓存**（Scrap）：即将进屏的 ViewHolder，最快
2. **二级缓存**（Cached）：最近离屏的 ViewHolder
3. **三级缓存**（ViewCacheExtension）：自定义缓存
4. **四级缓存**（RecycledViewPool）：对象池，可跨 RecyclerView 共享

---

## 7. Glide 图片加载库

### 7.1 干什么的

Glide 是 Google 推荐的图片加载库，提供了高效的图片加载、缓存、预加载和转换功能。

### 7.2 怎么用的

#### 基础用法

```kotlin
// 简单加载
Glide.with(context)
    .load("https://example.com/image.jpg")
    .into(imageView)

// 使用占位图和错误图
Glide.with(context)
    .load(imageUrl)
    .placeholder(R.drawable.placeholder)  // 加载中显示
    .error(R.drawable.error)  // 加载失败显示
    .into(imageView)

// 设置加载尺寸
Glide.with(context)
    .load(imageUrl)
    .override(300, 300)  // 指定宽高
    .into(imageView)

// 变换（裁剪、圆角等）
Glide.with(context)
    .load(imageUrl)
    .apply(
        RequestOptions()
            .fitCenter()  // 适应中心
            .circleCrop()  // 圆形裁剪
    )
    .into(imageView)
```

#### 缓存管理

```kotlin
// 内存缓存和磁盘缓存
Glide.with(context)
    .load(imageUrl)
    .diskCacheStrategy(DiskCacheStrategy.ALL)  // 缓存原始图片和转换后的图片
    .skipMemoryCache(false)  // 启用内存缓存
    .into(imageView)

class GlideConfiguration : GlideModule {
    override fun applyOptions(context: Context, builder: GlideBuilder) {
        // 设置内存缓存大小
        val memoryCache = MemoryCacheAdapter.Builder(context)
            .setDecodeFormat(DecodeFormat.PREFER_ARGB_8888)
            .setArrayPoolSize(1024 * 1024)  // 1MB
            .build()
        builder.setMemoryCache(memoryCache)
        
        // 设置磁盘缓存大小
        val diskCache = InternalCacheDiskCacheFactory(context, 100 * 1024 * 1024)  // 100MB
        builder.setDiskCache(diskCache)
    }
}

// 清除缓存
Glide.get(context).clearMemory()  // 清除内存缓存（主线程）

Glide.get(context).clearDiskCache()  // 清除磁盘缓存（后台线程）
```

#### 预加载与图片转换

```kotlin
// 预加载
Glide.with(context)
    .load(imageUrl)
    .preload(300, 300)  // 预加载到缓存，不显示到 View

// 下载到本地
Glide.with(context)
    .downloadOnly()
    .load(imageUrl)
    .into(object : SimpleTarget<File>() {
        override fun onResourceReady(resource: File, transition: Transition<in File>?) {
            Log.d("Glide", "Image saved to: ${resource.absolutePath}")
        }
    })

// 获取 Bitmap（不推荐在主线程）
Glide.with(context)
    .asBitmap()
    .load(imageUrl)
    .into(object : CustomTarget<Bitmap>(300, 300) {
        override fun onResourceReady(resource: Bitmap, transition: Transition<in Bitmap>?) {
            imageView.setImageBitmap(resource)
        }
        
        override fun onLoadCleared(placeholder: Drawable?) {
            imageView.setImageDrawable(placeholder)
        }
    })

// 自定义转换
class RoundedCornersTransform(val radius: Int) : Transformation<Bitmap> {
    override fun transform(
        context: Context,
        resource: Bitmap,
        outWidth: Int,
        outHeight: Int
    ): Bitmap {
        return Bitmap.createBitmap(resource.width, resource.height, Bitmap.Config.ARGB_8888).apply {
            val canvas = Canvas(this)
            val paint = Paint(Paint.ANTI_ALIAS_FLAG)
            val path = Path()
            path.addRoundRect(
                RectF(0f, 0f, resource.width.toFloat(), resource.height.toFloat()),
                radius.toFloat(),
                radius.toFloat(),
                Path.Direction.CW
            )
            canvas.clipPath(path)
            canvas.drawBitmap(resource, 0f, 0f, paint)
        }
    }
    
    override fun updateDiskCacheKey(messageDigest: MessageDigest) {
        messageDigest.update("RoundedCorners:$radius".toByteArray())
    }
}

// 使用自定义转换
Glide.with(context)
    .load(imageUrl)
    .transform(RoundedCornersTransform(16))
    .into(imageView)
```

#### 图片压缩

```kotlin
// 采样压缩
class BitmapTranscoder : ResourceTranscoder<Bitmap, Drawable> {
    override fun transcode(toTranscode: Resource<Bitmap>, options: Options): Resource<Drawable>? {
        // 应用采样压缩
        val sampledBitmap = BitmapFactory.Options().apply {
            inSampleSize = calculateInSampleSize(original.width, original.height, targetWidth, targetHeight)
        }.let {
            BitmapFactory.decodeStream(inputStream, null, it)
        }
        
        return DrawableResource(BitmapDrawable(context.resources, sampledBitmap))
    }
    
    private fun calculateInSampleSize(
        originalWidth: Int,
        originalHeight: Int,
        targetWidth: Int,
        targetHeight: Int
    ): Int {
        var inSampleSize = 1
        
        if (originalHeight > targetHeight || originalWidth > targetWidth) {
            val heightRatio = Math.round(originalHeight.toFloat() / targetHeight)
            val widthRatio = Math.round(originalWidth.toFloat() / targetWidth)
            
            inSampleSize = if (heightRatio < widthRatio) heightRatio else widthRatio
        }
        
        return inSampleSize
    }
}
```

### 7.3 源码原理

#### Glide 的引擎和缓存层

```kotlin
// Glide 引擎原理
class Engine(
    private val memoryCache: MemoryCache,
    private val diskCacheProvider: DiskCacheProvider,
    private val diskCacheExecutor: Executor,
    private val sourceExecutor: Executor,
    private val sourceUnlimitedExecutor: Executor,
    private val animationExecutor: Executor
) {
    
    private val jobs = HashMap<Key, EngineJob<*>>()
    private val resourceCache = HashMap<Key, Resource<*>>()
    
    fun load(
        glideContext: GlideContext,
        model: Any,
        requestOptions: RequestOptions,
        callback: ResourceCallback
    ): EngineResource<*>? {
        
        // 1. 检查内存缓存
        val memCacheKey = requestOptions.signature
        val fromMemCache = memoryCache.get(memCacheKey)
        if (fromMemCache != null) {
            callback.onResourceReady(fromMemCache)
            return fromMemCache
        }
        
        // 2. 查找或创建 EngineJob
        val key = EngineKey(model, requestOptions)
        var engineJob = jobs[key]
        
        if (engineJob == null) {
            // 3. 创建新 EngineJob，触发图片加载流程
            engineJob = EngineJob(
                diskCacheExecutor,
                sourceExecutor,
                sourceUnlimitedExecutor,
                animationExecutor,
                key,
                this
            )
            
            // 4. 创建 DecodeJob（真实的加载工作）
            val decodeJob = DecodeJob<Any>(
                glideContext,
                model,
                requestOptions,
                engineJob
            )
            
            engineJob.addCallback(callback)
            jobs[key] = engineJob
            engineJob.start(decodeJob)
        } else {
            engineJob.addCallback(callback)
        }
        
        return null
    }
}

// DecodeJob 的加载流程
class DecodeJob<R> : Runnable {
    
    override fun run() {
        try {
            // 加载流程：磁盘缓存 -> 源加载 -> 转换 -> 缓存
            runWrapped()
        } catch (e: Exception) {
            onLoadFailed(e)
        }
    }
    
    private fun runWrapped() {
        when (runReason) {
            RunReason.INITIALIZE -> {
                stage = Stage.RESOURCE_CACHE
                getNextStage()
                runGenerators()
            }
            
            RunReason.SWITCH_TO_SOURCE_SERVICE -> {
                stage = Stage.SOURCE
                runGenerators()
            }
            
            RunReason.DECODE_DATA -> {
                decodeFromFetcher()
            }
        }
    }
    
    private fun runGenerators() {
        when (stage) {
            Stage.RESOURCE_CACHE -> {
                // 从资源缓存加载（磁盘上的转换后图片）
                val cacheFile = diskCache.get(resourceKey)
                if (cacheFile != null) {
                    onDataFetched(DataSource.RESOURCE_DISK_CACHE, cacheFile)
                    return
                }
            }
            
            Stage.DATA_CACHE -> {
                // 从数据缓存加载（原始图片）
                val cacheFile = diskCache.get(sourceKey)
                if (cacheFile != null) {
                    onDataFetched(DataSource.DATA_DISK_CACHE, cacheFile)
                    return
                }
            }
            
            Stage.SOURCE -> {
                // 从网络/其他源加载
                val fetcher = registry.getModelLoaders(model)[0]
                fetcher.loadData(priority, object : DataCallback {
                    override fun onDataReady(data: InputStream) {
                        onDataFetched(DataSource.REMOTE, data)
                    }
                })
            }
        }
    }
    
    private fun decodeFromFetcher() {
        val resource = decodeWithNotify(data, dataSource)
        
        // 保存到内存缓存
        memoryCache.put(resourceKey, resource)
        
        // 异步保存到磁盘缓存
        diskCacheExecutor.execute {
            diskCache.put(resourceKey, encodedResource)
        }
    }
}
```

**加载流程**：

1. **内存缓存** - 极快，直接返回
2. **资源缓存** - 磁盘中的转换后图片
3. **数据缓存** - 磁盘中的原始图片
4. **源加载** - 网络/其他源

#### Glide 的 LRU 缓存实现

```kotlin
// Glide 使用 LRU（Least Recently Used）缓存算法
class MemoryCacheAdapter : MemoryCache {
    
    private val lruCache = object : LinkedHashMap<Key, Resource>(
        100,  // 初始容量
        0.75f,  // 加载因子
        true  // 访问顺序（LRU）
    ) {
        override fun removeEldestEntry(eldest: MutableMap.MutableEntry<Key, Resource>?): Boolean {
            // 超过最大大小时移除最少使用的项
            return size() > maxSize
        }
    }
    
    override fun put(key: Key, resource: Resource) {
        lruCache[key] = resource
    }
    
    override fun get(key: Key): Resource? {
        // 访问时更新顺序（LRU）
        return lruCache[key]
    }
}

// LRU 工作流程（LinkedHashMap with accessOrder=true）
val cache = LinkedHashMap<String, Int>(16, 0.75f, true)
cache["a"] = 1
cache["b"] = 2
cache["c"] = 3

// 当访问 "a" 时
val value = cache["a"]
// LinkedHashMap 内部会将 "a" 移到链表末尾
// 顺序变为: b -> c -> a

// 当需要移除时，移除链表头部（最少使用的）
cache.iterator().next()  // 返回 "b"
```

---

## 8. WorkManager 后台任务

### 8.1 干什么的

WorkManager 是 Google 推荐的后台任务调度库，提供了一致的 API 来调度异步的、可延迟执行且预期要求即使应用退出也继续运行的工作。

### 8.2 怎么用的

#### 基础任务定义

```kotlin
// 定义后台任务
class DataSyncWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {
    
    override suspend fun doWork(): Result {
        return try {
            val userId = inputData.getString("userId") ?: return Result.failure()
            
            // 执行同步操作
            repository.syncUserData(userId)
            
            // 返回成功
            Result.success(
                workDataOf("syncTime" to System.currentTimeMillis())
            )
        } catch (e: Exception) {
            if (runAttemptCount < 3) {
                // 重试
                Result.retry()
            } else {
                // 最终失败
                Result.failure()
            }
        }
    }
}

// 调度任务
fun scheduleDataSync(userId: String) {
    val syncRequest = OneTimeWorkRequestBuilder<DataSyncWorker>()
        .setInputData(workDataOf("userId" to userId))
        .setBackoffCriteria(
            BackoffPolicy.EXPONENTIAL,
            OneTimeWorkRequest.MIN_BACKOFF_MILLIS,
            TimeUnit.MILLISECONDS
        )
        .build()
    
    WorkManager.getInstance(context).enqueueUniqueWork(
        "sync_data_$userId",
        ExistingWorkPolicy.KEEP,  // 如果已有相同名称的工作，保留
        syncRequest
    )
}
```

#### 定时任务

```kotlin
// 定时任务
class PeriodicSyncWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {
    
    override suspend fun doWork(): Result {
        return try {
            repository.performPeriodicSync()
            Result.success()
        } catch (e: Exception) {
            Result.retry()
        }
    }
}

// 调度定时任务
fun schedulePeriodicSync() {
    val syncRequest = PeriodicWorkRequestBuilder<PeriodicSyncWorker>(
        15,  // 周期
        TimeUnit.MINUTES,
        5,  // flexInterval
        TimeUnit.MINUTES
    ).build()
    
    WorkManager.getInstance(context).enqueueUniquePeriodicWork(
        "periodic_sync",
        ExistingPeriodicWorkPolicy.KEEP,
        syncRequest
    )
}
```

#### 链式任务与观察结果

```kotlin
// 链式任务
class CompressImageWorker(context: Context, params: WorkerParameters) :
    CoroutineWorker(context, params) {
    
    override suspend fun doWork(): Result {
        return try {
            val imageUri = inputData.getString("imageUri") ?: return Result.failure()
            val compressedPath = ImageCompressor.compress(imageUri)
            
            Result.success(
                workDataOf("compressedPath" to compressedPath)
            )
        } catch (e: Exception) {
            Result.failure()
        }
    }
}

class UploadImageWorker(context: Context, params: WorkerParameters) :
    CoroutineWorker(context, params) {
    
    override suspend fun doWork(): Result {
        return try {
            val compressedPath = inputData.getString("compressedPath")
                ?: return Result.failure()
            
            repository.uploadImage(compressedPath)
            
            Result.success()
        } catch (e: Exception) {
            Result.retry()
        }
    }
}

// 构建工作链
fun uploadImageWorkChain(imageUri: String) {
    val compressRequest = OneTimeWorkRequestBuilder<CompressImageWorker>()
        .setInputData(workDataOf("imageUri" to imageUri))
        .build()
    
    val uploadRequest = OneTimeWorkRequestBuilder<UploadImageWorker>()
        .build()
    
    // 链接任务：先压缩，后上传
    WorkManager.getInstance(context)
        .beginWith(compressRequest)
        .then(uploadRequest)
        .enqueue()
}

// 观察任务结果
fun observeUploadProgress(imageUri: String) {
    WorkManager.getInstance(context)
        .getWorkInfosForUniqueWorkLiveData("upload_$imageUri")
        .observe(lifecycleOwner) { workInfos ->
            for (workInfo in workInfos) {
                when (workInfo.state) {
                    WorkInfo.State.ENQUEUED -> Log.d("Work", "Enqueued")
                    WorkInfo.State.RUNNING -> Log.d("Work", "Running")
                    WorkInfo.State.SUCCEEDED -> {
                        val result = workInfo.outputData.getString("result")
                        Log.d("Work", "Succeeded: $result")
                    }
                    WorkInfo.State.FAILED -> Log.d("Work", "Failed")
                    WorkInfo.State.BLOCKED -> Log.d("Work", "Blocked")
                    WorkInfo.State.CANCELLED -> Log.d("Work", "Cancelled")
                }
            }
        }
}
```

#### 离线数据上传

```kotlin
// 离线数据定义
@Entity(tableName = "pending_uploads")
data class PendingUpload(
    @PrimaryKey(autoGenerate = true)
    val id: Long = 0,
    val filePath: String,
    val uploadTime: Long = System.currentTimeMillis(),
    val isUploaded: Boolean = false
)

// 离线上传 Worker
class OfflineUploadWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {
    
    private val uploadDao = AppDatabase.getInstance(context).uploadDao()
    private val apiService = RetrofitBuilder.apiService
    
    override suspend fun doWork(): Result {
        return withContext(Dispatchers.IO) {
            try {
                val pendingUploads = uploadDao.getPendingUploads()
                
                for (upload in pendingUploads) {
                    try {
                        val file = File(upload.filePath)
                        val multipart = MultipartBody.Part.createFormData(
                            "file",
                            file.name,
                            file.asRequestBody("application/octet-stream".toMediaType())
                        )
                        
                        val response = apiService.uploadFile(
                            RequestBody.create("text/plain".toMediaType(), "Offline upload"),
                            multipart
                        )
                        
                        if (response.code == 200) {
                            // 标记为已上传
                            uploadDao.markAsUploaded(upload.id)
                        }
                    } catch (e: Exception) {
                        // 单个文件上传失败，继续处理下一个
                        continue
                    }
                }
                
                Result.success()
            } catch (e: Exception) {
                Result.retry()
            }
        }
    }
}

// 定期检查并上传离线数据
fun scheduleOfflineUpload() {
    val uploadRequest = PeriodicWorkRequestBuilder<OfflineUploadWorker>(
        15,
        TimeUnit.MINUTES
    )
        .setConstraints(
            Constraints.Builder()
                .setRequiredNetworkType(NetworkType.CONNECTED)
                .build()
        )
        .build()
    
    WorkManager.getInstance(context).enqueueUniquePeriodicWork(
        "offline_upload",
        ExistingPeriodicWorkPolicy.KEEP,
        uploadRequest
    )
}
```

### 8.3 源码原理

#### WorkManager 的任务调度原理

```kotlin
// WorkManager 数据库架构
@Database(
    entities = [
        WorkSpec::class,
        WorkTag::class,
        SystemIdInfo::class,
        WorkProgress::class,
        PreferenceKey::class,
        WorkName::class
    ],
    version = 16
)
abstract class WorkDatabase : RoomDatabase() {
    abstract fun workSpecDao(): WorkSpecDao
    abstract fun workTagDao(): WorkTagDao
    abstract fun systemIdInfoDao(): SystemIdInfoDao
}

// WorkSpec 存储任务信息
@Entity(
    tableName = "workspec",
    indices = [Index("schedule_requested_at"), Index("period_start_time")]
)
data class WorkSpec(
    @PrimaryKey
    @ColumnInfo(name = "id")
    val id: String,
    
    @ColumnInfo(name = "state")
    val state: WorkInfo.State,
    
    @ColumnInfo(name = "worker_class_name")
    val workerClassName: String,
    
    @ColumnInfo(name = "input_data")
    val inputData: Data,
    
    @ColumnInfo(name = "output_data")
    val outputData: Data,
    
    @ColumnInfo(name = "initial_delay")
    val initialDelay: Long,
    
    @ColumnInfo(name = "backoff_policy")
    val backoffPolicy: BackoffPolicy,
    
    @ColumnInfo(name = "run_attempt_count")
    val runAttemptCount: Int,
    
    @ColumnInfo(name = "next_schedule_time_millis")
    val nextScheduleTimeMillis: Long,
    
    @ColumnInfo(name = "period_duration")
    val periodDuration: Long
)

// WorkManager 的执行流程
class WorkerWrapper(
    private val appContext: Context,
    private val workSpec: WorkSpec,
    private val database: WorkDatabase,
    private val configuration: Configuration
) : Runnable {
    
    override fun run() {
        try {
            // 1. 更新状态为 RUNNING
            updateWorkSpec { it.copy(state = WorkInfo.State.RUNNING) }
            
            // 2. 实例化 Worker
            val workerFactory = configuration.workerFactory
            val worker = workerFactory.createWorkerWithDefaultFallback(
                appContext,
                workSpec.workerClassName,
                WorkerParameters(
                    workSpec.id,
                    workSpec.inputData,
                    configuration.minimumLoggingLevel
                )
            )
            
            // 3. 执行 doWork()
            val result = worker.doWork()
            
            // 4. 处理结果
            when (result) {
                is Result.Success -> {
                    updateWorkSpec {
                        it.copy(
                            state = WorkInfo.State.SUCCEEDED,
                            outputData = result.outputData
                        )
                    }
                }
                
                is Result.Retry -> {
                    if (workSpec.runAttemptCount < workSpec.backoffPolicy.maxRetries) {
                        // 计算下次重试时间
                        val nextRetryTime = computeNextRetryTime(
                            workSpec.runAttemptCount,
                            workSpec.backoffPolicy
                        )
                        
                        updateWorkSpec {
                            it.copy(
                                state = WorkInfo.State.ENQUEUED,
                                nextScheduleTimeMillis = nextRetryTime,
                                runAttemptCount = it.runAttemptCount + 1
                            )
                        }
                    } else {
                        updateWorkSpec { it.copy(state = WorkInfo.State.FAILED) }
                    }
                }
                
                is Result.Failure -> {
                    updateWorkSpec { it.copy(state = WorkInfo.State.FAILED) }
                }
            }
        } catch (e: Exception) {
            updateWorkSpec { it.copy(state = WorkInfo.State.FAILED) }
        }
    }
    
    private fun computeNextRetryTime(
        runAttemptCount: Int,
        backoffPolicy: BackoffPolicy
    ): Long {
        val backoffMs = when (backoffPolicy.policy) {
            BackoffPolicy.LINEAR -> {
                (1000L * (runAttemptCount + 1))
            }
            
            BackoffPolicy.EXPONENTIAL -> {
                // 指数退避：1s, 2s, 4s, 8s...
                (1000L * Math.pow(2.0, runAttemptCount.toDouble())).toLong()
            }
        }
        
        return System.currentTimeMillis() + backoffMs.coerceAtMost(backoffPolicy.maxBackoffMs)
    }
}
```

**执行流程**：

1. WorkManager 将任务存储到数据库
2. 根据约束条件和延迟时间调度
3. 到达执行时间时，从线程池中取出 Worker 执行
4. 根据返回的 Result 更新数据库状态
5. 支持重试、链接等高级功能

#### WorkManager 的约束处理

```kotlin
// Constraints 定义执行条件
val constraints = Constraints.Builder()
    .setRequiredNetworkType(NetworkType.CONNECTED)  // 需要网络连接
    .setRequiresBatteryNotLow(true)  // 电池不低于
    .setRequiresCharging(false)  // 不需要充电
    .setRequiresDeviceIdle(false)  // 不需要设备空闲
    .addContentUriTrigger(uri, true)  // 监听 ContentUri 变化
    .build()

// ConstraintProxy 处理约束
class ConstraintProxy(
    private val constraints: Constraints
) : BroadcastReceiver() {
    
    override fun onReceive(context: Context, intent: Intent) {
        when (intent.action) {
            ConnectivityManager.CONNECTIVITY_ACTION -> {
                // 检查网络状态
                val connMgr = context.getSystemService(Context.CONNECTIVITY_SERVICE) as ConnectivityManager
                val isConnected = connMgr.activeNetworkInfo?.isConnectedOrConnecting ?: false
                
                if (constraints.requiredNetworkType != NetworkType.NONE && isConnected) {
                    // 约束满足，触发任务执行
                    scheduleNextExecution()
                }
            }
            
            Intent.ACTION_BATTERY_CHANGED -> {
                // 检查电池状态
                val level = intent.getIntExtra(BatteryManager.EXTRA_LEVEL, -1)
                val scale = intent.getIntExtra(BatteryManager.EXTRA_SCALE, 100)
                val batteryPct = (level * 100 / scale)
                
                if (constraints.requiresBatteryNotLow && batteryPct > 15) {
                    scheduleNextExecution()
                }
            }
        }
    }
}
```

---

## 9-16. 其他关键技能

由于篇幅限制，以下技能将采用精简说明格式：

### 9. Hilt / Dagger 依赖注入

**干什么的**：自动化依赖注入框架，提高代码解耦性和可测试性。

**怎么用的**：

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object RepositoryModule {
    
    @Singleton
    @Provides
    fun provideUserRepository(
        apiService: UserApiService,
        userDao: UserDao
    ): UserRepository {
        return UserRepositoryImpl(apiService, userDao)
    }
}

@HiltViewModel
class UserViewModel @Inject constructor(
    private val repository: UserRepository
) : ViewModel() {
    // ...
}

@AndroidEntryPoint
class UserActivity : AppCompatActivity() {
    private val viewModel: UserViewModel by viewModels()
}
```

**源码原理**：Hilt 是 Dagger 的上层封装，编译时通过 KAPT 生成依赖注入代码。使用 `@Inject` 标记依赖注入点，编译器生成工厂方法实现对象创建。

---

### 10-12. 本地存储方案

**Room**：类型安全的数据库，编译时生成 DAO 实现代码。 **DataStore**：协程友好的键值存储，替代 SharedPreferences。 **SharedPreferences**：轻量级键值存储，适合简单配置。

**源码原理**：Room 使用 KAPT 生成 DAO 和数据库类，DataStore 基于 Protobuf 序列化，SharedPreferences 基于 XML 文件存储。

---

### 13. Android 性能优化

**内存优化**：

- LeakCanary 检测内存泄漏
- 及时释放 Bitmap 和大对象
- 使用弱引用存储缓存

**启动速度优化**：

- 延迟初始化非关键服务
- 使用 App Startup 库管理初始化顺序
- 异步加载数据

**布局优化**：

- 使用 ConstraintLayout 减少嵌套
- 避免过度绘制
- 合理使用 `<merge>` 和 `<stub>` 标签

**ANR 与 OOM 分析**：

- ANR：主线程阻塞超过 5 秒
- OOM：堆内存溢出
- 使用 Android Profiler 分析

---

### 14. Crash 日志分析

**工具**：

- Firebase Crashlytics：自动捕获崩溃
- Bugly、友盟等第三方平台
- Logcat 和 adb logcat

**分析方法**：

- 查看 Stack Trace 定位异常点
- 关联用户操作流程
- 收集设备信息（系统版本、内存等）

---

### 15. Android SDK 核心组件

**Activity 启动模式**：

- Standard：标准栈模式，每次都创建新实例
- SingleTop：栈顶重用
- SingleTask：栈内单一实例
- SingleInstance：应用内单一实例

**事件分发**：

```
Activity.dispatchTouchEvent()
  -> ViewGroup.dispatchTouchEvent()
    -> ViewGroup.onInterceptTouchEvent()
      -> View.dispatchTouchEvent()
        -> View.onTouchEvent()
```

**生命周期**：

```
onCreate() -> onStart() -> onResume() -> onPause() -> onStop() -> onDestroy()
```

---

### 16. Gradle 构建与开发工具

**Gradle 配置**：

```kotlin
android {
    compileSdk = 33
    defaultConfig {
        applicationId = "com.example.app"
        minSdk = 24
        targetSdk = 33
    }
    
    buildTypes {
        release {
            isMinifyEnabled = true
            proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"))
        }
    }
    
    buildFeatures {
        viewBinding = true
        dataBinding = true
    }
}

dependencies {
    implementation("androidx.lifecycle:lifecycle-viewmodel-ktx:2.5.1")
}
```

**签名打包**：

```kotlin
signingConfigs {
    create("release") {
        storeFile = file("release.keystore")
        storePassword = "password"
        keyAlias = "release"
        keyPassword = "password"
    }
}

buildTypes {
    release {
        signingConfig = signingConfigs.getByName("release")
    }
}
```

---

## 总结

这份指南涵盖了 Android 开发的核心技能领域。每个技能都包含了实际使用案例和源码级别的原理分析。建议按照以下顺序深化学习：

1. **基础**：Kotlin 特性 + MVVM 架构 + Jetpack 组件
2. **进阶**：协程 + 网络请求 + 本地存储
3. **优化**：性能优化 + 内存管理 + 日志分析
4. **工程化**：依赖注入 + 模块化 + 构建系统

持续关注官方文档和最新最佳实践，保持技能更新。