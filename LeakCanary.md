
## 什么是 LeakCanary？

LeakCanary 是 Square 开源的 **内存泄漏检测库**，能在开发阶段自动检测 Java 堆内存泄漏。它的核心特点：

- **自动检测**：监控 Activity/Fragment 销毁后的内存回收
- **可视化**：直接在 App 中展示泄漏堆栈信息
- **低开销**：仅在开发/Debug 版本运行，Release 版本禁用

---

## 集成步骤

### 1️⃣ 添加依赖（build.gradle）

```gradle
dependencies {
    // LeakCanary 3.x (最新推荐)
    debugImplementation 'com.squareup.leakcanary:leakcanary-android:3.0-beta-1'
    
    // 或使用稳定版 2.x
    debugImplementation 'com.squareup.leakcanary:leakcanary-android:2.13'
}
```

**重点**：使用 `debugImplementation`，确保只在 Debug 构建中包含。

### 2️⃣ 初始化（自动/手动）

**方案A：自动初始化（推荐）**

LeakCanary 3.x 使用 ContentProvider 自动初始化，无需手动代码：

```kotlin
// AndroidManifest.xml 中会自动注册 provider，无需任何操作
```

**方案B：手动初始化**

如果需要自定义配置：

```kotlin
// 在 Application 中
import leakcanary.LeakCanary

class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()
        
        // 自动初始化（3.x）
        // LeakCanary 已通过 ContentProvider 启动
        
        // 或手动配置（2.x）
        LeakCanary.config = LeakCanary.config.copy(
            retainedVisibleThreshold = 5  // 检测到5个retained对象后开始分析
        )
    }
}
```

---

## 使用方法

### 基本工作流

```
Activity/Fragment 销毁
        ↓
LeakCanary 监控（等待GC）
        ↓
确认对象未回收（内存泄漏）
        ↓
获取堆转储（.hprof 文件）
        ↓
分析引用链并生成报告
```

### 常见用法

#### ✅ 检测 Activity 泄漏

```kotlin
class MainActivity : AppCompatActivity() {
    private lateinit var listener: MyListener
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // ❌ 错误：持有 Activity 引用（泄漏）
        listener = MyListener(this)
    }
}

class MyListener(val activity: MainActivity) {
    fun onEvent() {
        // 监听器可能被全局单例持有，导致 Activity 无法回收
    }
}
```

LeakCanary 会检测到这个泄漏并自动报告。

#### ✅ 检测 Fragment 泄漏

```kotlin
class MyFragment : Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        // ❌ 错误：在全局监听器中保存 Fragment
        GlobalEventBus.subscribe(fragment = this)  // 泄漏！
    }
}
```

#### ✅ 自定义监控对象

```kotlin
import leakcanary.ObjectWatcher

class MyViewModel : ViewModel() {
    private val watcher = LeakCanary.objectWatcher
    
    override fun onCleared() {
        super.onCleared()
        
        // 手动注册要监控的对象
        watcher.expectWeaklyReachable(this, "MyViewModel cleared but not GC'd")
    }
}
```

---

## 如何诊断内存问题

### 1️⃣ 查看泄漏通知

运行 App 后，LeakCanary 会在以下情况触发：

```
📌 监控指标
- Activity 泄漏个数达到阈值 → 自动触发分析
- 默认：保留 5 个 retained Activity 后分析
- 可在 logcat 中看到：LeakCanary - Heuristic result: ...
```

### 2️⃣ 查看泄漏报告

**通知栏** → 点击通知 → **LeakCanary 应用** → 查看报告

```
┌─────────────────────────────────────────┐
│ 🔴 Leaked: MainActivity                 │
│                                         │
│ ┌─────────────────────────────────────┐ │
│ │ Reference chain:                    │ │
│ │ MainActivity ⇐ retained              │ │
│ │     ↓                                │ │
│ │ MyListener (listener field)          │ │
│ │     ↓                                │ │
│ │ GlobalEventBus (static instance)     │ │
│ └─────────────────────────────────────┘ │
│                                         │
│ 💾 Retained: 24.5 MB                    │
└─────────────────────────────────────────┘
```

### 3️⃣ 解读引用链

从 LeakCanary 报告理解泄漏原因：

```
MainActivity ⇐ retained → 为什么持有了它？
    ↓
listener (字段名) → 是哪个字段？
    ↓
MyListener → 什么类持有它？
    ↓
GlobalEventBus → 最终的持有者（根因）
```

**解决方案**：在 MyListener 中改用 `WeakReference<Activity>` 或使用 `interface` 回调。

### 4️⃣ 查看堆转储

LeakCanary 自动将 `.hprof` 文件保存在：

```
/data/data/com.example.app/files/leakcanary-results/
```

也可导出后用 Android Studio 打开：

```
Android Studio 
  → Profiler 
  → Memory 
  → 导入 .hprof 文件
  → 分析 Retained Objects
```

---

## 常见内存泄漏场景

|泄漏原因|检测方式|解决方案|
|---|---|---|
|**静态引用 Activity**|LeakCanary 直接报告|改用 Context.getApplicationContext()|
|**监听器未注销**|显示 Listener → Activity|在 onDestroy() 中 unregister|
|**内部类持有外部类**|LeakCanary 追踪嵌套引用|改用 static 内部类或分离为独立类|
|**Handler 消息堆积**|显示 Message → Handler → Activity|在 onDestroy() 中 removeCallbacks()|
|**Timer/Thread 未停止**|自定义 ObjectWatcher 监控|在合适时机调用 cancel()/interrupt()|

---

## 配置高级选项

```kotlin
// build.gradle 或 ProGuard 规则
leakcanary {
    // 只在 debuggable App 中启用
    watcher_implicit_leak {
        enabled = true
    }
}
```

```kotlin
// 排除已知误报
LeakCanary.config = LeakCanary.config.copy(
    exclusions = setOf(
        // 排除 Android Framework 已知泄漏
        ExcludedRefs.ANDROIDX_ASYNC_TASK,
        // 自定义排除规则
        ExcludedRefs.Builder()
            .name("com.example.MyClass")
            .reason("Known issue #123")
            .build()
    )
)
```

---

## 最佳实践

✅ **DO**

- 在 Debug 版本中始终启用 LeakCanary
- 运行自动化测试时观察 LeakCanary 报告
- 定期检查 logcat 中的 LeakCanary 输出

❌ **DON'T**

- 在 Release 版本中包含 LeakCanary（使用 `debugImplementation`）
- 忽视 LeakCanary 的警告
- 仅依赖 LeakCanary（配合 Memory Profiler 更全面）

---

## 配合 Memory Profiler 使用

```
LeakCanary 的适用场景 → 已知对象是否泄漏
Memory Profiler 的适用场景 → 整体堆内存趋势、未知泄漏源

推荐流程：
1. LeakCanary 快速定位已知泄漏
2. Memory Profiler 做全局内存分析
3. Perfetto/Systrace 分析长期内存趋势
```

这套方法在面试中也是高频提问点 😊 你最近是否在实际项目中用过 LeakCanary 排查真实的泄漏？