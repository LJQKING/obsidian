Android Coroutine 本质上就是 **Kotlin 协程在 Android 场景下的应用**。它不是“开一个线程”，而是把“异步任务”写成像同步代码一样，方便做网络请求、数据库读写、图片加载、定时任务、并发组合等。

你可以把它理解成：

- **线程**：负责真正执行代码
    
- **协程**：负责“挂起、恢复、调度”任务
    
- **Dispatcher**：决定代码跑在哪个线程池/线程上
    

---

## 1）Coroutine 是什么

协程最大的特点是：

1. **挂起不会阻塞线程**
    
2. **恢复时继续执行原来的逻辑**
    
3. **代码写法接近同步**
    
4. **天然支持取消**
    
5. **适合 Android 的生命周期场景**
    

比如网络请求：

```kotlin
viewModelScope.launch {
    val user = withContext(Dispatchers.IO) {
        api.getUser()
    }
    textView.text = user.name
}
```

看起来像同步写法，但实际上 `withContext(Dispatchers.IO)` 里的代码会切到 IO 线程执行，执行完再切回主线程。

---

## 2）代码原理是什么

协程的核心原理可以分成 4 点：

### 1. `suspend` 关键字

`suspend` 表示这个函数**可以被挂起**，挂起时不会阻塞线程。

例如：

```kotlin
suspend fun loadData(): String {
    delay(1000)
    return "Hello"
}
```

`delay()` 并不是睡死线程，而是把当前协程挂起，等时间到了再恢复。

---

### 2. 不是线程切换，而是“状态机”

Kotlin 编译器会把 `suspend` 函数编译成一种**状态机**。

你写的代码：

```kotlin
suspend fun test() {
    val a = getA()
    val b = getB()
    println(a + b)
}
```

编译器底层会拆成类似：

- 执行到 `getA()` 前保存状态
    
- 挂起
    
- 恢复后从保存的位置继续
    
- 再执行 `getB()`
    
- 再继续往下走
    

所以协程能“暂停后继续”，本质上靠的是**Continuation（续体）**保存上下文和执行位置。

---

### 3. `Continuation`

Continuation 可以理解为：

> “当前函数挂起后，之后要怎么继续执行”的对象

协程执行到挂起点时，会把“下一步该干什么”封装到 Continuation 里，等异步结果回来再恢复。

---

### 4. 调度器 Dispatcher

Dispatcher 决定协程运行在哪个线程：

- `Dispatchers.Main`：主线程
    
- `Dispatchers.IO`：IO 线程池
    
- `Dispatchers.Default`：CPU 密集型线程池
    

例如：

```kotlin
withContext(Dispatchers.IO) {
    // 数据库、网络、文件操作
}
```

---

## 3）Android 中为什么常用 Coroutine

Android 开发里协程特别适合这些场景：

- 网络请求
    
- Room 数据库
    
- 文件读写
    
- 定时轮询
    
- 多个请求并发执行
    
- 页面销毁时自动取消任务
    

相比 `Thread + Handler`，协程更简洁，也更容易做取消和异常管理。

---

## 4）怎么写：最常用的几种写法

### 4.1 在 Activity / Fragment 里使用

```kotlin
lifecycleScope.launch {
    val result = withContext(Dispatchers.IO) {
        api.getData()
    }
    binding.textView.text = result
}
```

`lifecycleScope` 会随着页面生命周期自动取消，避免内存泄漏。

---

### 4.2 在 ViewModel 里使用

```kotlin
class MainViewModel : ViewModel() {

    fun loadData() {
        viewModelScope.launch {
            val data = withContext(Dispatchers.IO) {
                repository.getData()
            }
            // 更新 UI 状态
        }
    }
}
```

`viewModelScope` 会在 ViewModel 销毁时自动取消。

---

### 4.3 网络请求写法

```kotlin
class UserRepository(private val api: UserApi) {

    suspend fun getUser(): User {
        return api.getUser()
    }
}
```

调用：

```kotlin
viewModelScope.launch {
    try {
        val user = repository.getUser()
        binding.name.text = user.name
    } catch (e: Exception) {
        binding.name.text = "加载失败"
    }
}
```

---

### 4.4 并发请求

```kotlin
viewModelScope.launch {
    val userDeferred = async(Dispatchers.IO) { api.getUser() }
    val orderDeferred = async(Dispatchers.IO) { api.getOrders() }

    val user = userDeferred.await()
    val orders = orderDeferred.await()

    // 合并结果
}
```

`async + await` 适合多个任务并发执行。

---

### 4.5 `withContext` 切线程

```kotlin
viewModelScope.launch {
    val result = withContext(Dispatchers.IO) {
        database.userDao().getUserById(1)
    }
    binding.textView.text = result.name
}
```

`withContext` 常用于**临时切线程**。

---

## 5）一个完整 Android 示例

### Repository

```kotlin
class UserRepository(private val api: UserApi) {
    suspend fun fetchUser(): User {
        return api.getUser()
    }
}
```

### ViewModel

```kotlin
class UserViewModel(private val repository: UserRepository) : ViewModel() {

    private val _userText = MutableLiveData<String>()
    val userText: LiveData<String> = _userText

    fun loadUser() {
        viewModelScope.launch {
            try {
                val user = withContext(Dispatchers.IO) {
                    repository.fetchUser()
                }
                _userText.value = user.name
            } catch (e: Exception) {
                _userText.value = "加载失败：${e.message}"
            }
        }
    }
}
```

### Activity

```kotlin
class MainActivity : AppCompatActivity() {

    private lateinit var viewModel: UserViewModel

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        viewModel = ViewModelProvider(this)[UserViewModel::class.java]

        viewModel.userText.observe(this) { text ->
            findViewById<TextView>(R.id.tvName).text = text
        }

        findViewById<Button>(R.id.btnLoad).setOnClickListener {
            viewModel.loadUser()
        }
    }
}
```

---

## 6）协程常用核心 API

### `launch`

启动一个协程，不返回结果。

```kotlin
lifecycleScope.launch {
    // 执行任务
}
```

### `async`

启动一个协程，返回结果，配合 `await()`。

```kotlin
val deferred = async { loadData() }
val data = deferred.await()
```

### `withContext`

切换协程上下文，常用于切线程。

```kotlin
withContext(Dispatchers.IO) {
    // IO 操作
}
```

### `delay`

挂起当前协程，不阻塞线程。

```kotlin
delay(1000)
```

### `Job`

协程任务句柄，可用于取消。

```kotlin
val job = launch { ... }
job.cancel()
```

---

## 7）协程和线程的区别

协程不是线程替代品，而是更轻量的异步方案。

|对比|线程|协程|
|---|---|---|
|开销|大|小|
|创建成本|高|低|
|是否阻塞|会|不一定|
|取消|麻烦|方便|
|写法|回调多|更接近同步|

---

## 8）Android 中协程的注意点

### 1. 不要在主线程做耗时操作

比如：

```kotlin
viewModelScope.launch {
    val data = withContext(Dispatchers.IO) { loadFromDb() }
}
```

### 2. 页面销毁要自动取消

优先使用：

- `lifecycleScope`
    
- `viewModelScope`
    

不要随便自己 `GlobalScope.launch {}`，它容易造成泄漏。

### 3. 异常要处理

```kotlin
viewModelScope.launch {
    try {
        val data = repository.load()
    } catch (e: Exception) {
        // 处理异常
    }
}
```

### 4. IO 和 CPU 任务分开

- 网络/数据库/文件：`Dispatchers.IO`
    
- 计算：`Dispatchers.Default`
    

---

## 9）一句话理解源码原理

协程本质上是：

> **编译器把 `suspend` 函数改造成状态机，借助 Continuation 保存执行位置，再由 Dispatcher 调度到合适线程继续执行。**

---

如果你愿意，我可以继续给你补一版：

1. **Kotlin Coroutine 源码级原理（Continuation、StateMachine、Dispatcher）**
    
2. **Android 协程面试题 50 问**
    
3. **Android 项目中 Coroutine + Flow + ViewModel 的标准写法**