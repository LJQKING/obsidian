
## KMP 是什么 & 能干什么

**Kotlin Multiplatform** 让你用 Kotlin 编写一次代码，编译到多个平台执行（Android、iOS、JVM、JS、Native 等）。核心目标是**共享业务逻辑代码**，减少重复开发。

### 适用场景

- **共享业务逻辑**：数据处理、网络请求、加密算法、状态管理
- **减少代码重复**：iOS 和 Android 的相同业务逻辑
- **跨平台库开发**：发布给其他项目使用的基础库

---

## 组成部分

```
shared/                           // KMP 模块
├── src/
│   ├── commonMain/              // ✅ 通用代码（所有平台）
│   │   └── kotlin/
│   │       └── com/example/
│   │           ├── data/       // 数据层
│   │           ├── domain/     // 业务逻辑
│   │           └── Network.kt  // 跨平台网络类
│   │
│   ├── androidMain/             // ✅ 仅 Android 平台
│   │   └── kotlin/
│   │       └── com/example/
│   │           └── PlatformConfig.kt
│   │
│   ├── iosMain/                 // ✅ 仅 iOS 平台
│   │   └── kotlin/
│   │       └── com/example/
│   │           └── PlatformConfig.kt
│   │
│   └── commonTest/              // 通用单元测试
│
app/                              // Android 应用（依赖 shared）
└── ...

iosApp/                           // Xcode 项目（依赖 shared）
└── ...
```

### 核心概念

|概念|说明|
|---|---|
|**expect**|在 commonMain 中声明接口/函数（不实现）|
|**actual**|在 androidMain/iosMain 中提供具体实现|
|**commonMain**|跨平台通用代码（编译到所有目标平台）|
|**platform-specific**|androidMain/iosMain（仅在特定平台编译）|

---

## 工作原理

```
┌─────────────────────────────────────────────────────────────┐
│ commonMain: expect 类/函数（抽象定义）                      │
└─────────────────────────────────────────────────────────────┘
                    ↙              ↘
        ┌──────────────────┐   ┌──────────────────┐
        │  androidMain     │   │   iosMain        │
        │  actual 实现     │   │   actual 实现     │
        │  (Java/Android   │   │   (NSFoundation  │
        │   APIs)          │   │   APIs)          │
        └──────────────────┘   └──────────────────┘
              ↓                        ↓
        Kotlin/JVM               Kotlin/Native
        字节码编译               LLVM 编译
              ↓                        ↓
        Android APK              iOS Framework
```

**编译流程**：

1. **Gradle 识别** commonMain、androidMain、iosMain 的源码
2. **条件编译** 根据目标平台选择相应的 actual 实现
3. **平台编译** androidMain 编译为 Java 字节码、iosMain 编译为 Native 代码
4. **链接依赖** Android App 和 iOS App 分别链接编译后的库

---

## 完整示例：跨平台日志系统

### 1️⃣ **commonMain** - 通用接口

```kotlin
// shared/src/commonMain/kotlin/Logger.kt
expect class Logger {
    fun debug(message: String)
    fun error(message: String, throwable: Throwable? = null)
}

expect fun createLogger(tag: String): Logger
```

### 2️⃣ **androidMain** - Android 实现

```kotlin
// shared/src/androidMain/kotlin/Logger.kt
import android.util.Log

actual class Logger {
    private val tag = "MyApp"
    
    actual fun debug(message: String) {
        Log.d(tag, message)
    }
    
    actual fun error(message: String, throwable: Throwable?) {
        Log.e(tag, message, throwable)
    }
}

actual fun createLogger(tag: String): Logger = Logger()
```

### 3️⃣ **iosMain** - iOS 实现

```kotlin
// shared/src/iosMain/kotlin/Logger.kt
import platform.Foundation.NSLog

actual class Logger {
    actual fun debug(message: String) {
        NSLog("[DEBUG] %@", message)
    }
    
    actual fun error(message: String, throwable: Throwable?) {
        NSLog("[ERROR] %@", message)
    }
}

actual fun createLogger(tag: String): Logger = Logger()
```

### 4️⃣ **业务代码** - 跨平台使用（commonMain）

```kotlin
// shared/src/commonMain/kotlin/UserRepository.kt
class UserRepository {
    private val logger = createLogger("UserRepository")
    
    suspend fun fetchUser(id: String): User {
        logger.debug("开始获取用户 $id")
        
        return try {
            val response = HttpClient.get("/users/$id")
            logger.debug("获取成功")
            response.parseAs<User>()
        } catch (e: Exception) {
            logger.error("获取用户失败", e)
            throw e
        }
    }
}
```

### 5️⃣ **Android App 中使用**

```kotlin
// app/src/main/kotlin/MainActivity.kt
import com.example.UserRepository

class MainActivity : AppCompatActivity() {
    private val userRepo = UserRepository()
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        lifecycleScope.launch {
            try {
                val user = userRepo.fetchUser("123")
                // UI 更新
            } catch (e: Exception) {
                // 错误处理
            }
        }
    }
}
```

### 6️⃣ **iOS App 中使用**

```swift
// iosApp/ViewController.swift
import shared

class ViewController: UIViewController {
    let userRepo = UserRepository()
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        userRepo.fetchUser(id: "123") { user, error in
            if let user = user {
                // UI 更新
            } else if let error = error {
                print("错误: \(error)")
            }
        }
    }
}
```

---

## 更复杂的示例：平台相关的数据存储

```kotlin
// commonMain - 定义接口
expect class PlatformStorage {
    fun save(key: String, value: String)
    fun read(key: String): String?
}

// androidMain - SharedPreferences 实现
actual class PlatformStorage(context: Context) {
    private val prefs = context.getSharedPreferences("app", Context.MODE_PRIVATE)
    
    actual fun save(key: String, value: String) {
        prefs.edit().putString(key, value).apply()
    }
    
    actual fun read(key: String): String? = prefs.getString(key, null)
}

// iosMain - UserDefaults 实现
actual class PlatformStorage {
    private val defaults = NSUserDefaults.standardUserDefaults
    
    actual fun save(key: String, value: String) {
        defaults.setObject(value, forKey = key)
    }
    
    actual fun read(key: String): String? {
        return defaults.objectForKey(key) as? String
    }
}
```

---

## 核心优势

✅ **代码复用**：业务逻辑一份代码  
✅ **类型安全**：Kotlin 编译时检查，避免 Java/ObjC 的类型错误  
✅ **维护成本低**：Bug 修复一次生效两个平台  
✅ **渐进式采用**：可以从简单业务逻辑开始

## 常见限制

❌ 无法访问平台特定 API（需要 actual 实现）  
❌ 学习曲线（expect/actual 机制、多平台构建）  
❌ 编译时间较长（需要编译多套目标代码）  
❌ iOS 开发需要 Mac 环境

---

这套模式在 Android 面试中经常被问到，特别是在大厂做跨端开发时。你理解了吗？要不要我给你讲讲 KMP 中的**Coroutine 跨平台** 或 **网络层设计** 的更深入例子？