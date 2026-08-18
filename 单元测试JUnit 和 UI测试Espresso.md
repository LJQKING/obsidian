
## 一、核心概念

| 类型                 | 作用域    | 运行环境           | 速度  | 用途            |
| ------------------ | ------ | -------------- | --- | ------------- |
| **JUnit 单元测试**     | 单个类/方法 | 本地 JVM         | 快   | 测试业务逻辑、工具类    |
| **Espresso UI 测试** | 用户交互流程 | Android 设备/模拟器 | 慢   | 测试 UI 行为、用户操作 |

---

## 二、JUnit 单元测试

### 2.1 基本使用

**build.gradle 依赖**：

```gradle
dependencies {
    testImplementation 'junit:junit:4.13.2'
    testImplementation 'org.mockito:mockito-core:5.0.0'
    testImplementation 'org.mockito:mockito-inline:5.0.0'
}
```

**示例1：测试简单的业务逻辑**

```kotlin
// ViewModel 业务逻辑
class UserViewModel {
    private val _userEmail = MutableLiveData<String>()
    val userEmail: LiveData<String> = _userEmail
    
    fun validateEmail(email: String): Boolean {
        val isValid = email.matches(Regex("^[A-Za-z0-9+_.-]+@(.+)$"))
        if (isValid) {
            _userEmail.value = email
        }
        return isValid
    }
    
    fun calculateAge(birthYear: Int): Int {
        return 2024 - birthYear
    }
}

// 对应的测试类
@RunWith(MockitoJUnitRunner::class)
class UserViewModelTest {
    
    private lateinit var viewModel: UserViewModel
    
    @Before  // 每个测试方法前执行
    fun setUp() {
        viewModel = UserViewModel()
    }
    
    @Test
    fun testValidEmailFormat() {
        val result = viewModel.validateEmail("user@example.com")
        assertTrue(result)
    }
    
    @Test
    fun testInvalidEmailFormat() {
        val result = viewModel.validateEmail("invalid-email")
        assertFalse(result)
    }
    
    @Test
    fun testCalculateAgeCorrectly() {
        val age = viewModel.calculateAge(1995)
        assertEquals(29, age)
    }
    
    @Test
    fun testEmailUpdatedAfterValidation() {
        viewModel.validateEmail("test@qq.com")
        assertEquals("test@qq.com", viewModel.userEmail.value)
    }
    
    @After  // 每个测试方法后执行
    fun tearDown() {
        // 清理资源
    }
}
```

### 2.2 使用 Mock 模拟依赖

```kotlin
// 数据仓库依赖
interface UserRepository {
    fun getUserInfo(userId: String): User
    fun saveUser(user: User): Boolean
}

// ViewModel 依赖注入
class UserDetailViewModel(private val repository: UserRepository) {
    fun loadUser(userId: String): String {
        val user = repository.getUserInfo(userId)
        return "${user.name}(${user.age}岁)"
    }
    
    fun updateUserAge(userId: String, newAge: Int): Boolean {
        val user = repository.getUserInfo(userId).copy(age = newAge)
        return repository.saveUser(user)
    }
}

// 测试类 - 使用 Mockito 模拟 Repository
@RunWith(MockitoJUnitRunner::class)
class UserDetailViewModelTest {
    
    @Mock
    private lateinit var mockRepository: UserRepository
    
    @InjectMocks  // 自动注入 Mock 对象
    private lateinit var viewModel: UserDetailViewModel
    
    @Before
    fun setUp() {
        MockitoAnnotations.openMocks(this)
    }
    
    @Test
    fun testLoadUserInfo() {
        // 1. 准备：设定 Mock 返回值
        val mockUser = User(id = "1", name = "张三", age = 28)
        `when`(mockRepository.getUserInfo("1")).thenReturn(mockUser)
        
        // 2. 执行
        val result = viewModel.loadUser("1")
        
        // 3. 验证
        assertEquals("张三(28岁)", result)
        verify(mockRepository, times(1)).getUserInfo("1")  // 验证调用次数
    }
    
    @Test
    fun testUpdateUserAge() {
        val mockUser = User(id = "1", name = "李四", age = 25)
        `when`(mockRepository.getUserInfo("1")).thenReturn(mockUser)
        `when`(mockRepository.saveUser(any())).thenReturn(true)
        
        val result = viewModel.updateUserAge("1", 26)
        
        assertTrue(result)
        // 验证保存方法被调用
        verify(mockRepository).saveUser(argThat {
            this.age == 26
        })
    }
}

data class User(val id: String, val name: String, val age: Int)
```

### 2.3 JUnit 的运行原理

```
┌─────────────────────────────────────────┐
│ JUnit Test Execution Flow               │
├─────────────────────────────────────────┤
│ 1. TestRunner 启动 (默认使用反射)        │
│    ├─ 扫描 @Test 标注的方法              │
│    └─ 构建 Test Suite                   │
│                                         │
│ 2. 生命周期控制                          │
│    ├─ @BeforeClass (类级别,仅一次)      │
│    ├─ For Each Test:                    │
│    │  ├─ @Before                        │
│    │  ├─ @Test                          │
│    │  ├─ @After                         │
│    │  └─ 记录结果                        │
│    └─ @AfterClass (类级别,仅一次)       │
│                                         │
│ 3. 断言失败时抛出异常 (AssertionError) │
│                                         │
│ 4. 生成测试报告                          │
└─────────────────────────────────────────┘
```

**内部机制**：

```kotlin
// JUnit 原理简化版本
abstract class TestCase {
    fun runTest(testName: String) {
        val testMethod = this.javaClass.getMethod(testName)
        
        // 1. 前置
        this.setUp()
        
        try {
            // 2. 执行测试
            testMethod.invoke(this)
        } catch (e: Exception) {
            // 3. 失败处理
            throw AssertionError("Test failed", e)
        } finally {
            // 4. 后置清理
            this.tearDown()
        }
    }
}
```

---

## 三、Espresso UI 测试

### 3.1 基本使用

**build.gradle 依赖**：

```gradle
dependencies {
    androidTestImplementation 'androidx.test.espresso:espresso-core:3.5.1'
    androidTestImplementation 'androidx.test:runner:1.5.2'
    androidTestImplementation 'androidx.test:rules:1.5.0'
    androidTestImplementation 'androidx.test.ext:junit:1.1.5'
}

android {
    defaultConfig {
        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
    }
}
```

**示例1：简单的 UI 交互测试**

```kotlin
// 被测试的 Activity
class LoginActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_login)
        
        val emailInput = findViewById<EditText>(R.id.email_input)
        val passwordInput = findViewById<EditText>(R.id.password_input)
        val loginBtn = findViewById<Button>(R.id.login_btn)
        val errorMsg = findViewById<TextView>(R.id.error_msg)
        
        loginBtn.setOnClickListener {
            val email = emailInput.text.toString()
            val password = passwordInput.text.toString()
            
            if (email.isEmpty() || password.isEmpty()) {
                errorMsg.text = "邮箱和密码不能为空"
                errorMsg.visibility = View.VISIBLE
            } else {
                // 执行登录
                performLogin(email, password)
            }
        }
    }
    
    private fun performLogin(email: String, password: String) {
        // 登录逻辑
    }
}

// Espresso 测试
@RunWith(AndroidJUnit4::class)
class LoginActivityTest {
    
    @get:Rule
    val activityRule = ActivityScenarioRule(LoginActivity::class.java)
    
    @Test
    fun testEmptyEmailShowsError() {
        // 1. 查找元素 (onView)
        onView(withId(R.id.email_input))
            .perform(typeText(""))  // 清空
        
        onView(withId(R.id.password_input))
            .perform(typeText("password123"))
        
        // 2. 点击按钮
        onView(withId(R.id.login_btn))
            .perform(click())
        
        // 3. 检查错误提示是否显示
        onView(withId(R.id.error_msg))
            .check(matches(isDisplayed()))
            .check(matches(withText("邮箱和密码不能为空")))
    }
    
    @Test
    fun testSuccessfulLogin() {
        onView(withId(R.id.email_input))
            .perform(typeText("user@example.com"))
        
        onView(withId(R.id.password_input))
            .perform(typeText("password123"))
        
        onView(withId(R.id.login_btn))
            .perform(click())
        
        // 验证导航到主界面（假设主界面有特定的 TextView）
        onView(withText("欢迎登录"))
            .check(matches(isDisplayed()))
    }
}
```

### 3.2 高级用法：RecyclerView 和列表测试

```kotlin
@RunWith(AndroidJUnit4::class)
class ProductListActivityTest {
    
    @get:Rule
    val activityRule = ActivityScenarioRule(ProductListActivity::class.java)
    
    @Test
    fun testClickItemInList() {
        // 在 RecyclerView 中查找并点击指定位置的项
        onView(withId(R.id.product_list))
            .perform(RecyclerViewActions.actionOnItemAtPosition<ProductAdapter.ViewHolder>(
                0,  // 第一个项
                click()
            ))
        
        // 验证导航到详情页
        onView(withText("产品详情"))
            .check(matches(isDisplayed()))
    }
    
    @Test
    fun testScrollAndFindItem() {
        // 滚动到包含特定文本的项
        onView(withId(R.id.product_list))
            .perform(
                RecyclerViewActions.scrollTo<ProductAdapter.ViewHolder>(
                    hasDescendant(withText("iPhone 15"))
                )
            )
            .perform(click())
    }
    
    @Test
    fun testListItemCount() {
        onView(withId(R.id.product_list))
            .check(matches(RecyclerViewMatcher.hasItemCount(10)))
    }
}

// 自定义 Matcher
class RecyclerViewMatcher {
    companion object {
        fun hasItemCount(count: Int): Matcher<View> {
            return object : TypeSafeMatcher<View>() {
                override fun matchesSafely(view: View): Boolean {
                    val recyclerView = view as? RecyclerView
                    return recyclerView?.adapter?.itemCount == count
                }
                
                override fun describeTo(description: Description) {
                    description.appendText("RecyclerView item count: $count")
                }
            }
        }
    }
}
```

### 3.3 Espresso 的执行原理

```
┌───────────────────────────────────────────────┐
│ Espresso Execution Architecture               │
├───────────────────────────────────────────────┤
│                                               │
│  ┌─────────────────┐                          │
│  │ Test Process    │ (APK 形式安装)             │
│  │  (Test APK)     │                          │
│  └────────┬────────┘                          │
│           │ AndroidJUnitRunner                │
│           ├─ Instrumentation API              │
│           │                                   │
│  ┌────────▼───────────────┐                   │
│  │ Target App Process     │ (应用 APK)         │
│  │ ┌──────────────────┐   │                   │
│  │ │ Main Thread      │   │                   │
│  │ │ (UI Thread)      │   │                   │
│  │ │ Looper/Handler   │   │                   │
│  │ └────────┬─────────┘   │                   │
│  │          │              │                   │
│  │ ┌────────▼────────────┐ │                   │
│  │ │ Espresso Adapter    │ │                   │
│  │ │ - IdlingRegistry    │ │ 同步机制           │
│  │ │ - Idle check        │ │                   │
│  │ └─────────────────────┘ │                   │
│  │                         │                   │
│  │ ┌─────────────────────┐ │                   │
│  │ │ View Hierarchy      │ │                   │
│  │ │ (ViewMatchers)      │ │                   │
│  │ └─────────────────────┘ │                   │
│  └─────────────────────────┘                   │
│           ▲                                    │
│           │ Binder IPC                         │
│           │                                    │
└───────────┼────────────────────────────────────┘
            │
      ┌─────▼─────┐
      │ Test Suite│
      │ (同步等待)  │
      └───────────┘
```

**核心机制**：

```kotlin
// Espresso 的同步原理
object Espresso {
    private val idlingRegistry = IdlingRegistry.getInstance()
    
    fun onView(matcher: Matcher<View>): ViewInteraction {
        // 1. 等待 UI 空闲 (Idle)
        waitForUIIdle()  // 阻塞直到主线程空闲
        
        // 2. 在主线程上执行 View 查找
        val view = findViewOnMainThread(matcher)
        
        // 3. 返回交互对象
        return ViewInteraction(view)
    }
    
    private fun waitForUIIdle() {
        // 检查条件：
        // - 主线程消息队列为空
        // - 没有待处理的异步任务 (AsyncTask/Handler)
        // - 所有 IdlingResource 都处于空闲状态
        
        while (!isMainThreadIdle()) {
            Thread.sleep(100)  // 轮询检查
        }
    }
}

// 自定义 IdlingResource (模拟网络请求)
class NetworkIdlingResource : IdlingResource {
    private var isIdle = true
    private var callback: IdlingResource.ResourceCallback? = null
    
    override fun getName(): String = "NetworkIdlingResource"
    
    override fun isIdleNow(): Boolean = isIdle
    
    override fun registerIdleTransitionCallback(callback: IdlingResource.ResourceCallback) {
        this.callback = callback
    }
    
    fun setNetworkCallActive() {
        isIdle = false
    }
    
    fun setNetworkCallInactive() {
        isIdle = true
        callback?.onTransitionToIdle()  // 通知 Espresso 线程已空闲
    }
}

// 在测试中注册
@Before
fun setUp() {
    val networkIdling = NetworkIdlingResource()
    IdlingRegistry.getInstance().register(networkIdling)
}
```

### 3.4 完整的综合测试示例

```kotlin
@RunWith(AndroidJUnit4::class)
class UserProfileActivityTest {
    
    @get:Rule
    val activityRule = ActivityScenarioRule(UserProfileActivity::class.java)
    
    @get:Rule
    val permissionRule = GrantPermissionRule.grant(Manifest.permission.CAMERA)
    
    private val networkIdling = NetworkIdlingResource()
    
    @Before
    fun setUp() {
        IdlingRegistry.getInstance().register(networkIdling)
    }
    
    @After
    fun tearDown() {
        IdlingRegistry.getInstance().unregister(networkIdling)
    }
    
    @Test
    fun testCompleteUserProfileFlow() {
        // 1. 等待数据加载
        onView(withId(R.id.loading_spinner))
            .check(matches(not(isDisplayed())))
        
        // 2. 验证用户信息显示
        onView(withId(R.id.user_name))
            .check(matches(withText("张三")))
        
        // 3. 点击编辑按钮
        onView(withId(R.id.edit_btn))
            .perform(click())
        
        // 4. 修改昵称
        onView(withId(R.id.nickname_input))
            .perform(clearText(), typeText("我是张三"))
        
        // 5. 点击保存
        onView(withId(R.id.save_btn))
            .perform(click())
        
        // 6. 验证成功提示
        onView(withText("保存成功"))
            .inRoot(RootMatchers.withDecorView(
                not(activityRule.scenario.onActivity { it.window.decorView })
            ))
            .check(matches(isDisplayed()))
    }
}
```

---

## 四、关键对比

|特性|JUnit|Espresso|
|---|---|---|
|**测试粒度**|方法/类|完整交互流程|
|**被测对象**|业务逻辑、工具类|UI 组件、用户操作|
|**执行速度**|毫秒级|秒级|
|**依赖注入**|Mockito/Dagger|Hilt (推荐)|
|**并发问题**|不涉及|需要处理主线程同步|
|**覆盖率**|代码覆盖率|功能覆盖率|

---

## 五、最佳实践

1. **测试金字塔**：单元测试 70% → 集成测试 20% → UI 测试 10%
2. **Page Object Pattern**（UI 测试）
3. **每个测试一个主题** (单一职责)
4. **避免 Thread.sleep()** → 使用 IdlingResource
5. **使用 Fakes 替代 Mocks** (当 Mock 太复杂时)

这个知识点在智能座舱面试中经常被问到，尤其是涉及**语音交互测试** (Espresso + 语音识别 Mock) 和**CarPropertyManager 集成测试**的场景。