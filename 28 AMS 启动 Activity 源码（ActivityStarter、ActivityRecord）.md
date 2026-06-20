这是 Android Framework 面试中出现频率最高的问题之一：

> **startActivity() 到底发生了什么？**

如果从源码角度分析，核心类就是：

```text
ActivityStarter
ActivityRecord
ActivityTaskManagerService(ATMS)
ActivityTaskSupervisor
RootWindowContainer
ActivityThread
```

---

# 一、整体流程图

```text
App进程
│
├── Activity.startActivity()
│
├── Instrumentation.execStartActivity()
│
├── ActivityTaskManager.getService()
│
└──── Binder ────► SystemServer

SystemServer
│
├── ActivityTaskManagerService
│
├── ActivityStartController
│
├── ActivityStarter
│
├── ActivityTaskSupervisor
│
├── RootWindowContainer
│
├── ActivityRecord
│
└── AMS启动目标进程

目标App进程
│
├── ActivityThread
│
├── performLaunchActivity()
│
└── Activity.onCreate()
```

---

# 二、startActivity源码入口

Activity.java

```java
public void startActivity(Intent intent) {
    startActivityForResult(intent, -1);
}
```

↓

```java
public void startActivityForResult(
        Intent intent,
        int requestCode) {

    Instrumentation.ActivityResult ar =
            mInstrumentation.execStartActivity(
                    this,
                    mMainThread.getApplicationThread(),
                    mToken,
                    this,
                    intent,
                    requestCode,
                    options);
}
```

---

# 三、Instrumentation

Instrumentation相当于：

```text
Activity启动代理
```

源码：

```java
public ActivityResult execStartActivity(
        Context who,
        IBinder contextThread,
        IBinder token,
        Activity target,
        Intent intent,
        int requestCode,
        Bundle options) {

    ActivityTaskManager.getService()
            .startActivity(...);
}
```

这里第一次跨进程。

获取：

```java
IActivityTaskManager
```

然后：

```java
Binder IPC
```

调用：

```java
ActivityTaskManagerService
```

---

# 四、进入SystemServer

### ActivityTaskManagerService

源码：

```java
frameworks/base/services/core/java/
com/android/server/wm/
ActivityTaskManagerService.java
```

接收：

```java
startActivity()
```

↓

```java
public int startActivityAsUser(...)
```

↓

```java
getActivityStartController()
```

↓

```java
obtainStarter()
```

↓

```java
execute()
```

---

# 五、ActivityStarter

核心类：

```java
ActivityStarter
```

源码：

```java
frameworks/base/services/core/java/
com/android/server/wm/
ActivityStarter.java
```

作用：

```text
Activity启动总调度器
```

---

执行：

```java
execute()
```

↓

```java
startActivityUnchecked()
```

↓

```java
startActivityInner()
```

---

# 六、创建ActivityRecord

AMS不会直接管理Activity对象。

而是创建：

```java
ActivityRecord
```

---

源码：

```java
final ActivityRecord r =
        new ActivityRecord(...);
```

---

ActivityRecord是什么？

类似：

```java
class ActivityRecord {

    Intent intent;

    ActivityInfo info;

    String packageName;

    IBinder token;

}
```

---

理解：

```text
ActivityRecord
=
Framework中的Activity身份证
```

一个Activity对应一个：

```java
ActivityRecord
```

例如：

```text
MainActivity
 ↓
ActivityRecord

LoginActivity
 ↓
ActivityRecord
```

---

# 七、寻找Task栈

Android必须决定：

```text
放到哪个Task？
```

例如：

```text
Task1

MainActivity
   ↓
DetailActivity
```

启动：

```java
startActivity(LoginActivity)
```

需要判断：

```text
新建Task？
还是压栈？
```

---

源码：

```java
Task task = getReusableTask();
```

↓

根据：

```text
launchMode

singleTask

singleTop

singleInstance
```

决定。

---

# 八、ActivityTaskSupervisor

类：

```java
ActivityTaskSupervisor
```

作用：

```text
管理Activity栈
```

源码：

```java
startSpecificActivity()
```

---

判断：

```java
目标进程存在？
```

---

如果存在：

```java
realStartActivityLocked()
```

---

如果不存在：

```java
AMS启动进程
```

---

# 九、AMS启动应用进程

AMS：

```java
startProcessAsync()
```

↓

```java
ProcessList.startProcessLocked()
```

↓

```java
ZygoteProcess.start()
```

↓

```java
fork()
```

↓

新App进程产生

```text
Zygote
   ↓
fork
   ↓
App进程
```

---

# 十、ActivityThread接收启动消息

新进程启动：

```java
ActivityThread.main()
```

创建：

```java
Looper.prepareMainLooper();
```

进入主线程消息循环。

---

AMS通知：

```java
scheduleLaunchActivity()
```

↓

Binder

↓

ActivityThread

---

# 十一、Handler切换主线程

收到消息：

```java
H.LAUNCH_ACTIVITY
```

↓

```java
handleLaunchActivity()
```

↓

```java
performLaunchActivity()
```

---

源码：

```java
private Activity performLaunchActivity(...)
```

---

# 十二、反射创建Activity

通过：

```java
ClassLoader
```

加载：

```java
MainActivity.class
```

---

反射：

```java
Instrumentation.newActivity()
```

↓

```java
clazz.newInstance()
```

---

创建：

```java
MainActivity
```

对象。

---

# 十三、执行生命周期

依次：

```java
activity.attach(...)
```

↓

```java
activity.onCreate()
```

↓

```java
activity.onStart()
```

↓

```java
activity.onResume()
```

---

最终：

```text
用户看到界面
```

---

# 十四、源码级流程图

```text
Activity.startActivity()

        ↓

Instrumentation.execStartActivity()

        ↓

IActivityTaskManager

        ↓ Binder

ActivityTaskManagerService

        ↓

ActivityStartController

        ↓

ActivityStarter.execute()

        ↓

创建ActivityRecord

        ↓

ActivityTaskSupervisor

        ↓

判断目标进程

        ↓

不存在
        ↓

AMS

        ↓

Zygote fork

        ↓

App进程

        ↓

ActivityThread

        ↓

performLaunchActivity()

        ↓

反射创建Activity

        ↓

onCreate()

        ↓

onStart()

        ↓

onResume()
```

---

# ActivityStarter核心职责

```java
ActivityStarter
```

主要负责：

|功能|说明|
|---|---|
|Intent解析|找到目标Activity|
|启动模式处理|standard/singleTask|
|Task选择|复用还是新建|
|ActivityRecord创建|创建Activity描述对象|
|Activity栈调整|压栈、出栈|
|启动流程调度|调用Supervisor|

---

# ActivityRecord核心职责

```java
ActivityRecord
```

保存：

```text
ActivityInfo

Intent

Token

Task

生命周期状态

进程信息
```

生命周期状态：

```java
RESUMED

PAUSED

STOPPED

DESTROYED
```

AMS/ATMS 实际上管理的不是 Activity 实例，而是 **ActivityRecord**。真正的 Activity 对象只存在于应用进程中，而 SystemServer 里只保存 Activity 的“元数据”和状态。

---

### 面试回答模板（3分钟）

> 用户调用 startActivity 后，会先进入 Instrumentation.execStartActivity，通过 Binder 调用 SystemServer 中的 ActivityTaskManagerService。ATMS 会通过 ActivityStarter 解析 Intent、处理 launchMode、选择 Task，并创建 ActivityRecord。随后 ActivityTaskSupervisor 判断目标进程是否存在，不存在则通过 AMS 请求 Zygote fork 新进程。新进程启动后 ActivityThread 收到 LAUNCH_ACTIVITY 消息，在 performLaunchActivity 中通过反射创建 Activity 实例，执行 attach、onCreate、onStart、onResume，最终显示界面。整个过程中 SystemServer 管理的是 ActivityRecord，而不是 Activity 对象本身。

这个回答已经达到 Android 高级工程师面试对 Activity 启动流程的源码级要求。