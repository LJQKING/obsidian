
## Section 1: Java/OOP 与设计模式

### 1.1 面向对象设计基础

**Q1: 什么是 SOLID 原则？在 Android 开发中如何应用？**

A: SOLID 是五个面向对象设计原则的首字母缩写：

|原则|定义|Android 应用例子|
|---|---|---|
|**S** - 单一职责|一个类只负责一个职责|ViewModel 专注状态管理，不混入 View 逻辑|
|**O** - 开闭原则|对扩展开放，对修改关闭|通过接口定义 Listener，而非修改现有代码|
|**L** - 里氏替换|子类可替换父类|Drawable 的各种实现类可互相替换|
|**I** - 接口隔离|不依赖不需要的接口|拆分大接口为多个小接口|
|**D** - 依赖倒置|依赖抽象而非具体实现|Hilt/Dagger 注入接口，而非直接 new 对象|

**在 IVI 应用中的具体实例**：

```kotlin
// ❌ 违反单一职责 - 一个类干了太多事
class CarControlManager {
    fun updateSpeed() { }
    fun updateTemperature() { }
    fun updateGPS() { }
    fun logData() { }
    fun sendToCloud() { }
}

// ✅ 应用 SOLID - 职责清晰
interface SpeedController { fun updateSpeed(): Unit }
interface TemperatureController { fun updateTemperature(): Unit }
class CarControlMediator(
    private val speedCtrl: SpeedController,
    private val tempCtrl: TemperatureController
)
```

---

**Q2: 说说 Java 中的 equals() 和 hashCode() 的关系，为什么要重写？**

A:

- **contracts 关系**：
    
    - 若 `a.equals(b)` 返回 true，则 `a.hashCode() == b.hashCode()` 必须成立
    - 反过来不一定：hashCode 相同不代表对象相等（哈希冲突）
- **为什么要重写**：用于 HashMap/HashSet 中的查找、去重
    
- **标准实现**：
    

```java
public class User {
    private String name;
    private int age;
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof User)) return false;
        User user = (User) o;
        return age == user.age && Objects.equals(name, user.name);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(name, age);  // 同一套字段
    }
}
```

- **Android 中的常见错误**：

```kotlin
// ❌ 只重写 equals，未重写 hashCode
data class Location(val lat: Double, val lon: Double) {
    override fun equals(other: Any?): Boolean = ...
    // 遗漏了 hashCode()！
}

// ✅ Kotlin data class 自动生成
data class Location(val lat: Double, val lon: Double)
```

---

**Q3: 说说 Java 中的反射机制。性能如何？在 Android 中的应用？**

A:

- **什么是反射**：在运行时动态获取/修改类的信息（字段、方法、构造函数等）
    
- **性能对比**：
    
    - 直接调用：1ns
    - 反射调用：100-1000ns （慢 100-1000 倍）
    - 原因：需要检查权限、类型转换、方法查找等
- **Android 中的应用**：
    
    - **Hilt/Dagger DI 框架**：编译期生成代码，运行时反射注入依赖
    - **Gson/Retrofit**：序列化/反序列化对象
    - **ButterKnife**：注解绑定视图（已过时，Kotlin Synthetics/View Binding 是现代做法）
    - **数据库 ORM**：Room 在编译期处理，运行时避免反射
- **性能优化策略**：
    

```kotlin
// ❌ 每次调用都反射
fun getFieldByReflection(obj: Any, fieldName: String): Any? {
    return obj.javaClass.getDeclaredField(fieldName).get(obj)
}

// ✅ 缓存 Field 对象
private val fieldCache = mutableMapOf<String, Field>()
fun getFieldCached(obj: Any, fieldName: String): Any? {
    val field = fieldCache.getOrPut(fieldName) {
        obj.javaClass.getDeclaredField(fieldName).apply { isAccessible = true }
    }
    return field.get(obj)
}
```

---

### 1.2 设计模式

**Q4: 单例模式有哪些实现方式？在 Android 中的应用？**

A:

|方式|代码|优缺点|Android 应用|
|---|---|---|---|
|**饿汉式**|`static final INSTANCE = new Singleton()`|简单、线程安全；但类加载时就创建，占用内存|-|
|**懒汉式**（非线程安全）|`if(instance==null) instance = new ...`|延迟创建；多线程不安全|不推荐|
|**双重检查锁**|`if(instance==null) synchronized { if(instance==null) ...}`|线程安全、延迟创建；代码复杂|数据库、日志|
|**静态内部类**|`static class Holder { static final INSTANCE = new ...}`|**推荐**：简洁、线程安全、支持延迟加载|常用|
|**枚举**|`enum Singleton { INSTANCE; }`|最安全、防反射；但不支持继承|线程池、监听器|

**在 IVI 项目中的最佳实践**：

```kotlin
// ✅ Kotlin object 声明（线程安全、天然单例）
object CarStateManager {
    private val _speed = MutableLiveData<Int>()
    val speed: LiveData<Int> get() = _speed
    
    fun updateSpeed(value: Int) {
        _speed.value = value
    }
}

// 使用
CarStateManager.updateSpeed(100)
```

**为什么 Android 中要用单例**：

- 全局共享的资源：GPS、传感器、数据库连接
- 避免重复创建（内存紧张的移动设备）
- **风险**：内存泄漏（持有 Activity context），单元测试困难

---

**Q5: 工厂模式、建造者模式、装饰者模式的区别和应用场景？**

A:

|模式|目的|代码示例|Android 应用|
|---|---|---|---|
|**工厂模式**|创建对象，隐藏实现细节|`URLConnection conn = URL.openConnection()`|Intent、Fragment、View 创建|
|**建造者模式**|逐步构建复杂对象|`AlertDialog.Builder().setTitle(...).build()`|Notification、OkHttpClient、Room 数据库配置|
|**装饰者模式**|动态增强对象功能|`BufferedInputStream wraps FileInputStream`|InputStream 装饰、Java NIO|

**具体代码 - 建造者模式在 IVI 中的应用**：

```kotlin
// 智能座舱配置构建
data class CabinConfig(
    val brightnessLevel: Int,
    val temperatureMode: String,
    val musicVolume: Int,
    val gpsEnabled: Boolean
)

class CabinConfigBuilder {
    private var brightnessLevel = 50
    private var temperatureMode = "auto"
    private var musicVolume = 30
    private var gpsEnabled = true
    
    fun brightness(level: Int) = apply { this.brightnessLevel = level }
    fun temperature(mode: String) = apply { this.temperatureMode = mode }
    fun volume(vol: Int) = apply { this.musicVolume = vol }
    fun gps(enabled: Boolean) = apply { this.gpsEnabled = enabled }
    
    fun build() = CabinConfig(brightnessLevel, temperatureMode, musicVolume, gpsEnabled)
}

// 使用 - 链式调用，可读性强
val config = CabinConfigBuilder()
    .brightness(80)
    .temperature("cool")
    .volume(60)
    .build()
```

**装饰者模式 - 动态添加功能**：

```kotlin
interface CarSpeed {
    fun getSpeed(): Int
}

class BasicSpeed : CarSpeed {
    override fun getSpeed() = 100
}

// 装饰器 - 加入 GPS 校准
class GPSCalibratedSpeed(private val wrapped: CarSpeed) : CarSpeed {
    override fun getSpeed(): Int = (wrapped.getSpeed() * 0.95).toInt()  // GPS 通常比速度表慢 5%
}

// 装饰器 - 加入滑动平均（避免抖动）
class SmoothSpeed(private val wrapped: CarSpeed) : CarSpeed {
    private var lastSpeed = 0
    override fun getSpeed(): Int {
        val current = wrapped.getSpeed()
        lastSpeed = (lastSpeed + current) / 2
        return lastSpeed
    }
}

// 组合多个装饰器
val speed: CarSpeed = SmoothSpeed(GPSCalibratedSpeed(BasicSpeed()))
```

---

**Q6: 观察者模式与发布-订阅模式的区别？**

A:

|特性|观察者模式|发布-订阅模式|
|---|---|---|
|**耦合度**|Subject 与 Observer 直接关联|Publisher 与 Subscriber 通过 Broker 解耦|
|**中间件**|无，一对多直接通知|有消息队列/事件总线（Broker）|
|**同步性**|通常同步|可异步|
|**用途**|一个对象状态变化→通知多个观察者|事件驱动、异步通信|

**Android 中的实际应用**：

```kotlin
// ✅ 观察者模式 - LiveData/StateFlow
class UserViewModel : ViewModel() {
    private val _userName = MutableLiveData<String>()
    val userName: LiveData<String> get() = _userName
    
    fun setName(name: String) {
        _userName.value = name  // 直接通知所有观察者
    }
}

// ✅ 发布-订阅 - EventBus/Kotlin Flow
object CarEventBus {
    private val _events = MutableSharedFlow<CarEvent>()
    val events = _events.asSharedFlow()
    
    suspend fun publish(event: CarEvent) {
        _events.emit(event)
    }
}

// 订阅
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        CarEventBus.events.collect { event ->
            handleCarEvent(event)
        }
    }
}
```

---

## Section 2: Android 核心框架与组件

### 2.1 Lifecycle 与生命周期

**Q7: 说说 Activity、Fragment、Service 的生命周期。区别是什么？**

A:

**Activity 生命周期**（7 个回调）：

```
onCreate() → onStart() → onResume() → [运行中] → onPause() → onStop() → onDestroy()
      ↑                                                           ↓
      ←←←←←←←←←← 返回时 ←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←
```

|回调|何时调用|常见操作|
|---|---|---|
|`onCreate()`|Activity 创建|初始化 UI、恢复 savedInstanceState|
|`onStart()`|Activity 可见（但可能在后台）|启动动画|
|`onResume()`|Activity 获得焦点，用户交互|启动摄像头、GPS、获取焦点|
|`onPause()`|Activity 失焦（弹窗、返回等）|暂停音乐、释放摄像头|
|`onStop()`|Activity 不可见|保存数据、停止后台任务|
|`onDestroy()`|Activity 销毁|释放资源、取消注册|

**Fragment 生命周期**（11 个回调，包括 View 相关）：

```
onAttach() → onCreate() → onCreateView() → onViewCreated() → onStart() → onResume()
→ [运行中] → onPause() → onStop() → onDestroyView() → onDestroy() → onDetach()
```

**Service 生命周期**（仅 5 个回调）：

```
启动方式：startService()
onCreate() → onStartCommand() → [运行中] → onDestroy()

绑定方式：bindService()
onCreate() → onBind() → [运行中] → onUnbind() → onDestroy()
```

**三者的区别**：

- **Activity**：有 UI、响应用户交互、可见
- **Fragment**：轻量级组件、依附于 Activity、可复用
- **Service**：后台进程、无 UI、独立于用户交互

---

**Q8: Lifecycle 架构组件的核心机制是什么？如何实现的？**

A:

**核心思想**：将生命周期管理抽象为 `Lifecycle` 类，解耦业务逻辑与生命周期。

**关键组件**：

```kotlin
// 1. Lifecycle - 状态容器
class Lifecycle {
    enum class Event { ON_CREATE, ON_START, ON_RESUME, ON_PAUSE, ON_STOP, ON_DESTROY }
    enum class State { DESTROYED, INITIALIZED, CREATED, STARTED, RESUMED }
    
    fun addObserver(observer: LifecycleObserver) { }
    fun removeObserver(observer: LifecycleObserver) { }
}

// 2. LifecycleObserver - 观察者接口
interface LifecycleObserver {
    @OnLifecycleEvent(Lifecycle.Event.ON_RESUME)
    fun onResume() { }
}

// 3. LifecycleOwner - 生命周期所有者
interface LifecycleOwner {
    fun getLifecycle(): Lifecycle
}
```

**实现原理 - 基于注解的自动回调**：

```kotlin
// Android 内部实现（简化版）
class LifecycleRegistry(val owner: LifecycleOwner) {
    private val observers = mutableListOf<LifecycleObserver>()
    
    fun addObserver(observer: LifecycleObserver) {
        observers.add(observer)
        // 通过反射找到 @OnLifecycleEvent 注解的方法
        for (method in observer.javaClass.methods) {
            val annotation = method.getAnnotation(OnLifecycleEvent::class.java)
            if (annotation != null) {
                // 缓存方法，生命周期变化时调用
                cacheMethod(observer, method, annotation.value)
            }
        }
    }
    
    fun handleLifecycleEvent(event: Lifecycle.Event) {
        observers.forEach { observer ->
            // 调用对应的方法
            invokeMethod(observer, event)
        }
    }
}

// 在 Activity/Fragment 中（自动处理）
class ComponentActivity : AppCompatActivity(), LifecycleOwner {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        lifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_CREATE)
    }
    
    override fun onResume() {
        super.onResume()
        lifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_RESUME)
    }
}
```

**在 IVI 项目中的应用 - GPS 传感器管理**：

```kotlin
class GPSLocationObserver : LifecycleObserver {
    private lateinit var locationManager: LocationManager
    
    @OnLifecycleEvent(Lifecycle.Event.ON_RESUME)
    fun startGPS() {
        // 仅当 Activity/Fragment 可见时才启动 GPS
        locationManager.requestLocationUpdates(LocationManager.GPS_PROVIDER, 1000, 0f, listener)
    }
    
    @OnLifecycleEvent(Lifecycle.Event.ON_PAUSE)
    fun stopGPS() {
        // Activity 进入后台立即停止 GPS，节省电池
        locationManager.removeUpdates(listener)
    }
}

// 在 Activity 中使用
class MapActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        lifecycle.addObserver(GPSLocationObserver())
    }
}
```

**优势**：

- ✅ 自动跟随生命周期，无需手动管理
- ✅ 避免内存泄漏（Observer 可自动解除绑定）
- ✅ 代码清晰、职责明确

---

### 2.2 ViewModel 与状态管理

**Q9: ViewModel 的作用是什么？为什么 Activity 重建时 ViewModel 能保留数据？**

A:

**ViewModel 的作用**：

1. 管理 UI 相关的数据，生命周期独立于 Activity/Fragment
2. 在配置变化（屏幕旋转、语言切换等）时保留数据
3. 避免 Activity 持有过多业务逻辑，提升可测试性

**ViewModel 能保留数据的原理**：

```
                        Activity Destroyed
                                ↓
                    [Configuration Change]
                                ↓
                        Activity Recreated
                                ↓
                        ViewModel 幸存！
```

**具体机制**：

```kotlin
// 1. ViewModelStore - 存储 ViewModel 的容器
class ViewModelStore {
    private val map = mutableMapOf<String, ViewModel>()
    
    fun put(key: String, viewModel: ViewModel) {
        map[key] = viewModel
    }
    
    fun get(key: String): ViewModel? = map[key]
}

// 2. ViewModelProvider - 工厂模式创建 ViewModel
class ViewModelProvider(private val store: ViewModelStore) {
    inline fun <reified T : ViewModel> get(modelClass: Class<T>): T {
        val key = modelClass.canonicalName
        var viewModel = store.get(key) as? T
        
        if (viewModel == null) {
            // 首次创建，存入 Store
            viewModel = modelClass.newInstance()
            store.put(key, viewModel)
        }
        return viewModel
    }
}

// 3. ComponentActivity 保留 ViewModelStore
class ComponentActivity : AppCompatActivity() {
    private var viewModelStore: ViewModelStore? = null
    
    override fun getViewModelStore(): ViewModelStore {
        if (viewModelStore == null) {
            // 关键：retain instance fragment 技巧
            val retainFragment = RetainedFragment()
            supportFragmentManager.beginTransaction()
                .add(retainFragment, TAG)
                .commitNow()
            viewModelStore = retainFragment.viewModelStore
        }
        return viewModelStore
    }
}

// 4. RetainedFragment - 配置变化时保留
class RetainedFragment : Fragment() {
    val viewModelStore = ViewModelStore()
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        retainInstance = true  // ← 关键：保留 Fragment 实例
    }
}
```

**实际应用 - IVI 导航页面数据保留**：

```kotlin
class NavigationViewModel : ViewModel() {
    private val _currentLocation = MutableLiveData<Location>()
    val currentLocation: LiveData<Location> get() = _currentLocation
    
    private val _routeList = MutableLiveData<List<RoutePoint>>()
    val routeList: LiveData<List<RoutePoint>> get() = _routeList
    
    init {
        loadInitialRoute()
    }
    
    private fun loadInitialRoute() {
        // 数据加载只执行一次
        viewModelScope.launch {
            _routeList.value = repository.getRoute()
        }
    }
}

class NavigationFragment : Fragment() {
    private val viewModel: NavigationViewModel by viewModels()
    
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        // 屏幕旋转后，routeList 的数据保留，无需重新加载
        viewModel.routeList.observe(viewLifecycleOwner) { routes ->
            updateMap(routes)
        }
    }
}
```

**ViewModel 的生命周期图**：

```
Activity 创建 1 → ViewModel 创建 → Activity 销毁（旋转）
                    ↓
           ViewModel 保留在内存
                    ↓
Activity 创建 2（新实例）→ 找到旧 ViewModel → Activity 销毁（真正关闭）
                                                    ↓
                                           ViewModel.onCleared() 调用，释放资源
```

---

**Q10: 解释 MutableLiveData、LiveData、StateFlow、SharedFlow 的区别？**

A:

|特性|LiveData|MutableLiveData|StateFlow|SharedFlow|
|---|---|---|---|---|
|**可变性**|只读|可读写|可读写|可读写|
|**初始值**|可无|需传入|必须有|可无|
|**Scope**|主线程|主线程|任意线程|任意线程|
|**重放**|仅最后一个值|仅最后一个值|最后一个值|可配置缓冲区|
|**背压**|不支持|不支持|不支持|支持背压处理|
|**推荐用途**|UI 观察（Lifecycle 感知）|ViewModel 状态管理|状态管理（Kotlin 协程）|事件流、多播|

**代码对比**：

```kotlin
// ❌ LiveData - 过时做法
class OldUserViewModel : ViewModel() {
    private val _userName = MutableLiveData<String>()
    val userName: LiveData<String> get() = _userName
    
    // 必须在主线程调用
    fun setName(name: String) {
        _userName.value = name
    }
}

// ✅ StateFlow - 现代做法（Kotlin 协程优先）
class UserViewModel : ViewModel() {
    private val _userName = MutableStateFlow("初始值")
    val userName: StateFlow<String> get() = _userName.asStateFlow()
    
    fun setName(name: String) {
        _userName.value = name  // 自动处理线程安全
    }
}

// ✅ SharedFlow - 多播事件
class CarEventViewModel : ViewModel() {
    private val _events = MutableSharedFlow<CarEvent>(replay = 1)
    val events: SharedFlow<CarEvent> get() = _events.asSharedFlow()
    
    fun publishEvent(event: CarEvent) {
        viewModelScope.launch {
            _events.emit(event)  // 支持异步发送
        }
    }
}

// 观察方式对比
class MapFragment : Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        // LiveData - Lifecycle 感知
        viewModel.userName.observe(viewLifecycleOwner) { name ->
            updateUI(name)
        }
        
        // StateFlow - 手动管理生命周期
        viewLifecycleOwner.lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.userName.collect { name ->
                    updateUI(name)
                }
            }
        }
    }
}
```

**在 IVI 仪表盘中的应用**：

```kotlin
class DashboardViewModel : ViewModel() {
    // 仪表盘速度显示
    private val _speed = MutableStateFlow(0)
    val speed: StateFlow<Int> get() = _speed.asStateFlow()
    
    // 仪表盘告警事件（一次性消费）
    private val _alarms = MutableSharedFlow<DashboardAlarm>(replay = 0)
    val alarms: SharedFlow<DashboardAlarm> get() = _alarms.asSharedFlow()
    
    init {
        // 监听速度变化
        viewModelScope.launch {
            sensorRepository.speedUpdates.collect { speed ->
                _speed.value = speed
            }
        }
        
        // 发布告警事件
        viewModelScope.launch {
            sensorRepository.alarmEvents.collect { alarm ->
                _alarms.emit(alarm)
            }
        }
    }
}

class DashboardFragment : Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        // 速度更新 - 永不丢失最新值
        viewLifecycleOwner.lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.speed.collect { speed ->
                    speedGauge.setSpeed(speed)
                }
            }
        }
        
        // 告警事件 - 每次都消费
        viewLifecycleOwner.lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.alarms.collect { alarm ->
                    showAlarmNotification(alarm)
                }
            }
        }
    }
}
```

---

### 2.3 Hilt 与 Dagger2 依赖注入

**Q11: Hilt 的工作原理？相比 Dagger2 的优势是什么？**

A:

**Hilt 是什么**：Google 基于 Dagger2 开发的 Android 专用依赖注入框架，简化 Dagger2 的复杂配置。

**Hilt 的核心原理**：

```
1. 编译期：注解处理器扫描 @Module、@Inject、@HiltViewModel 等注解
                    ↓
2. 代码生成：生成 Dagger Component（Hilt_XXXActivity 等）
                    ↓
3. 运行期：通过反射生成的代码注入依赖
```

**核心架构**：

```kotlin
// 1. @HiltAndroidApp - 应用入口，必须标注
@HiltAndroidApp
class MyApplication : Application() {
    // Hilt 会自动初始化
}

// 2. @Module + @Provides - 定义如何创建对象
@Module
@InstallIn(SingletonComponent::class)  // 指定生命周期
object DatabaseModule {
    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: Context): AppDatabase {
        return Room.databaseBuilder(context, AppDatabase::class.java, "app.db").build()
    }
    
    @Provides
    fun provideUserDao(database: AppDatabase): UserDao = database.userDao()
}

// 3. @Inject - 注入字段或构造函数
class UserRepository @Inject constructor(
    private val userDao: UserDao,
    private val apiService: UserApiService
) {
    // 依赖自动注入
}

// 4. @HiltViewModel - ViewModel 注入（自动作用域管理）
@HiltViewModel
class UserViewModel @Inject constructor(
    private val repository: UserRepository
) : ViewModel() {
    // ...
}

// 5. Activity/Fragment 注入
@AndroidEntryPoint
class UserListActivity : AppCompatActivity() {
    private val viewModel: UserViewModel by viewModels()
    
    @Inject
    lateinit var analytics: AnalyticsService
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // 依赖已自动注入
    }
}
```

**Hilt 与 Dagger2 的对比**：

|特性|Dagger2|Hilt|
|---|---|---|
|**配置复杂度**|需手动定义 Component、Scope|自动生成，开箱即用|
|**生命周期管理**|手动管理|自动与 Activity/Fragment 绑定|
|**学习曲线**|陡峭|平缓|
|**性能**|原生性能|稍慢（多一层抽象）|
|**适用**|通用 Java 项目|Android 专用|

**Hilt 的生命周期组件**：

```kotlin
// 不同的 @InstallIn 决定对象生命周期
@InstallIn(SingletonComponent::class)      // 应用级单例
@InstallIn(ActivityComponent::class)       // Activity 级作用域
@InstallIn(FragmentComponent::class)       // Fragment 级作用域
@InstallIn(ViewComponent::class)           // View 级作用域（API 31+）
```

**在 IVI 项目中的应用**：

```kotlin
// 定义模块
@Module
@InstallIn(SingletonComponent::class)
object IVIModule {
    @Provides
    @Singleton
    fun provideLocationManager(@ApplicationContext context: Context): LocationManager {
        return context.getSystemService(Context.LOCATION_SERVICE) as LocationManager
    }
    
    @Provides
    @Singleton
    fun provideCarSensorService(@ApplicationContext context: Context): CarSensorService {
        return CarSensorService(context)
    }
}

// ViewModel 依赖注入
@HiltViewModel
class CabinControlViewModel @Inject constructor(
    private val locationManager: LocationManager,
    private val sensorService: CarSensorService,
    private val repository: CarStateRepository
) : ViewModel() {
    
    fun updateCabinState() {
        // 所有依赖已自动注入
    }
}

// Activity 依赖注入
@AndroidEntryPoint
class CabinControlActivity : AppCompatActivity() {
    private val viewModel: CabinControlViewModel by viewModels()
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // viewModel 已注入，可直接使用
    }
}
```

**生成代码示例**（Hilt 编译期生成）：

```java
// Hilt 自动生成（开发者看不到）
public final class Hilt_CabinControlActivity extends AppCompatActivity {
    private IVIModule module;
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        // 自动创建模块实例
        module = new IVIModule();
        
        // 注入依赖
        LocationManager locationManager = module.provideLocationManager(this);
        CarSensorService sensorService = module.provideCarSensorService(this);
        
        super.onCreate(savedInstanceState);
    }
}
```

---

## Section 3: 并发、多线程与性能优化

### 3.1 线程池与异步

**Q12: ThreadPoolExecutor 的核心参数是什么？如何选择？**

A:

**7 大核心参数**：

```java
public ThreadPoolExecutor(
    int corePoolSize,           // 1. 核心线程数
    int maximumPoolSize,        // 2. 最大线程数
    long keepAliveTime,         // 3. 线程存活时间
    TimeUnit unit,              // 4. 时间单位
    BlockingQueue<Runnable> workQueue,  // 5. 任务队列
    ThreadFactory threadFactory,        // 6. 线程工厂
    RejectedExecutionHandler handler    // 7. 拒绝策略
)
```

|参数|含义|选择建议|
|---|---|---|
|**corePoolSize**|始终保留的线程数|CPU 密集：CPU 核心数；I/O 密集：2×CPU 核心数|
|**maximumPoolSize**|最多创建的线程数|≥ corePoolSize，通常 2-4 倍 corePoolSize|
|**keepAliveTime + unit**|非核心线程闲置多久后销毁|60-300 秒，根据场景调整|
|**workQueue**|任务队列|LinkedBlockingQueue（无界）/ ArrayBlockingQueue（有界）/ SynchronousQueue（同步）|
|**threadFactory**|创建线程的工厂|自定义可设置线程名称便于调试|
|**handler**|队列满+线程满时的拒绝策略|AbortPolicy（抛异常）/ CallerRunsPolicy（主线程执行）/ DiscardPolicy（丢弃）|

**执行流程**：

```
提交任务
  ↓
是否 < corePoolSize？→ 是 → 创建核心线程执行
  ↓ 否
队列是否未满？→ 是 → 放入队列等待
  ↓ 否
是否 < maximumPoolSize？→ 是 → 创建非核心线程执行
  ↓ 否
执行拒绝策略
```

**在 IVI 数据采集中的应用**：

```kotlin
class SensorDataCollector {
    // 配置线程池 - 传感器数据采集是 I/O 密集
    private val sensorExecutor = ThreadPoolExecutor(
        corePoolSize = 4,           // 4 个传感器并发读取
        maximumPoolSize = 8,        // 最多 8 个线程
        keepAliveTime = 60,
        unit = TimeUnit.SECONDS,
        workQueue = LinkedBlockingQueue(100),  // 缓冲队列
        threadFactory = ThreadFactory { Thread(it).apply { name = "SensorThread-${id}" } },
        handler = ThreadPoolExecutor.CallerRunsPolicy()  // 队列满时主线程处理
    )
    
    fun collectSensorData() {
        repeat(20) { index ->
            sensorExecutor.submit {
                val data = readSensor(index)
                processSensorData(data)
            }
        }
    }
    
    fun shutdown() {
        sensorExecutor.shutdown()
    }
}
```

**常见线程池预设**：

```kotlin
// 不推荐！Executors 工厂方法隐藏了参数配置
val fixedPool = Executors.newFixedThreadPool(4)      // 可能 OOM（队列无界）
val singlePool = Executors.newSingleThreadExecutor()
val cachedPool = Executors.newCachedThreadPool()     // 线程无限制增长

// ✅ 推荐：显式创建 ThreadPoolExecutor
val pool = ThreadPoolExecutor(
    4, 8, 60, TimeUnit.SECONDS,
    LinkedBlockingQueue(100),
    threadFactory = ThreadFactory { Thread(it).apply { isDaemon = false } }
)
```

---

**Q13: Kotlin 协程与 Java 线程池的区别？为什么用协程？**

A:

| 特性       | Java 线程池          | Kotlin 协程       |
| -------- | ----------------- | --------------- |
| **成本**   | 创建 1 个线程 ≈ 1MB 内存 | 轻量级，可创建百万级协程    |
| **切换开销** | 高（线程上下文切换）        | 低（用户空间调度）       |
| **编程模型** | 回调/Future（复杂）     | 顺序代码（简洁）        |
| **异常处理** | try-catch（外层捕获困难） | try-catch（自然处理） |
| **背压处理** | 困难                | 通过 suspend 函数简单 |

**对比代码**：

```kotlin
// ❌ 线程池方式 - 回调地狱
executorService.submit {
    try {
        val userData = loadUserFromNetwork()
        mainHandler.post {
            val posts = loadPostsFromDb(userData.id)
            mainHandler.post {
                updateUI(userData, posts)
            }
        }
    } catch (e: Exception) {
        mainHandler.post {
            showError(e)
        }
    }
}

// ✅ 协程方式 - 顺序写法
viewModelScope.launch {
    try {
        val userData = loadUserFromNetwork()  // 挂起函数
        val posts = loadPostsFromDb(userData.id)  // 在 Main 线程安全执行
        updateUI(userData, posts)
    } catch (e: Exception) {
        showError(e)
    }
}
```

**协程的 Dispatcher（类似线程池的角色）**：

```kotlin
// 不同的 Dispatcher 调度协程到不同线程
viewModelScope.launch(Dispatchers.Main) {
    // UI 操作 - Main 线程
    updateUI()
}

viewModelScope.launch(Dispatchers.IO) {
    // 网络/数据库 - IO 线程池（64 个线程）
    val data = loadDataFromNetwork()
}

viewModelScope.launch(Dispatchers.Default) {
    // CPU 密集 - Default 线程池（CPU 核心数）
    val result = heavyComputation()
}

viewModelScope.launch(Dispatchers.Unconfined) {
    // 不指定线程（不推荐）
}
```

**在 IVI 项目中的应用**：

```kotlin
@HiltViewModel
class MapViewModel @Inject constructor(
    private val locationRepo: LocationRepository,
    private val routeRepo: RouteRepository
) : ViewModel() {
    
    private val _mapState = MutableStateFlow<MapState>(MapState.Loading)
    val mapState: StateFlow<MapState> get() = _mapState.asStateFlow()
    
    fun loadMap(location: Location) {
        viewModelScope.launch {
            try {
                _mapState.value = MapState.Loading
                
                // 并发加载位置和路线
                val (userLocation, nearbyRoutes) = awaitAll(
                    async(Dispatchers.IO) { locationRepo.getUserLocation() },
                    async(Dispatchers.IO) { routeRepo.getNearbyRoutes(location) }
                )
                
                _mapState.value = MapState.Success(userLocation, nearbyRoutes)
            } catch (e: Exception) {
                _mapState.value = MapState.Error(e.message ?: "Unknown error")
            }
        }
    }
}
```

**为什么协程优于线程池**：

1. **内存效率**：1 个线程 1MB vs 1 个协程 0.1KB
2. **代码可读性**：顺序写法 vs 回调地狱
3. **异常处理**：try-catch 自然处理 vs 回调中异常难捕获
4. **取消机制**：job.cancel() 自动清理 vs 手动 shutdown

---

### 3.2 内存优化

**Q14: Android 内存管理机制是什么？常见内存泄漏场景？**

A:

**Android 内存结构**：

```
[Java Heap] ← GC 管理
  - Objects
  - Strings
  - Arrays

[Native Heap] ← C/C++ 手动管理
  - NDK 代码

[Graphics Memory]
  - 纹理、缓存
```

**Java 堆内存管理 - 分代 GC**：

```
Younger Generation (频繁回收)          Older Generation (少量回收)
┌─────────────────────────┐          ┌─────────────────────┐
│ Eden  │ Survivor0│ Sur1  │ ──────→ │     Old Gen         │
│ (活跃)│ (中等)  │ (少活)│          │ (长期存活)          │
└─────────────────────────┘          └─────────────────────┘
```

**GC 过程**：

1. **Minor GC**（Young Gen）：回收 Eden + Survivor，频繁发生
2. **Major GC/Full GC**（Old Gen）：回收整个堆，停止应用（Stop-the-world）
3. **触发条件**：内存不足、System.gc() 调用、达到阈值

**常见内存泄漏场景**：

|场景|原因|解决方案|
|---|---|---|
|**静态集合持有 Context**|静态变量生命周期长于 Activity|改为实例变量或使用 WeakReference|
|**内部类持有外部类引用**|非静态内部类隐含持有 `this$0`|改为静态内部类或 Lambda|
|**监听器注册未注销**|Activity 销毁时未移除监听器|在 onDestroy() 中 removeListener()|
|**Timer 未取消**|Timer 持有线程|在 onDestroy() 中 cancel()|
|**WebView**|WebView 持有 Context 导致泄漏|单独进程或封装清理逻辑|
|**Handler + Looper**|消息队列未清空|使用 WeakReference 或及时 removeMessages()|

**代码示例**：

```kotlin
// ❌ 内存泄漏 1：静态集合持有 Activity
class LeakyActivity : AppCompatActivity() {
    companion object {
        private val activities = mutableListOf<Activity>()  // 危险！
    }
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        activities.add(this)  // Activity 永不释放
    }
}

// ✅ 修复方案 1：使用 WeakReference
companion object {
    private val activities = mutableListOf<WeakReference<Activity>>()
}

override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    activities.add(WeakReference(this))  // 弱引用，允许 GC 回收
}

// ❌ 内存泄漏 2：非静态内部类持有外部类引用
class MapActivity : AppCompatActivity() {
    inner class LocationListener : OnLocationChangedListener {  // 隐含持有 MapActivity
        override fun onLocationChanged(location: Location) {
            updateMap(location)
        }
    }
}

// ✅ 修复方案 2：改为静态内部类 + 弱引用
private class LocationListener(activity: MapActivity) : OnLocationChangedListener {
    private val activityRef = WeakReference(activity)
    
    override fun onLocationChanged(location: Location) {
        activityRef.get()?.updateMap(location)
    }
}

// ❌ 内存泄漏 3：注册监听器未注销
class SensorActivity : AppCompatActivity() {
    private val locationListener = LocationListener()
    
    override fun onResume() {
        super.onResume()
        locationManager.requestLocationUpdates(..., locationListener)
    }
    
    // 遗漏 onDestroy()
}

// ✅ 修复方案 3：
override fun onDestroy() {
    super.onDestroy()
    locationManager.removeUpdates(locationListener)  // 必须手动移除
}

// ❌ 内存泄漏 4：Handler 消息队列未清空
class MapActivity : AppCompatActivity() {
    private val handler = Handler(Looper.getMainLooper())
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        handler.postDelayed({ updateMap() }, 5000)
    }
    
    // Activity 销毁前消息执行，持有 Activity 引用
}

// ✅ 修复方案 4：
override fun onDestroy() {
    super.onDestroy()
    handler.removeCallbacksAndMessages(null)  // 清空所有消息
}

// ✅ 最佳实践：使用 lifecycleScope（自动管理）
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    lifecycleScope.launch {
        delay(5000)
        updateMap()  // Activity 销毁时自动取消
    }
}
```

**内存分析工具**：

- **Android Studio Memory Profiler**：实时内存监控
- **LeakCanary**：自动检测内存泄漏
- **Heap Dump**：分析对象引用链

---

**Q15: OOM（内存溢出）异常有哪些类型？如何避免？**

A:

**OOM 的主要类型**：

|类型|原因|症状|避免方案|
|---|---|---|---|
|**Java Heap OOM**|对象创建过多|应用崩溃、日志有 OutOfMemoryError|GC root 优化、内存池模式|
|**Native OOM**|JNI/NDK 分配失败|崩溃（C++ 异常）|限制 native 内存使用|
|**Bitmap OOM**|大图片加载|常见于图片库|图片压缩、LRU 缓存、分片加载|
|**String OOM**|字符串拼接|循环中大量字符串创建|使用 StringBuilder|

**Bitmap OOM 的常见场景**：

```kotlin
// ❌ 一次性加载高分辨率图片
val bitmap = BitmapFactory.decodeFile("/sdcard/4K_image.jpg")  // 4K = 4000×3000×4bytes = 48MB

// ✅ 加载前计算所需尺寸并缩放
class BitmapOptimizer {
    fun loadBitmap(path: String, targetWidth: Int, targetHeight: Int): Bitmap? {
        // 1. 读取图片原始尺寸（不加载像素数据）
        val options = BitmapFactory.Options().apply {
            inJustDecodeBounds = true
        }
        BitmapFactory.decodeFile(path, options)
        
        // 2. 计算缩放比例
        val scale = calculateInSampleSize(options, targetWidth, targetHeight)
        
        // 3. 加载缩放后的图片
        options.inJustDecodeBounds = false
        options.inSampleSize = scale
        return BitmapFactory.decodeFile(path, options)
    }
    
    private fun calculateInSampleSize(options: BitmapFactory.Options, reqWidth: Int, reqHeight: Int): Int {
        val (height, width) = options.run { outHeight to outWidth }
        var inSampleSize = 1
        
        if (height > reqHeight || width > reqWidth) {
            val halfHeight = height / 2
            val halfWidth = width / 2
            
            while (halfHeight / inSampleSize >= reqHeight && halfWidth / inSampleSize >= reqWidth) {
                inSampleSize *= 2
            }
        }
        return inSampleSize
    }
}

// ✅ 使用图片库（Glide）自动处理
Glide.with(context)
    .load(imageUrl)
    .override(targetWidth, targetHeight)  // 指定最终尺寸
    .into(imageView)
```

**字符串拼接 OOM**：

```kotlin
// ❌ 低效 - 每次迭代创建新字符串对象
var result = ""
repeat(10000) { i ->
    result += "Line $i\n"  // 创建 10000 个临时 String 对象
}

// ✅ 使用 StringBuilder
val result = StringBuilder()
repeat(10000) { i ->
    result.append("Line $i\n")  // 单个缓冲区
}
return result.toString()
```

**内存对象池模式（避免频繁分配/回收）**：

```kotlin
// 传感器数据采集 - 频繁创建 SensorEvent 对象
class SensorEventPool {
    private val pool = Stack<SensorEvent>()
    
    fun acquire(): SensorEvent {
        return if (pool.isNotEmpty()) {
            pool.pop().reset()
        } else {
            SensorEvent()
        }
    }
    
    fun release(event: SensorEvent) {
        event.clear()
        pool.push(event)
    }
}

class SensorCollector(private val pool: SensorEventPool) {
    fun onSensorEvent(rawData: ByteArray) {
        val event = pool.acquire()
        try {
            event.parseFromBytes(rawData)
            processSensorEvent(event)
        } finally {
            pool.release(event)
        }
    }
}
```

---

### 3.3 性能优化

**Q16: 如何排查和优化应用的卡顿问题？**

A:

**卡顿的根本原因**：

```
帧率要求：60 FPS（Android） = 16.67 ms / 帧
             120 FPS（高刷屏）= 8.33 ms / 帧

若某一帧超过 16.67ms → 掉帧 → 用户感受到卡顿
```

**卡顿的常见原因**：

1. **主线程做了耗时操作**（网络请求、数据库查询、复杂计算）
2. **布局过度复杂**（嵌套层级深、过多视图）
3. **频繁 GC**（内存抖动）
4. **过度绘制**（多层透明背景）

**排查工具与方法**：

|工具|用途|
|---|---|
|**Systrace**|函数调用耗时、帧率分析|
|**Perfetto**|现代系统性能分析|
|**Android Profiler**|内存、CPU、帧率实时监控|
|**Choreographer**|帧率监控|
|**StrictMode**|检测主线程 IO/网络|

**代码优化示例**：

```kotlin
// ❌ 问题 1：主线程做耗时操作
class UserListFragment : Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        // 主线程查询数据库 - 50ms 耗时
        val users = database.userDao().getAllUsers()  // 阻塞主线程
        adapter.setData(users)
    }
}

// ✅ 优化 1：使用协程切换线程
override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
    super.onViewCreated(view, savedInstanceState)
    
    viewLifecycleOwner.lifecycleScope.launch {
        // 在 IO 线程执行数据库查询
        val users = withContext(Dispatchers.IO) {
            database.userDao().getAllUsers()
        }
        // 自动切回 Main 线程更新 UI
        adapter.setData(users)
    }
}

// ❌ 问题 2：布局层级过深
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout> <!-- 第 1 层 -->
    <LinearLayout> <!-- 第 2 层 -->
        <LinearLayout> <!-- 第 3 层 -->
            <LinearLayout> <!-- 第 4 层 -->
                <TextView /> <!-- 嵌套过深！ -->
            </LinearLayout>
        </LinearLayout>
    </LinearLayout>
</LinearLayout>

// ✅ 优化 2：扁平化布局 - 使用 ConstraintLayout
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto">
    
    <TextView
        android:id="@+id/textView"
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintTop_toTopOf="parent" />
        
</androidx.constraintlayout.widget.ConstraintLayout>

// ❌ 问题 3：列表项复杂布局导致滑动卡顿
class UserAdapter : RecyclerView.Adapter<UserViewHolder>() {
    override fun onBindViewHolder(holder: UserViewHolder, position: Int) {
        val user = users[position]
        
        // 每次绑定都加载头像 - I/O 阻塞
        holder.avatar.setImageBitmap(loadBitmapFromDisk(user.avatarPath))
        
        // 复杂的文本布局重排
        holder.nameView.text = formatUserName(user)
        holder.statusView.text = fetchUserStatus(user)  // 网络请求！
    }
}

// ✅ 优化 3：
override fun onBindViewHolder(holder: UserViewHolder, position: Int) {
    val user = users[position]
    
    // 使用图片库 + 缓存
    Glide.with(context)
        .load(user.avatarUrl)
        .override(48.dp, 48.dp)
        .into(holder.avatar)
    
    // 预加载数据，不在 bind 时请求
    holder.nameView.text = user.name
    holder.statusView.text = user.cachedStatus  // 使用缓存数据
}

// ❌ 问题 4：频繁创建对象导致 GC 抖动
class MapRenderer {
    override fun onDrawFrame() {
        repeat(1000) { // 每帧创建 1000 个对象
            val point = Point(x, y)  // 短生命周期对象
            drawPoint(point)
        }
    }
}

// ✅ 优化 4：对象复用
class MapRenderer {
    private val pointPool = ObjectPool(Point::class.java)
    
    override fun onDrawFrame() {
        repeat(1000) {
            val point = pointPool.acquire()
            point.set(x, y)
            drawPoint(point)
            pointPool.release(point)
        }
    }
}
```

**在 IVI 仪表盘中的具体应用**：

```kotlin
// 关键：40ms 内完成所有UI更新（100ms 的 1/2.5）
@HiltViewModel
class DashboardViewModel @Inject constructor(
    private val sensorService: CarSensorService
) : ViewModel() {
    
    private val _dashboardState = MutableStateFlow<DashboardState>(DashboardState.Initial)
    
    init {
        viewModelScope.launch {
            // 在 Default Dispatcher（CPU 密集）处理数据
            sensorService.sensorUpdates
                .throttleTime(16, TimeUnit.MILLISECONDS)  // 限制更新频率，避免过度绘制
                .map { sensorData ->
                    // 预处理数据
                    withContext(Dispatchers.Default) {
                        processSensorData(sensorData)
                    }
                }
                .collect { processedData ->
                    // 切回 Main 线程更新 UI
                    _dashboardState.value = DashboardState.Ready(processedData)
                }
        }
    }
}
```

---

## Section 4: UI 框架与屏幕适配

### 4.1 屏幕适配

**Q17: Android 屏幕适配的原则是什么？如何适配不同屏幕尺寸？**

A:

**屏幕适配的核心概念**：

```
屏幕尺寸（对角线） ≠ 分辨率 ≠ 密度

例如：
- 5.5 英寸 @ 1080×1920 → density 401 dpi
- 6.5 英寸 @ 1080×1920 → density 269 dpi
```

**适配的 3 个维度**：

|维度|指标|适配方式|
|---|---|---|
|**1. 屏幕尺寸**|2.7"-6.5"+|使用 dp（设备无关像素）而非 px|
|**2. 屏幕密度**|ldpi / mdpi / hdpi / xhdpi / xxhdpi|提供多倍图资源（1x / 1.5x / 2x / 3x）|
|**3. 屏幕宽高比**|16:9 / 18:9 / 20:9 / 刘海屏|响应式布局 + Fragment 分栏|

**适配方案对比**：

|方案|实现方式|优缺点|
|---|---|---|
|**dp 单位**|所有尺寸用 dp 而非 px|✅ 简单，官方推荐；❌ 无法精确适配|
|**百分比布局**|父容器百分比|✅ 真正响应式；❌ API 28+ 才有 PercentFrameLayout|
|**ConstraintLayout**|约束+比例|✅ 最灵活，官方推荐|
|**屏幕限定符**|res/layout-sw600dp / land / port|✅ 大屏精准适配；❌ 需维护多份布局|
|**分栏适配**|平板使用 Fragment 分栏|✅ 大屏体验好；❌ 逻辑复杂|

**实现代码**：

```kotlin
// 方案 1：dp 单位（基础）
<LinearLayout
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:paddingHorizontal="16dp"  // 16dp 在 xxhdpi 上 = 48px，在 mdpi = 16px
    android:paddingVertical="12dp">
    
    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:textSize="16sp" />  // sp = scaled pixels，会跟随用户字体大小设置
</LinearLayout>

// 方案 2：ConstraintLayout 响应式（推荐）
<androidx.constraintlayout.widget.ConstraintLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent">
    
    <!-- 宽度为父容器 80%，高度按 1:1 比例 -->
    <ImageView
        android:id="@+id/mapImage"
        android:layout_width="0dp"
        android:layout_height="0dp"
        android:scaleType="centerCrop"
        app:layout_constraintWidth_percent="0.8"
        app:layout_constraintDimensionRatio="1:1"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintTop_toTopOf="parent" />
    
</androidx.constraintlayout.widget.ConstraintLayout>

// 方案 3：屏幕限定符（大屏平板专用布局）
// res/layout/activity_map.xml（手机默认布局）
// res/layout-sw600dp/activity_map.xml（平板布局 - 600dp 宽以上）
// res/layout-land/activity_map.xml（横屏布局）

// 平板主从分栏
// res/layout-sw600dp/activity_map.xml
<LinearLayout
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="horizontal">
    
    <!-- 左侧：地图 -->
    <fragment
        android:id="@+id/mapFragment"
        android:name="com.example.MapFragment"
        android:layout_width="0dp"
        android:layout_height="match_parent"
        android:layout_weight="0.6" />
    
    <!-- 右侧：详情列表 -->
    <fragment
        android:id="@+id/detailFragment"
        android:name="com.example.DetailFragment"
        android:layout_width="0dp"
        android:layout_height="match_parent"
        android:layout_weight="0.4" />
</LinearLayout>

// 方案 4：运行时判断屏幕尺寸
class ScreenAdapter {
    fun isTablet(context: Context): Boolean {
        val displayMetrics = context.resources.displayMetrics
        val screenSizeInInches = sqrt(
            (displayMetrics.widthPixels / displayMetrics.xdpi).pow(2) +
            (displayMetrics.heightPixels / displayMetrics.ydpi).pow(2)
        )
        return screenSizeInInches >= 6.0  // >= 6 英寸认为是平板
    }
    
    fun adjustUIForScreen(context: Context) {
        if (isTablet(context)) {
            // 平板布局：分栏、加大字体等
            showFragmentDetailsPanel()
        } else {
            // 手机布局：全屏展示单个 Fragment
            hideFragmentDetailsPanel()
        }
    }
}
```

**IVI（车机）特殊适配**：

```kotlin
// 车机屏幕通常 7-12 英寸，超宽屏，分辨率固定
// 例如：1920×720（16:6 比例）

// res/layout/activity_dashboard.xml（通用）
<androidx.constraintlayout.widget.ConstraintLayout
    android:layout_width="match_parent"
    android:layout_height="match_parent">
    
    <!-- 上半部分：仪表盘 -->
    <FrameLayout
        android:id="@+id/speedGaugeContainer"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        app:layout_constraintHeight_percent="0.5"
        app:layout_constraintTop_toTopOf="parent" />
    
    <!-- 下半部分：导航 + 媒体 -->
    <FrameLayout
        android:id="@+id/navContainer"
        android:layout_width="0dp"
        android:layout_height="0dp"
        android:layout_weight="0.5"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toStartOf="@id/mediaContainer"
        app:layout_constraintTop_toBottomOf="@id/speedGaugeContainer" />
    
    <FrameLayout
        android:id="@+id/mediaContainer"
        android:layout_width="0dp"
        android:layout_height="0dp"
        android:layout_weight="0.5"
        app:layout_constraintStart_toEndOf="@id/navContainer"
        app:layout_constraintEnd_toEndOf="parent" />
</androidx.constraintlayout.widget.ConstraintLayout>
```

---

**Q18: 说说过度绘制（Overdraw）及其优化方案？**

A:

**什么是过度绘制**：

```
同一像素被绘制多次 → GPU 浪费计算能力 → 帧率下降
```

**常见的过度绘制场景**：

```
❌ 问题 1：嵌套半透明背景
Activity Background（白色）
  ↓
Fragment Background（半透明灰色）
  ↓
View Background（半透明蓝色）
  ↓
Text（黑色）

结果：后台的像素被绘制 3 次以上
```

**检测工具**：

- **开发者选项** → Show GPU Overdraw
- 颜色对应关系：
    - **蓝色** = 1x 绘制（正常）
    - **绿色** = 2x 绘制（轻微）
    - **粉红色** = 3x 绘制（中度）
    - **红色** = 4x+ 绘制（严重）

**优化方案**：

```kotlin
// ❌ 问题 1：不必要的背景堆叠
<LinearLayout
    android:background="@color/white"
    android:layout_width="match_parent"
    android:layout_height="match_parent">
    
    <FrameLayout
        android:background="@drawable/bg_shadow"  // 额外绘制
        android:layout_width="match_parent"
        android:layout_height="match_parent">
        
        <TextView
            android:text="用户信息"
            android:layout_width="match_parent"
            android:layout_height="wrap_content" />
    </FrameLayout>
</LinearLayout>

// ✅ 优化 1：移除不必要的背景
<LinearLayout
    android:background="@color/white"
    android:layout_width="match_parent"
    android:layout_height="match_parent">
    
    <TextView
        android:text="用户信息"
        android:background="@drawable/bg_shadow"  // 只在这里绘制
        android:layout_width="match_parent"
        android:layout_height="wrap_content" />
</LinearLayout>

// ❌ 问题 2：复杂的自定义 View 绘制
class ComplexView : View() {
    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)
        
        // 绘制背景
        canvas.drawRect(0f, 0f, width.toFloat(), height.toFloat(), bgPaint)
        
        // 绘制阴影（低效）
        repeat(10) { i ->
            canvas.drawCircle(
                (width / 2).toFloat(),
                (height / 2).toFloat(),
                100f - i * 10f,
                shadowPaint
            )
        }
        
        // 绘制前景
        canvas.drawText("Text", 50f, 50f, textPaint)
    }
}

// ✅ 优化 2：使用 Canvas 缓存
class OptimizedView : View() {
    private var cachedBitmap: Bitmap? = null
    private var isCacheDirty = true
    
    override fun onDraw(canvas: Canvas) {
        // 仅在数据变化时重绘缓存
        if (isCacheDirty) {
            cachedBitmap = Bitmap.createBitmap(width, height, Bitmap.Config.ARGB_8888)
            val cacheCanvas = Canvas(cachedBitmap!!)
            
            drawComplexContent(cacheCanvas)  // 复杂绘制
            isCacheDirty = false
        }
        
        // 直接绘制缓存
        canvas.drawBitmap(cachedBitmap!!, 0f, 0f, null)
    }
}

// ❌ 问题 3：ListViewRecyclerView 中的过度绘制
class UserAdapter : RecyclerView.Adapter<UserViewHolder>() {
    override fun onBindViewHolder(holder: UserViewHolder, position: Int) {
        holder.itemView.setBackgroundColor(Color.WHITE)  // 每次绑定都设置
        holder.itemView.setBackgroundColor(Color.TRANSPARENT)  // 为了显示列表背景
        
        // 导致背景被绘制 2 次
    }
}

// ✅ 优化 3：复用 ViewHolder
class UserViewHolder(itemView: View) : RecyclerView.ViewHolder(itemView) {
    init {
        // 在初始化时设置一次背景
        itemView.setBackgroundColor(Color.TRANSPARENT)
    }
    
    fun bind(user: User) {
        // 不再重复设置背景
        bindData(user)
    }
}
```

**Compose 中的过度绘制优化**：

```kotlin
// Compose 自动处理大部分过度绘制问题
@Composable
fun UserCard(user: User) {
    // ✅ Compose 只在必要时重新组合
    Box(
        modifier = Modifier
            .fillMaxWidth()
            .background(Color.White)
            .padding(16.dp)
    ) {
        Column {
            Text(text = user.name)
            Text(text = user.email)
        }
    }
}
```

---

## Section 5: 系统问题排查

### 5.1 ANR（应用无响应）

**Q19: 什么是 ANR？常见原因？如何排查？**

A:

**ANR 的定义**：

```
应用 5 秒（Service 10s、ContentProvider 10s）内未能响应用户输入或系统回调
→ 系统认为应用卡死 → 弹出"应用无响应"对话框
```

**常见原因**：

|原因|代码示例|检测方法|
|---|---|---|
|**主线程执行耗时操作**|`Thread.sleep(6000)` 在 onClick|Systrace 看主线程时间线|
|**主线程等待锁**|synchronized 获取锁时被阻塞|`adb shell cat /data/anr/traces.txt`|
|**频繁 GC**|内存抖动导致频繁垃圾回收|Memory Profiler 看锯齿波形|
|**I/O 阻塞**|`SharedPreferences.commit()` 写入|查看 GC/Binder 日志|

**排查工具**：

```bash
# 1. 查看 ANR traces 日志
adb shell cat /data/anr/traces.txt

# 2. 使用 Systrace（系统级性能分析）
python systrace.py --time 5 -o mynewtrace.html gfx view wm

# 3. Android Profiler 中的 CPU Profiler
# 选择 "Sample" 或 "Trace" 模式，看 UI Thread 的时间消耗
```

**代码示例**：

```kotlin
// ❌ ANR 问题 1：主线程睡眠
class MapActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        Thread.sleep(6000)  // 主线程阻塞 6 秒 → ANR
        setContentView(R.layout.activity_map)
    }
}

// ✅ 修复 1：使用协程
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.activity_map)
    
    lifecycleScope.launch {
        delay(6000)  // 后台延迟，不阻塞主线程
        initializeMap()
    }
}

// ❌ ANR 问题 2：onClick 中做网络请求
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    
    button.setOnClickListener {
        val response = retrofit.getRoute().execute()  // 同步网络请求 → ANR
        updateMap(response.body())
    }
}

// ✅ 修复 2：异步请求
button.setOnClickListener {
    lifecycleScope.launch {
        try {
            val response = withContext(Dispatchers.IO) {
                retrofit.getRoute().execute()
            }
            updateMap(response.body())
        } catch (e: Exception) {
            showError(e)
        }
    }
}

// ❌ ANR 问题 3：SharedPreferences 同步写入（主线程）
val sharedPref = getSharedPreferences("config", Context.MODE_PRIVATE)
sharedPref.edit().putString("route_cache", routeJson).commit()  // 同步提交 → 可能 ANR

// ✅ 修复 3：异步写入
sharedPref.edit().putString("route_cache", routeJson).apply()  // 异步提交

// 或使用 DataStore（推荐）
context.dataStore.edit { preferences ->
    preferences[ROUTE_KEY] = routeJson
}
```

**IVI 项目中的 ANR 排查**：

```kotlin
// 车机系统对响应时间要求严格（行驶中操作）
// 触发 ANR 可能导致安全问题

@HiltViewModel
class CabinControlViewModel @Inject constructor(
    private val sensorService: CarSensorService,
    private val repository: CarStateRepository
) : ViewModel() {
    
    fun updateClimateControl(temperature: Int) {
        viewModelScope.launch {
            try {
                // 绝对不要在主线程做这些操作
                withContext(Dispatchers.IO) {
                    // 1. 数据库更新
                    repository.saveClimatePreference(temperature)
                    
                    // 2. 硬件通信（可能阻塞）
                    sensorService.setTargetTemperature(temperature)
                }
                
                // 切回 Main 线程更新 UI
                _cabinState.value = CabinState.Ready(temperature)
            } catch (e: Exception) {
                _cabinState.value = CabinState.Error(e.message)
            }
        }
    }
}
```

---

### 5.2 内存问题进阶

**Q20: 什么是 LeakCanary？如何使用？**

A:

**LeakCanary 工作原理**：

```
1. Heuristics Detection（启发式检测）
   ↓
2. Dump Heap → 分析对象引用链
   ↓
3. 找到 GC roots → 该对象不可达 = 内存泄漏
   ↓
4. 展示泄漏路径
```

**集成使用**：

```kotlin
// build.gradle
dependencies {
    debugImplementation 'com.squareup.leakcanary:leakcanary-android:2.12'
}

// 无需配置，自动初始化！
class MyApplication : Application() {
    // LeakCanary 会通过 ContentProvider 自动启动
}
```

**常见泄漏模式检测**：

```kotlin
// LeakCanary 能自动检测的泄漏
// 1. Activity 泄漏
class LeakyActivity : AppCompatActivity() {
    companion object {
        var instance: Activity? = null
    }
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        instance = this  // ← LeakCanary 标记为可疑
    }
}

// 2. Fragment 泄漏
class LeakyFragment : Fragment() {
    override fun onCreateView(...): View {
        return View(context).apply {
            setOnClickListener {
                val fragment = this@LeakyFragment  // 隐含捕获外部类
            }
        }
    }
}

// 3. 监听器未注销
class LocationActivity : AppCompatActivity() {
    private val listener = OnLocationChangedListener { location ->
        Log.d("Location", location.toString())
    }
    
    override fun onResume() {
        super.onResume()
        locationManager.requestLocationUpdates(..., listener)
    }
    
    // 遗漏 onDestroy() 中的 removeUpdates()
}
```

**LeakCanary 调试工作流**：

```
1. 运行应用并执行导致泄漏的操作
   ↓
2. 销毁可能泄漏的 Activity/Fragment
   ↓
3. LeakCanary 触发 GC，检查堆内存
   ↓
4. 发现泄漏 → 通知栏显示
   ↓
5. 点击通知 → 展示泄漏详情（引用链）
   ↓
6. 根据引用链找到根本原因并修复
```

**优化建议**：

- **不在生产环境使用**：LeakCanary 有性能开销
- **Focus on GC roots**：通常问题出在 static、单例持有 Context
- **自定义排除规则**：某些系统泄漏是已知的，可添加忽略列表

---

## Section 6: 算法与数据结构

### 6.1 常见算法题

**Q21: 手写二分查找算法。有什么变体？**

A:

**标准二分查找**：

```kotlin
fun binarySearch(arr: IntArray, target: Int): Int {
    var left = 0
    var right = arr.size - 1
    
    while (left <= right) {
        val mid = left + (right - left) / 2
        when {
            arr[mid] == target -> return mid
            arr[mid] < target -> left = mid + 1
            else -> right = mid - 1
        }
    }
    return -1  // 未找到
}
```

**变体 1：查找第一个≥target 的元素**（左边界）：

```kotlin
fun searchLeftBound(arr: IntArray, target: Int): Int {
    var left = 0
    var right = arr.size
    
    while (left < right) {
        val mid = left + (right - left) / 2
        when {
            arr[mid] < target -> left = mid + 1
            else -> right = mid
        }
    }
    return left
}

// 应用：在地图中查找最接近当前位置的路线点
val routes = arrayOf(1.0, 2.5, 3.0, 4.2, 5.1)
val position = 3.5
val nextRouteIndex = searchLeftBound(routes.map { it.toInt() }.toIntArray(), position.toInt())
```

**变体 2：查找最后一个≤target 的元素**（右边界）：

```kotlin
fun searchRightBound(arr: IntArray, target: Int): Int {
    var left = 0
    var right = arr.size
    
    while (left < right) {
        val mid = left + (right - left) / 2
        when {
            arr[mid] <= target -> left = mid + 1
            else -> right = mid
        }
    }
    return left - 1
}
```

**在 IVI 应用中的实战场景**：

```kotlin
// 场景：导航中查找当前路线段
data class RouteSegment(val id: Int, val startKm: Double, val endKm: Double, val speedLimit: Int)

class RouteNavigator {
    private val segments = arrayOf(
        RouteSegment(1, 0.0, 10.5, 60),
        RouteSegment(2, 10.5, 25.3, 80),
        RouteSegment(3, 25.3, 50.0, 100),
        RouteSegment(4, 50.0, 78.2, 60)
    )
    
    fun getCurrentSegment(currentKm: Double): RouteSegment? {
        // 使用二分查找找到当前位置所在的路段
        var left = 0
        var right = segments.size - 1
        
        while (left <= right) {
            val mid = left + (right - left) / 2
            when {
                currentKm < segments[mid].startKm -> right = mid - 1
                currentKm >= segments[mid].endKm -> left = mid + 1
                else -> return segments[mid]
            }
        }
        return null
    }
}
```

---

**Q22: 手写快速排序和合并排序。时间复杂度对比？**

A:

**快速排序（平均 O(n log n)，最坏 O(n²)）**：

```kotlin
fun quickSort(arr: IntArray, left: Int = 0, right: Int = arr.size - 1) {
    if (left < right) {
        val pivotIndex = partition(arr, left, right)
        quickSort(arr, left, pivotIndex - 1)
        quickSort(arr, pivotIndex + 1, right)
    }
}

private fun partition(arr: IntArray, left: Int, right: Int): Int {
    val pivot = arr[right]
    var i = left - 1
    
    for (j in left until right) {
        if (arr[j] < pivot) {
            i++
            arr[i] = arr[j].also { arr[j] = arr[i] }
        }
    }
    arr[i + 1] = arr[right].also { arr[right] = arr[i + 1] }
    return i + 1
}
```

**合并排序（O(n log n) 稳定）**：

```kotlin
fun mergeSort(arr: IntArray, left: Int = 0, right: Int = arr.size - 1) {
    if (left < right) {
        val mid = left + (right - left) / 2
        mergeSort(arr, left, mid)
        mergeSort(arr, mid + 1, right)
        merge(arr, left, mid, right)
    }
}

private fun merge(arr: IntArray, left: Int, mid: Int, right: Int) {
    val leftArr = arr.sliceArray(left..mid)
    val rightArr = arr.sliceArray((mid + 1)..right)
    
    var i = 0
    var j = 0
    var k = left
    
    while (i < leftArr.size && j < rightArr.size) {
        if (leftArr[i] <= rightArr[j]) {
            arr[k++] = leftArr[i++]
        } else {
            arr[k++] = rightArr[j++]
        }
    }
    
    while (i < leftArr.size) arr[k++] = leftArr[i++]
    while (j < rightArr.size) arr[k++] = rightArr[j++]
}
```

**对比与选择**：

|特性|快速排序|合并排序|
|---|---|---|
|**平均时间复杂度**|O(n log n)|O(n log n)|
|**最坏时间复杂度**|O(n²)|O(n log n)|
|**空间复杂度**|O(log n)【递归栈】|O(n)【临时数组】|
|**稳定性**|不稳定|稳定|
|**缓存友好性**|好（局部性强）|一般|
|**Android 中的应用**|Comparator.sort|Collections.sort（外部排序大数据）|

**在 IVI 导航中的应用**：

```kotlin
// 场景：按距离排序附近的加油站
data class GasStation(val id: Int, val name: String, val distance: Double)

class GasStationFinder {
    fun findNearestStations(current: Location, allStations: List<GasStation>, limit: Int = 5): List<GasStation> {
        // 计算距离并排序
        return allStations
            .map { station ->
                val distance = calculateDistance(current, station.location)
                station.copy(distance = distance)
            }
            .sortedBy { it.distance }  // Kotlin 自动选择最优排序算法
            .take(limit)
    }
}

// 在 Comparator 中使用自定义排序
class RouteSorter {
    // 按路线优先级排序
    val routeComparator = compareBy<Route> { it.priority }
        .thenBy { it.distance }
        .thenBy { it.estimatedTime }
    
    fun sortRoutes(routes: List<Route>): List<Route> {
        return routes.sortedWith(routeComparator)
    }
}
```

---

### 6.2 数据结构

**Q23: HashMap 的原理是什么？Hash 碰撞如何解决？**

A:

**HashMap 的内部结构**：

```
┌─────────────────────────────────────────────┐
│ HashMap (16 个 bucket 的数组)               │
├─────────────────────────────────────────────┤
│[0] → null                                   │
│[1] → Entry(key="name", value="Alice")       │
│      → Entry(key="age", value=30)           │ ← 链表（碰撞处理）
│[2] → null                                   │
│...                                         │
│[15] → Entry(key="city", value="NYC")        │
└─────────────────────────────────────────────┘
```

**Put 操作流程**：

```kotlin
fun put(key: K, value: V) {
    // 1. 计算 hash 值
    val hash = key.hashCode()
    
    // 2. 计算 bucket 索引（与数组长度 & 操作）
    val index = hash and (capacity - 1)
    
    // 3. 遍历 bucket 中的链表/红黑树
    var entry = table[index]
    while (entry != null) {
        if (entry.key.equals(key)) {
            entry.value = value  // 更新已存在的 key
            return
        }
        entry = entry.next
    }
    
    // 4. 未找到，创建新 Entry
    addEntry(key, value, index)
    
    // 5. 如果 size > threshold，扩容
    if (size > loadFactor * capacity) {
        resize()
    }
}
```

**Hash 碰撞的解决方案**：

|方案|实现|优缺点|
|---|---|---|
|**链表法**|多个 Entry 形成链表|简单；链表长时查询 O(n)|
|**红黑树**|Java 8+ 链表长>8 转红黑树|查询 O(log n)；复杂度增加|
|**开放寻址**|冲突时探测下一个空槽|缓存友好；但容易聚集|
|**双哈希**|用第二个哈希函数重新哈希|分散性好；实现复杂|

**Java 8+ HashMap 的优化**：

```java
// 当 bucket 中的链表长度 >= 8 时，转换为红黑树
if (binCount >= TREEIFY_THRESHOLD - 1) {
    treeifyBin(tab, hash);
}

// 当红黑树节点 <= 6 时，转回链表
if (tab != null && (tab.length > 64 || (tab.length) >= UNTREEIFY_THRESHOLD)) {
    untreeify(tab);
}
```

**Load Factor（负载因子）的影响**：

```
负载因子 = size / capacity

默认 0.75 的权衡：
- 太低（0.25）：浪费空间，但碰撞少
- 太高（0.9）：空间省，但碰撞多，查询变慢
- 0.75：空间 vs 性能的最优平衡
```

**在 IVI 缓存中的应用**：

```kotlin
// 导航路线缓存 - 使用 HashMap
class RouteCache {
    private val cache = mutableMapOf<String, CachedRoute>()  // HashMap 底层实现
    
    fun getRoute(startId: String, endId: String): CachedRoute? {
        val key = "$startId-$endId"
        return cache[key]
    }
    
    fun cacheRoute(startId: String, endId: String, route: Route) {
        val key = "$startId-$endId"
        cache[key] = CachedRoute(route, System.currentTimeMillis())
    }
}

// LinkedHashMap - 保留插入顺序（用于 LRU 缓存）
class LRURouteCache(private val maxSize: Int) {
    private val cache = LinkedHashMap<String, CachedRoute>(16, 0.75f, true) {
        override fun removeEldestEntry(eldest: Map.Entry<String, CachedRoute>?): Boolean {
            return size > maxSize  // 超过 maxSize 时移除最久未用的
        }
    }
}
```

---

**Q24: 说说哈希表、TreeMap、LinkedHashMap 的区别？**

A:

|特性|HashMap|TreeMap|LinkedHashMap|
|---|---|---|---|
|**底层结构**|哈希表+链表/红黑树|红黑树|哈希表+双向链表|
|**有序性**|无序|按 key 排序（升序）|按插入顺序（可配置访问顺序）|
|**查询时间**|O(1) 平均|O(log n)|O(1) 平均|
|**插入/删除时间**|O(1) 平均|O(log n)|O(1) 平均|
|**空间复杂度**|O(n)|O(n)|O(n)|
|**线程安全**|否；Collections.synchronizedMap()|否；Collections.synchronizedSortedMap()|否|
|**应用场景**|通用缓存、快速查询|范围查询、排序|LRU 缓存、FIFO 队列|

**代码对比**：

```kotlin
// HashMap - 无序，快查询
val map = hashMapOf(3 to "C", 1 to "A", 2 to "B")
map.forEach { println(it) }  // 输出顺序不确定：可能 1-2-3 或 3-1-2

// TreeMap - 有序（按 key 升序），支持范围查询
val treeMap = sortedMapOf(3 to "C", 1 to "A", 2 to "B")
treeMap.forEach { println(it) }  // 输出：1-2-3
println(treeMap.subMap(1, 3))  // 范围查询：1..3

// LinkedHashMap - 按插入顺序
val linkedMap = linkedMapOf(3 to "C", 1 to "A", 2 to "B")
linkedMap.forEach { println(it) }  // 输出：3-1-2（插入顺序）

// LinkedHashMap 访问顺序版本（LRU）
val lruMap = object : LinkedHashMap<String, String>(16, 0.75f, true) {
    override fun removeEldestEntry(eldest: Map.Entry<String, String>?): Boolean {
        return size > 3  // 保持最多 3 个元素
    }
}

lruMap["a"] = "1"
lruMap["b"] = "2"
lruMap["c"] = "3"
_ = lruMap["a"]  // 访问 "a"，变成最近使用
lruMap["d"] = "4"  // 超容量，移除最久未用的 "b"
println(lruMap)  // {a=1, c=3, d=4}
```

**在 IVI 系统中的应用**：

```kotlin
// 1. HashMap - 快速查询传感器数据
class SensorDataCache {
    private val sensorReadings = mutableMapOf<String, SensorReading>()
    
    fun getSensorReading(sensorName: String): SensorReading? {
        return sensorReadings[sensorName]  // O(1) 查询
    }
}

// 2. TreeMap - 查询速度范围内的加油站
class GasStationIndex {
    private val stationsByDistance = sortedMapOf<Double, GasStation>()
    
    fun findStationsInRange(minKm: Double, maxKm: Double): List<GasStation> {
        return stationsByDistance.subMap(minKm, maxKm).values.toList()
    }
}

// 3. LinkedHashMap - LRU 路线缓存
class RouteCache(private val maxCacheSize: Int = 10) {
    private val cache = object : LinkedHashMap<String, Route>(16, 0.75f, true) {
        override fun removeEldestEntry(eldest: Map.Entry<String, Route>?): Boolean {
            return size > maxCacheSize
        }
    }
    
    fun getRoute(key: String): Route? = cache[key]
    
    fun cacheRoute(key: String, route: Route) {
        cache[key] = route
    }
}
```

---

## Section 7: 架构设计与项目经验

### 7.1 架构模式

**Q25: 说说 MVVM、MVP、MVC 架构模式。在 Android 中如何选择？**

A:

**三种架构对比**：

|架构|数据流|UI 更新|测试性|复杂度|使用场景|
|---|---|---|---|---|---|
|**MVC**|Model←→Controller←→View|Controller 更新 View|弱|低|简单应用|
|**MVP**|Model←→Presenter←→View|Presenter 更新 View（通过接口）|中|中|中等复杂应用|
|**MVVM**|Model←→ViewModel←→View（单向数据流）|数据绑定自动更新 View|强|高|复杂业务逻辑|

**具体代码对比**：

```kotlin
// ❌ MVC（不推荐） - Activity 混合 Controller 和 View 职责
class UserListActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_user_list)
        
        // Controller 逻辑混在 Activity 中
        val users = loadUsersFromDatabase()  // ← Model 操作
        adapter.setData(users)  // ← View 操作
        
        button.setOnClickListener {
            val newUser = createNewUser()  // ← 混乱：业务逻辑在 View 层
            saveUserToDatabase(newUser)
            refreshList()
        }
    }
}

// ✅ MVP - Model、Presenter、View 职责清晰
// 1. Model 层
interface UserRepository {
    suspend fun getUsers(): List<User>
    suspend fun createUser(name: String): User
}

// 2. Presenter 层
class UserListPresenter(
    private val repository: UserRepository
) {
    private var view: UserListView? = null
    
    fun attachView(view: UserListView) {
        this.view = view
    }
    
    suspend fun loadUsers() {
        val users = repository.getUsers()
        view?.showUsers(users)  // 通过接口与 View 通信
    }
    
    suspend fun createAndAddUser(name: String) {
        val newUser = repository.createUser(name)
        view?.addUser(newUser)
    }
}

// 3. View 层（接口）
interface UserListView {
    fun showUsers(users: List<User>)
    fun addUser(user: User)
    fun showError(error: String)
}

// 4. Activity 实现 View 接口
class UserListActivity : AppCompatActivity(), UserListView {
    private lateinit var presenter: UserListPresenter
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_user_list)
        
        presenter.attachView(this)
        presenter.loadUsers()
        
        button.setOnClickListener {
            presenter.createAndAddUser(nameInput.text.toString())
        }
    }
    
    override fun showUsers(users: List<User>) {
        adapter.setData(users)
    }
    
    override fun addUser(user: User) {
        adapter.addItem(user)
    }
    
    override fun showError(error: String) {
        Toast.makeText(this, error, Toast.LENGTH_SHORT).show()
    }
}

// ✅ MVVM（推荐） - ViewModel + LiveData/StateFlow
// 1. Model 层（同上）
interface UserRepository {
    suspend fun getUsers(): List<User>
    suspend fun createUser(name: String): User
}

// 2. ViewModel 层
@HiltViewModel
class UserListViewModel @Inject constructor(
    private val repository: UserRepository
) : ViewModel() {
    
    // 暴露状态给 View（单向数据流）
    private val _userList = MutableStateFlow<List<User>>(emptyList())
    val userList: StateFlow<List<User>> get() = _userList.asStateFlow()
    
    private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
    val uiState: StateFlow<UiState> get() = _uiState.asStateFlow()
    
    init {
        loadUsers()
    }
    
    private fun loadUsers() {
        viewModelScope.launch {
            try {
                _uiState.value = UiState.Loading
                val users = withContext(Dispatchers.IO) {
                    repository.getUsers()
                }
                _userList.value = users
                _uiState.value = UiState.Success
            } catch (e: Exception) {
                _uiState.value = UiState.Error(e.message ?: "Unknown error")
            }
        }
    }
    
    fun createAndAddUser(name: String) {
        viewModelScope.launch {
            try {
                val newUser = withContext(Dispatchers.IO) {
                    repository.createUser(name)
                }
                _userList.value = _userList.value + newUser
            } catch (e: Exception) {
                _uiState.value = UiState.Error(e.message)
            }
        }
    }
}

// 3. View 层（简化）
class UserListFragment : Fragment() {
    private val viewModel: UserListViewModel by viewModels()
    
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        // 观察数据，自动更新 UI
        viewLifecycleOwner.lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.userList.collect { users ->
                    adapter.setData(users)
                }
            }
        }
        
        // 监听 UI 状态
        viewLifecycleOwner.lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.uiState.collect { state ->
                    when (state) {
                        UiState.Loading -> showLoadingIndicator()
                        UiState.Success -> hideLoadingIndicator()
                        is UiState.Error -> showError(state.message)
                    }
                }
            }
        }
        
        // 用户交互
        button.setOnClickListener {
            viewModel.createAndAddUser(nameInput.text.toString())
        }
    }
}

// 4. UI State 定义
sealed class UiState {
    object Loading : UiState()
    object Success : UiState()
    data class Error(val message: String) : UiState()
}
```

**在 IVI 项目中的应用 - MVVM 最适合**：

```kotlin
// IVI 系统通常复杂度高，MVVM 能很好管理状态

@HiltViewModel
class CabinControlViewModel @Inject constructor(
    private val sensorService: CarSensorService,
    private val repository: CarStateRepository
) : ViewModel() {
    
    // 单向数据流：数据 → ViewModel → UI
    private val _cabinState = MutableStateFlow<CabinState>(CabinState.Idle)
    val cabinState: StateFlow<CabinState> get() = _cabinState.asStateFlow()
    
    private val _temperature = MutableStateFlow(20)
    val temperature: StateFlow<Int> get() = _temperature.asStateFlow()
    
    init {
        observeSensorData()
    }
    
    private fun observeSensorData() {
        viewModelScope.launch {
            sensorService.temperatureSensor
                .collect { temp ->
                    _temperature.value = temp
                }
        }
    }
    
    fun setTargetTemperature(temp: Int) {
        viewModelScope.launch {
            try {
                repository.updateClimateControl(temp)
                _temperature.value = temp
            } catch (e: Exception) {
                _cabinState.value = CabinState.Error(e.message)
            }
        }
    }
}
```

---

**Q26: 说说你参与过的一个商业级应用开发。项目难点、如何解决？**

A:

> 这是行为问题，面试官想了解你的：
> 
> 1. **实际项目经验**
> 2. **问题分析与解决能力**
> 3. **技术决策能力**
> 4. **团队协作**

**建议答题框架**：

```
1. 项目背景（2分钟）
   - 项目名称、规模、团队人数
   - 你的角色职责
   - 项目难度等级

2. 核心功能 + 技术栈（2分钟）
   - 主要功能模块
   - 使用的技术框架
   - 为什么选择这些技术

3. 遇到的主要难点（3分钟）
   - 难点 1：具体问题是什么
   - 难点 2：分析根因
   - 难点 3：最后如何解决

4. 个人贡献（2分钟）
   - 你负责的模块
   - 取得的成果（性能指标、用户反馈等）
   - 学到了什么
```

**参考答案（基于 IVI 项目）**：

```
【项目背景】
项目名称：某车企的智能座舱 IVI 系统
团队规模：8 人（3 个 Android、2 个前端、3 个后端）
项目周期：18 个月

我作为高级 Android 工程师，主要负责：
- 仪表盘模块（速度、油量、温度显示）
- 导航集成模块
- 蓝牙通话模块

【核心技术栈】
- 架构：MVVM + Repository Pattern + Hilt DI
- UI 框架：Jetpack Compose（新版仪表盘）+ 传统 View（兼容旧系统）
- 数据存储：Room + DataStore
- 后端通信：Retrofit + OkHttp + RxJava
- 传感器数据：Native C++ 采集 + JNI 传递

【遇到的难点】

难点 1：传感器数据实时更新导致帧率下降
问题：
- 仪表盘需要 60 FPS 显示
- 传感器数据每 16ms 更新一次
- 每次更新都触发 UI 刷新 → 帧率低至 30 FPS → 用户感受到卡顿

分析：
- 使用 Memory Profiler 发现频繁 GC（内存抖动）
- 每次更新创建新数据对象 → GC 压力大
- Compose 重新组合过于频繁

解决方案：
1. 改用对象池模式复用 SensorEvent 对象
2. 使用 throttle 限制数据更新频率（16ms 一次）
3. Compose 中用 remember 缓存计算结果
4. 开启 R8 代码混淆优化

结果：帧率稳定在 58-60 FPS


难点 2：蓝牙连接稳定性差
问题：
- 用户在高速行驶时蓝牙经常断连
- 断连后自动重连延迟 5-10 秒
- 影响通话连续性和用户体验

分析：
- 蓝牙 RSSI 值在 -80 到 -95 dBm 时容易断连
- 原有代码没有预连接（等断连后再连）
- 状态机管理不够细致

解决方案：
1. 实现 Bluetooth State Machine
   - IDLE → SCANNING → FOUND → CONNECTING → CONNECTED → 若 RSSI < -90 → 预重连
2. 使用 RxJava 的重试机制
   ````kotlin
   bluetoothClient.connect()
       .retry { error, attemptCount ->
           attemptCount < 3 && error is BluetoothException
       }
       .delaySubscription(Duration.ofMillis(1000 * attemptCount))
```

3. 在后台定期监测 RSSI，提前预连接

结果：平均重连时间从 8s 降至 2s，断连频率降低 80%

难点 3：导航地图大文件加载慢 问题：

- 某些地区地图包 200+ MB
- 初次启动要 30+ 秒加载完
- 用户体验差

分析：

- 地图数据采用单文件，无法分块加载
- Bitmap 解析时一次性加载全部像素
- 磁盘 I/O 成为瓶颈

解决方案：

1. 地图分片加载
    
    ```kotlin
    class TiledMapLoader {    private val tileSize = 512  // 512×512 像素一个 tile        fun loadTile(x: Int, y: Int, zoom: Int): Bitmap {        return BitmapFactory.decodeFile(getTileFile(x, y, zoom))    }}
    ```
    
2. 使用 LRU 缓存保存最近使用的 tile
3. 后台线程预加载相邻 tile
4. 地图压缩：原始 GeoJSON → Protocol Buffers（减小 40%）

结果：初次启动时间从 30s → 3s（只加载当前视区）

【个人成长】 通过这个项目，我深入理解了：

- 性能优化的方法论（profiling → 分析 → 优化 → 验证）
- 状态机在复杂业务中的应用
- 蓝牙、传感器等系统级编程
- 大型应用的架构设计（模块解耦、依赖注入）

项目成果：

- 最终应用 Google Play 评分 4.2★（19k+ 评论）
- 性能指标：帧率 58-60 FPS，内存占用 180 MB（对标行业平均 280 MB）

```

---

### 7.2 网络与数据同步

**Q27: 说说 Retrofit + OkHttp + RxJava 的组合应用。如何处理网络错误和重试？**

A:

**架构设计**：
```

┌─────────────────┐ │ ViewModel │ └────────┬────────┘ ↓ ┌─────────────────┐ │ Repository │ ← 数据层抽象 └────────┬────────┘ ↓ ┌──────────────────────────────────────┐ │ RetrofitService (API 调用) │ └────────┬──────────────────────────────┘ ↓ ┌──────────────────────────────────────┐ │ OkHttpClient (网络层) │ │ - 拦截器（日志、重试、超时等） │ │ - 连接池、缓存 │ └──────────────────────────────────────┘

````

**完整代码实现**：

```kotlin
// 1. 定义 API 接口
interface CarRouteService {
    @GET("/api/routes/{routeId}")
    suspend fun getRoute(@Path("routeId") routeId: String): ApiResponse<Route>
    
    @POST("/api/routes")
    suspend fun createRoute(@Body route: Route): ApiResponse<RouteId>
}

// 2. 创建 OkHttpClient（配置拦截器、超时等）
class OkHttpClientFactory {
    companion object {
        fun create(): OkHttpClient {
            return OkHttpClient.Builder()
                // 添加日志拦截器
                .addInterceptor(HttpLoggingInterceptor().apply {
                    level = if (BuildConfig.DEBUG) 
                        HttpLoggingInterceptor.Level.BODY 
                    else 
                        HttpLoggingInterceptor.Level.NONE
                })
                // 添加重试拦截器
                .addInterceptor(RetryInterceptor())
                // 添加请求头拦截器（token、签名等）
                .addInterceptor(HeaderInterceptor())
                // 配置超时
                .connectTimeout(15, TimeUnit.SECONDS)
                .readTimeout(20, TimeUnit.SECONDS)
                .writeTimeout(20, TimeUnit.SECONDS)
                // 连接池
                .connectionPool(ConnectionPool(8, 5, TimeUnit.MINUTES))
                // 缓存策略
                .cache(Cache(cacheDir, 50 * 1024 * 1024))  // 50MB 缓存
                .build()
        }
    }
}

// 3. 重试拦截器
class RetryInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        var request = chain.request()
        var response: Response? = null
        var exception: Exception? = null
        
        // 重试 3 次
        for (i in 0 until 3) {
            try {
                response = chain.proceed(request)
                
                // 如果是 5xx 错误或网络问题，继续重试
                if (response.isSuccessful || !shouldRetry(response)) {
                    return response
                }
            } catch (e: IOException) {
                exception = e
                // 网络异常，继续重试
            }
            
            // 指数退避（避免雷鸣羊群效应）
            if (i < 2) {
                Thread.sleep(1000L * (i + 1))  // 1s, 2s, ...
            }
        }
        
        return response ?: throw exception ?: IOException("Max retries exceeded")
    }
    
    private fun shouldRetry(response: Response): Boolean {
        return response.code in 500..599 || response.code == 408  // 5xx 或超时
    }
}

// 4. 请求头拦截器
class HeaderInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val originalRequest = chain.request()
        val requestWithHeaders = originalRequest.newBuilder()
            .addHeader("User-Agent", "IVI-System/1.0")
            .addHeader("Authorization", "Bearer ${getAuthToken()}")
            .addHeader("X-Request-ID", generateRequestId())
            .build()
        
        return chain.proceed(requestWithHeaders)
    }
}

// 5. Retrofit 配置
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    @Provides
    @Singleton
    fun provideRetrofit(): Retrofit {
        return Retrofit.Builder()
            .baseUrl("https://api.carservice.com/")
            .client(OkHttpClientFactory.create())
            .addConverterFactory(GsonConverterFactory.create())
            .addCallAdapterFactory(SuspendingCoroutineCallAdapterFactory())  // 协程支持
            .build()
    }
    
    @Provides
    @Singleton
    fun provideCarRouteService(retrofit: Retrofit): CarRouteService {
        return retrofit.create(CarRouteService::class.java)
    }
}

// 6. Repository 层（处理数据和错误）
@Singleton
class RouteRepository @Inject constructor(
    private val apiService: CarRouteService,
    private val routeDao: RouteDao
) {
    // 返回 Result 类型进行错误处理
    suspend fun getRoute(routeId: String): Result<Route> = runCatching {
        try {
            // 1. 尝试从网络获取
            val response = apiService.getRoute(routeId)
            
            if (response.success) {
                val route = response.data
                // 2. 保存到本地数据库（离线模式）
                routeDao.insertRoute(route)
                route
            } else {
                // 3. API 返回业务级错误
                throw ApiException(response.errorCode, response.errorMessage)
            }
        } catch (e: IOException) {
            // 4. 网络异常，尝试使用本地缓存
            val cachedRoute = routeDao.getRoute(routeId)
            if (cachedRoute != null) {
                cachedRoute
            } else {
                throw NetworkException("无网络且无本地缓存", e)
            }
        }
    }
}

// 7. ViewModel 中的使用
@HiltViewModel
class RouteViewModel @Inject constructor(
    private val repository: RouteRepository
) : ViewModel() {
    
    private val _routeState = MutableStateFlow<RouteState>(RouteState.Loading)
    val routeState: StateFlow<RouteState> get() = _routeState.asStateFlow()
    
    fun loadRoute(routeId: String) {
        viewModelScope.launch {
            try {
                _routeState.value = RouteState.Loading
                val result = repository.getRoute(routeId)
                
                result
                    .onSuccess { route ->
                        _routeState.value = RouteState.Success(route)
                    }
                    .onFailure { error ->
                        val message = when (error) {
                            is ApiException -> "API 错误: ${error.message} (${error.code})"
                            is NetworkException -> "网络错误: ${error.message}"
                            else -> "未知错误: ${error.message}"
                        }
                        _routeState.value = RouteState.Error(message)
                    }
            } catch (e: Exception) {
                _routeState.value = RouteState.Error(e.message ?: "Unknown error")
            }
        }
    }
}

// 8. 自定义异常
sealed class NetworkException(message: String) : Exception(message)
class ApiException(val code: Int, message: String) : NetworkException(message)
class NetworkException(message: String, cause: Throwable) : NetworkException(message)

// 9. UI 状态
sealed class RouteState {
    object Loading : RouteState()
    data class Success(val route: Route) : RouteState()
    data class Error(val message: String) : RouteState()
}

// 10. API 响应包装
data class ApiResponse<T>(
    val success: Boolean,
    val errorCode: Int = 0,
    val errorMessage: String = "",
    val data: T? = null
)
````

**错误处理最佳实践**：

```kotlin
// ✅ 统一的错误处理
suspend inline fun <T> safeApiCall(
    crossinline call: suspend () -> T
): Result<T> = try {
    Result.success(call())
} catch (e: Exception) {
    when (e) {
        is HttpException -> Result.failure(ApiException(e.code(), e.message()))
        is IOException -> Result.failure(NetworkException("网络错误", e))
        is TimeoutException -> Result.failure(NetworkException("请求超时", e))
        else -> Result.failure(e)
    }
}

// 使用
val result = safeApiCall { apiService.getRoute(routeId) }
result.fold(
    onSuccess = { route -> updateUI(route) },
    onFailure = { error -> showError(error) }
)
```

---

**（完）**

---

## 💡 面试备考建议

### 高频必考题（重中之重）

1. **Lifecycle 机制** ✅
2. **ViewModel 生命周期** ✅
3. **Hilt/Dagger2 依赖注入** ✅
4. **线程池与协程** ✅
5. **内存泄漏与 OOM** ✅
6. **ANR 排查** ✅
7. **MVVM 架构** ✅
8. **Retrofit + OkHttp** ✅

### 备考计划（4-6 周）

- **第 1 周**：Java/OOP + 设计模式
- **第 2 周**：Android 核心框架（Lifecycle、ViewModel、Hilt）
- **第 3 周**：并发与性能优化（线程池、协程、内存管理）
- **第 4 周**：UI 框架与屏幕适配
- **第 5 周**：系统问题排查（ANR、OOM）+ 算法
- **第 6 周**：架构设计 + 项目实战复习

### 答题技巧

1. **回答要分层**：先讲原理，再讲应用，最后讲优化
2. **配合代码示例**：不要只讲概念
3. **突出个人经验**：从项目经验说起
4. **体现技术深度**：问题如何一步步解决的
5. **关注性能指标**：用数字说话（帧率、内存占用、加载时间等）

---

现在你可以：

1. 针对每个题目深入研究相关源码
2. 在自己的项目中实践这些知识
3. 模拟面试，特别是架构设计题

有任何题目需要深入讨论的，随时告诉我！🚀