下面给你整理一套**UC / 字节 / 腾讯 Android 高级工程师高频面试题（带标准答案 + 面试得分点）**，基本覆盖：

> 🔥 Android系统原理 + Framework + 性能 + 架构 + SDK + 商业化（这些公司重点方向）

---

# 🧠 一、Android 系统原理（必考核心）

---

## ❓1. Activity 启动流程？

### ✅ 标准答案：

Activity 启动分为：

1. 触发 startActivity
    
2. AMS（ActivityManagerService）接管
    
3. ActivityTaskManager 处理任务栈
    
4. Zygote fork 新进程（如果需要）
    
5. ActivityThread 启动主线程
    
6. Handler 发送 LAUNCH_ACTIVITY
    
7. onCreate → onStart → onResume
    

---

### 🔥 面试加分点：

- AMS + ATMS 分工
    
- Binder 跨进程调用
    
- LaunchMode 影响 Task 栈
    

---

## ❓2. Binder 原理？

### ✅ 标准答案：

Binder 是 Android IPC 机制：

- Client 调用 Proxy
    
- 通过 Binder 驱动（内核）
    
- Server Stub 接收
    
- Parcel 序列化数据传输
    

---

### 🔥 本质：

> Linux 驱动 + mmap + 用户态/内核态通信

---

### 🚀 加分点：

- 一次拷贝（mmap）
    
- 比 socket 更高效
    
- 四大组件底层都依赖 Binder
    

---

## ❓3. AMS / WMS / PMS 分别是什么？

### AMS（Activity Manager Service）

- 管 Activity 生命周期
    
- 任务栈管理
    

### WMS（Window Manager Service）

- 控制 View 显示
    
- Window 添加/绘制
    

### PMS（Package Manager Service）

- APK 安装解析
    
- manifest 管理
    

---

# ⚡ 二、Handler / 线程模型（字节必问）

---

## ❓4. Handler 机制原理？

### ✅ 标准答案：

- Looper：死循环取消息
    
- MessageQueue：消息队列
    
- Handler：发送消息
    
- ThreadLocal：绑定线程
    

---

流程：

```
Handler.sendMessage()
→ MessageQueue.enqueueMessage()
→ Looper.loop()
→ Handler.dispatchMessage()
```

---

### 🔥 加分点：

- epoll + Linux pipe 唤醒
    
- 无消息时阻塞不耗 CPU
    

---

## ❓5. 为什么主线程不会卡死？

因为：

- Looper.loop 是阻塞但事件驱动
    
- 有消息才执行
    
- 无消息 epoll 等待
    

---

# ⚡ 三、性能优化（腾讯/字节重点）

---

## ❓6. ANR 产生原因？

### ✅ 标准答案：

ANR = 主线程超时：

- Input事件：5秒无响应
    
- Broadcast：10秒
    
- Service：20秒
    

---

### 常见原因：

- 主线程 IO
    
- 锁竞争
    
- 死循环
    
- Binder阻塞
    

---

### 🔥 优化方案：

- 异步化（Coroutine / Thread）
    
- StrictMode 检测 IO
    
- TraceView / Systrace 分析
    

---

## ❓7. 内存泄漏怎么排查？

### 工具：

- LeakCanary
    
- MAT
    

---

### 常见原因：

- Handler 内部类
    
- 静态引用 Context
    
- 单例持有 Activity
    
- 线程未释放
    

---

### 加分点：

> GC Root 链分析

---

# ⚡ 四、View / 绘制机制（腾讯常问）

---

## ❓8. View 绘制流程？

```
Activity → ViewRootImpl
→ performTraversals()
→ measure()
→ layout()
→ draw()
```

---

### 三大流程：

- measure：测量大小
    
- layout：确定位置
    
- draw：绘制
    

---

### 加分点：

- invalidate vs requestLayout
    
- Choreographer + VSYNC
    

---

## ❓9. 为什么滑动不卡？

因为：

- GPU + SurfaceFlinger
    
- 16ms刷新机制
    
- Choreographer同步帧
    

---

# ⚡ 五、网络（字节/腾讯必问）

---

## ❓10. OkHttp 原理？

### 核心链路：

```
Request → Interceptor Chain → Cache → Connection → Server
```

---

### 关键点：

- 责任链模式
    
- 连接复用（ConnectionPool）
    
- HTTP2多路复用
    

---

## ❓11. Retrofit 原理？

- 动态代理（Proxy）
    
- Method 注解解析
    
- OkHttp 执行请求
    

---

# ⚡ 六、架构设计（UC重点）

---

## ❓12. MVP / MVVM 区别？

|模式|特点|
|---|---|
|MVC|View混乱|
|MVP|Presenter解耦|
|MVVM|DataBinding + 响应式|

---

## ❓13. 为什么用 MVVM？

- 数据驱动UI
    
- 生命周期安全
    
- LiveData 自动更新
    

---

# ⚡ 七、Gradle / 打包（UC浏览器必考）

---

## ❓14. APK 打包流程？

```
AIDL → AAPT2 → R.java → Kotlin/Java编译 → D8 → R8 → APK
```

---

### 加分点：

- class.dex 生成
    
- multidex
    
- shrinkResources
    

---

## ❓15. R8 / Proguard 区别？

- Proguard：旧混淆工具
    
- R8：Google新一代（合并D8）
    

---

# ⚡ 八、商业化 / SDK（UC/字节广告岗重点🔥）

---

## ❓16. 广告SDK流程？

```
请求广告 → 竞价 → 返回广告 → 展示 → 点击 → 转化回传
```

---

### 关键点：

- 曝光率统计
    
- eCPM
    
- waterfall / bidding
    

---

## ❓17. SDK如何防崩溃？

- try-catch 全链路
    
- 宿主隔离
    
- 异常上报
    
- 超时控制
    

---

# ⚡ 九、高阶加分（字节/腾讯区分点）

---

## ❓18. 插件化原理？

- ClassLoader 加载外部 dex
    
- Hook AMS / PMS
    
- 代理 Activity
    

---

## ❓19. 多进程通信？

- Binder（主流）
    
- AIDL
    
- Messenger
    

---

## ❓20. 为什么 Android 用 Binder 而不是 socket？

因为：

- 更快（零拷贝）
    
- 安全（UID校验）
    
- 系统级优化
    

---

# 🚀 十、面试官最喜欢的“终极问题”

---

## ❓21. 你怎么设计一个广告SDK？

### 标准答法：

1. 请求模块（网络层）
    
2. 策略模块（广告排序）
    
3. 渲染模块（View容器）
    
4. 回调系统（曝光/点击）
    
5. 数据埋点系统
    
6. 宿主隔离设计
    

---

# 🎯 如果你要冲 UC / 字节 / 腾讯（关键建议）

必须重点准备：

🔥 Binder / AMS / WMS  
🔥 Handler / Looper  
🔥 Gradle 打包  
🔥 性能优化  
🔥 广告SDK设计（最加分）

---

# 📌 如果你想继续冲高薪（我可以帮你）

我可以给你下一步：

### 🚀 1. 字节/腾讯 Android 面试 100题（进阶版）

### 🚀 2. 广告SDK完整架构（可写进简历）

### 🚀 3. Android Framework源码级解析

### 🚀 4. 面试突击（7天上岸计划）

直接说一句：

> 我要冲字节/腾讯Android面试

我可以帮你做一套**高命中率冲刺方案**。