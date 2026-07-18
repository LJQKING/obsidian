### Kotlin 协程面试题 100 问（含答案）

覆盖基础概念、源码原理、结构化并发、Flow、Android 实战、性能优化与高级话题，适合 Android 高级工程师面试复习。

### 一、基础概念（1~20）

1. 什么是协程？

协程是一种轻量级并发编程模型，可以在少量线程上运行大量任务，通过挂起与恢复实现异步编程。

2. 协程和线程有什么区别？

线程由操作系统调度，创建和切换成本较高；协程由 Kotlin 运行时调度，切换发生在用户态，成本更低。

3. 为什么协程比线程更轻量？

协程挂起时不会阻塞线程，切换无需内核态上下文切换，因此资源消耗更小。

4. 什么是 suspend 函数？

使用 suspend 修饰的函数，可以在执行过程中挂起而不阻塞线程，并在之后恢复执行。

5. suspend 是否意味着异步？

不是。suspend 只表示函数可挂起，是否异步取决于调用它的协程和调度器。

6. 什么是挂起（suspension）？

挂起是协程暂停执行并保存当前状态的过程，线程可以去执行其他任务。

7. 什么是恢复（resume）？

恢复是协程从挂起点继续执行的过程，由 Continuation 负责。

8. 什么是 Continuation？

Continuation（续体）保存协程的执行状态和上下文，并提供恢复协程的方法。

9. 什么是 CoroutineContext？

CoroutineContext 是协程上下文，包含 Job、Dispatcher、CoroutineName、异常处理器等元素。

10. CoroutineContext 的本质是什么？

它本质上是一个键值集合（类似 Map），用于存储和组合协程相关信息。

11. 什么是 CoroutineScope？

CoroutineScope 定义协程的生命周期范围，所有在该作用域内启动的协程都受其管理。

12. 为什么要使用 CoroutineScope？

它可以统一管理协程的启动、取消和异常，避免协程泄漏。

13. 什么是 Job？

Job 表示一个协程任务，负责管理协程的生命周期和取消状态。

14. Job 的生命周期有哪些状态？

典型状态包括 New、Active、Completing、Completed、Cancelling 和 Cancelled。

15. 什么是 Dispatcher？

Dispatcher 决定协程在哪个线程或线程池上执行。

16. Dispatchers.Main 用于什么场景？

用于 UI 线程任务，例如更新界面。

17. Dispatchers.IO 用于什么场景？

用于网络、数据库、文件等 IO 密集型任务。

18. Dispatchers.Default 用于什么场景？

用于计算、排序、图像处理等 CPU 密集型任务。

19. Dispatchers.Unconfined 有什么特点？

它不限制协程执行线程，恢复时可能在不同线程执行，一般不推荐在业务代码中使用。

20. 什么是 CoroutineName？

CoroutineName 是上下文元素，用于给协程命名，便于日志和调试。

### 二、协程构建与启动（21~35）

21. launch 和 async 有什么区别？

launch 返回 Job，适合无返回值任务；async 返回 Deferred，适合需要结果的任务。

22. Deferred 是什么？

Deferred 是带结果的 Job，可以通过 await() 获取结果。

23. await() 的作用是什么？

等待 Deferred 完成并返回结果，如果协程失败则抛出异常。

24. withContext() 的作用是什么？

切换协程上下文并执行代码块，常用于线程切换。

25. withContext 和 launch 的主要区别是什么？

withContext 会挂起当前协程并返回结果；launch 会启动一个新的子协程并立即返回 Job。

26. 什么是 CoroutineStart？

CoroutineStart 定义协程的启动方式，如 DEFAULT、LAZY、ATOMIC、UNDISPATCHED。

27. LAZY 模式有什么特点？

协程不会立即启动，只有调用 start() 或 await() 时才开始执行。

28. ATOMIC 模式有什么特点？

协程启动后，在首次挂起之前不会被取消。

29. UNDISPATCHED 模式有什么特点？

协程会立即在当前线程执行，直到遇到第一个挂起点。

30. runBlocking 的作用是什么？

它会阻塞当前线程，直到协程及其子协程执行完成，常用于测试或 main 函数。

31. 为什么不建议在 Android 主线程使用 runBlocking？

因为它会阻塞主线程，可能导致界面卡顿甚至 ANR。

32. coroutineScope 和 supervisorScope 的区别是什么？

coroutineScope 中一个子协程失败会取消其他子协程；supervisorScope 中子协程失败不会影响兄弟协程。

33. 什么是结构化并发？

结构化并发要求协程有明确的父子关系，子协程生命周期受父协程管理，避免“野协程”。

34. 为什么不推荐 GlobalScope？

GlobalScope 的协程不受生命周期管理，容易造成内存泄漏和任务失控。

35. Android 中常用的协程作用域有哪些？

常用有 lifecycleScope 和 viewModelScope，分别与 Lifecycle 和 ViewModel 生命周期绑定。

### 三、取消与异常（36~50）

36. 如何取消协程？

通过调用 Job.cancel() 或取消其所在作用域。

37. 协程取消是如何传播的？

取消会沿父子关系向下传播，父协程取消时，所有子协程也会被取消。

38. 什么是协作式取消？

协程不会被强制中断，而是在挂起点或主动检查 isActive 时响应取消。

39. 哪些挂起函数支持取消？

delay、withContext、yield 等大多数 kotlinx.coroutines 提供的挂起函数都支持取消。

40. NonCancellable 的作用是什么？

在取消状态下仍允许执行关键清理逻辑，例如关闭资源或保存数据。

41. 什么是 CancellationException？

它是协程取消时使用的异常，通常不应当作为错误上报。

42. launch 中的异常如何处理？

未捕获异常会传播到父协程，并最终由 CoroutineExceptionHandler 处理。

43. async 中的异常如何处理？

异常会封装在 Deferred 中，调用 await() 时才会抛出。

44. CoroutineExceptionHandler 有什么作用？

它用于处理根协程中未捕获的异常，但不能捕获 async 中未 await 的异常。

45. SupervisorJob 的作用是什么？

SupervisorJob 允许子协程独立失败，不会因为一个子协程异常而取消整个作用域。

46. supervisorScope 与 SupervisorJob 的关系是什么？

supervisorScope 会创建一个具有 SupervisorJob 行为的作用域。

47. 为什么子协程异常会默认取消父协程？

这是结构化并发的设计，确保错误不会被静默忽略。

48. 如何在协程中安全地处理异常？

在合适的位置使用 try/catch，或使用 supervisorScope 隔离失败。

49. 什么是异常聚合？

多个子协程同时失败时，协程框架会保留主要异常，并将其他异常作为 suppressed exceptions 附加。

50. 为什么不建议用 try/catch 包裹整个作用域？

因为子协程异常可能在不同时间发生，更好的做法是在任务边界处理异常并设计合理的监督结构。

### 四、Flow（51~70）

51. 什么是 Flow？

Flow 是 Kotlin 协程提供的冷流数据流，用于异步地发送多个值。

52. 为什么 Flow 是冷流？

只有在 collect 时，Flow 的代码才会真正执行。

53. Flow 和 Sequence 有什么区别？

Sequence 是同步的，而 Flow 支持挂起和异步执行。

54. Flow 和 LiveData 有什么区别？

Flow 更通用、支持背压和丰富操作符；LiveData 与 Android 生命周期深度集成但功能相对有限。

55. collect() 的作用是什么？

collect 用于启动 Flow 并接收其发射的值。

56. flowOn() 的作用是什么？

改变上游 Flow 的执行上下文，而不会影响下游收集上下文。

57. map() 操作符有什么作用？

对 Flow 发射的每个元素进行转换。

58. filter() 操作符有什么作用？

根据条件过滤元素。

59. take() 操作符有什么作用？

只获取前 N 个元素，然后取消 Flow 收集。

60. catch() 操作符有什么作用？

捕获上游 Flow 中发生的异常。

61. onCompletion() 的作用是什么？

在 Flow 完成或取消时执行收尾逻辑。

62. 什么是 SharedFlow？

SharedFlow 是热流，多个收集者共享同一数据源。

63. 什么是 StateFlow？

StateFlow 是一种特殊的 SharedFlow，始终持有最新状态值。

64. StateFlow 与 LiveData 的主要区别是什么？

StateFlow 与协程生态更紧密，支持背压和丰富操作符；LiveData 自动感知生命周期。

65. SharedFlow 与 StateFlow 的主要区别是什么？

StateFlow 必须有初始值并始终保存最新值；SharedFlow 可以不保存状态，适合事件流。

66. 什么是背压（Backpressure）？

生产速度超过消费速度时，系统需要限制、缓冲或丢弃数据的机制。

67. Flow 如何处理背压？

通过 buffer、conflate、collectLatest 等操作符来缓冲、合并或取消旧任务。

68. collectLatest() 的作用是什么？

当新的值到来时，取消前一个尚未完成的处理逻辑。

69. zip() 和 combine() 有什么区别？

zip 按位置配对元素；combine 会在任一 Flow 发射新值时，使用双方最新值组合结果。

70. flatMapConcat、flatMapMerge、flatMapLatest 有什么区别？

Concat 串行展开；Merge 并发合并；Latest 会取消旧 Flow，只保留最新 Flow。

### 五、源码原理（71~85）

71. suspend 函数编译后会发生什么？

编译器会将其转换为 CPS（Continuation Passing Style）形式，并额外添加一个 Continuation 参数。

72. 什么是协程状态机？

编译器会把 suspend 函数拆分为多个状态，并生成状态机以支持挂起和恢复。

73. Continuation 在底层的作用是什么？

保存当前状态、局部变量和上下文，并在恢复时继续执行。

74. launch 底层创建了什么对象？

它会创建一个继承自 AbstractCoroutine 的协程对象，并关联 Job 和 Continuation。

75. Job 为什么也是 CoroutineContext.Element？

因为 Job 需要存储在协程上下文中，方便统一管理生命周期和取消。

76. CoroutineDispatcher 的核心方法是什么？

dispatch(context, block)，用于将任务投递到对应线程或线程池。

77. Dispatchers.Main 在 Android 中如何实现？

底层基于 Handler，通过 Handler.post() 将任务发送到主线程消息队列。

78. Dispatchers.IO 与 Dispatchers.Default 的关系是什么？

它们共享底层调度器，但 IO 调度器允许更多线程，以适应阻塞型任务。

79. 什么是 CoroutineScheduler？

CoroutineScheduler 是 kotlinx.coroutines 的核心线程池实现，支持工作窃取（work-stealing）。

80. 什么是工作窃取（Work Stealing）？

空闲线程可以从其他线程的任务队列中窃取任务，从而提高 CPU 利用率。

81. delay 为什么不会阻塞线程？

delay 使用 suspendCancellableCoroutine 挂起协程，并通过定时器在时间到达后恢复协程，线程期间可以执行其他任务。

82. yield() 的作用是什么？

主动让出线程执行权，使调度器有机会运行其他协程。

83. 协程切换线程的成本为什么低？

因为协程切换主要是保存和恢复状态，不需要进行昂贵的操作系统线程上下文切换。

84. Kotlin 协程是否是语言级特性？

suspend 与 Continuation 是语言级支持，而调度器、Job、Flow 等由 kotlinx.coroutines 库实现。

85. 协程的核心实现可以概括为什么？

“状态机 + Continuation + 调度器”共同构成了协程的底层机制。

### 六、Android 实战与性能（86~100）

86. viewModelScope 的作用是什么？

与 ViewModel 生命周期绑定，在 ViewModel 清除时自动取消所有协程。

87. lifecycleScope 的作用是什么？

与 LifecycleOwner 生命周期绑定，在生命周期结束时自动取消协程。

88. repeatOnLifecycle() 有什么用途？

在指定生命周期状态下启动协程，在状态离开时自动取消并在重新进入时重启。

89. 为什么推荐使用 repeatOnLifecycle 收集 Flow？

它能避免界面不可见时仍持续收集数据，从而节省资源并防止泄漏。

90. 如何在 Android 中进行线程切换？

使用 withContext(Dispatchers.IO) 执行后台任务，再返回主线程更新 UI。

91. 什么情况下应该使用 async 并发？

当多个独立任务可以同时执行并且需要组合结果时，例如并发请求多个接口。

92. 如何限制协程并发数量？

可以使用 Semaphore、Channel，或配置自定义 Dispatcher 的并行度。

93. 协程会导致内存泄漏吗？

会。如果协程持有长生命周期对象或使用 GlobalScope 且未正确取消，就可能造成泄漏。

94. 如何避免协程泄漏？

使用生命周期感知的作用域，并及时取消不再需要的协程。

95. 协程适合 CPU 密集型任务吗？

适合，但应使用 Dispatchers.Default，避免在 Main 或 IO 上执行大量计算。

96. 协程适合阻塞式 IO 吗？

可以，但应放在 Dispatchers.IO 上执行，以避免阻塞主线程和计算线程。

97. 如何在协程中实现超时控制？

使用 withTimeout 或 withTimeoutOrNull 为任务设置最大执行时间。

98. 协程与 RxJava 的主要区别是什么？

协程语法更接近同步代码，学习成本较低；RxJava 在复杂流式组合和生态方面更成熟。

99. Kotlin 协程是否能替代所有异步方案？

大多数 Android 异步场景都可以使用协程，但仍需根据项目生态和需求选择合适方案。

100. 面试中一句话总结协程原理？

Kotlin 协程通过编译器生成的状态机与 Continuation 保存执行现场，再由 Dispatcher 调度到线程执行，从而以低成本实现挂起与恢复。

复习建议

先掌握 CoroutineScope、Job、Dispatcher、suspend、launch/async、withContext 和取消机制，再深入 Flow、StateFlow、SharedFlow 与源码实现。面试中重点准备“为什么协程轻量”“suspend 底层原理”“结构化并发”“Flow 与 LiveData 区别”等高频问题。