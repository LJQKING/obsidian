Android Framework 是 Android 系统中最核心的一层，它位于 Linux Kernel 和应用层之间，为开发者提供了大量 API，使开发者无需直接操作底层驱动即可开发 App。

---

# 一、Android 系统架构

Android 整体架构分为 5 层：

```text
+----------------------+
|      Applications    |
|  微信、抖音、QQ等     |
+----------------------+
|   Application        |
|     Framework        |
| ActivityManager      |
| WindowManager        |
| PackageManager       |
| NotificationManager  |
+----------------------+
| Android Runtime      |
|      ART/Dalvik      |
| Core Java Libraries  |
+----------------------+
| Native Libraries     |
| SQLite/OpenGL/WebKit |
| SurfaceFlinger       |
+----------------------+
| Linux Kernel         |
| Driver/Bluetooth/Wifi|
+----------------------+
```

---

# 二、Framework是什么

Framework本质上是一套Java API + 系统服务。

例如：

```java
startActivity(intent);
```

实际上调用的是：

```java
ActivityManagerService(AMS)
```

Framework帮我们完成：

- Activity管理
    
- Service管理
    
- Window管理
    
- View绘制
    
- 广播管理
    
- 权限管理
    
- 蓝牙管理
    
- Wifi管理
    
- 传感器管理
    

等等。

---

# 三、Framework组成

Framework源码目录：

```text
frameworks/
    base/
        core/
        services/
        packages/
```

核心模块：

---

## 1 ActivityManagerService (AMS)

管理：

```text
Activity
Service
Broadcast
ContentProvider
```

源码：

```text
frameworks/base/services/core/java/
    com/android/server/am/
```

例如：

```java
startActivity()
```

流程：

```text
Activity
 ↓
Instrumentation
 ↓
ActivityTaskManager
 ↓
Binder
 ↓
AMS
 ↓
创建Activity
```

---

## 2 WindowManagerService (WMS)

管理：

```text
窗口
Dialog
Toast
屏幕旋转
```

源码：

```text
frameworks/base/services/core/java/
    com/android/server/wm/
```

例如：

```java
dialog.show();
```

最终：

```text
WindowManagerService
```

负责添加窗口。

---

## 3 PackageManagerService (PMS)

管理：

```text
APP安装
卸载
APK解析
权限管理
```

例如：

```java
PackageManager pm=getPackageManager();
```

查询：

```java
pm.getInstalledPackages();
```

底层：

```text
PackageManagerService
```

---

## 4 NotificationManagerService

管理：

```text
通知栏
推送
消息提醒
```

例如：

```java
NotificationManager.notify();
```

最终：

```text
NotificationManagerService
```

---

## 5 LocationManagerService

管理：

```text
GPS
基站定位
网络定位
```

例如：

```java
LocationManager manager=
    getSystemService(LocationManager.class);
```

---

## 6 BluetoothManagerService

管理：

```text
BLE
经典蓝牙
设备连接
```

例如：

```java
BluetoothAdapter.getDefaultAdapter();
```

最终：

```text
BluetoothManagerService
```

你做 BLE 开发时：

```java
RxAndroidBle
```

底层最终也是：

```text
BluetoothGatt
↓
Framework
↓
BluetoothManagerService
↓
蓝牙驱动
```

---

# 四、Framework通信机制

Framework大量使用 Binder。

---

## Binder结构

```text
APP
 ↓
Proxy
 ↓
Binder Driver
 ↓
Stub
 ↓
System Service
```

例如：

```java
ActivityManager am=
    getSystemService(ActivityManager.class);
```

实际上：

```java
IActivityManager
```

通过 Binder 调用：

```java
ActivityManagerService
```

---

# 五、Framework启动过程

开机流程：

```text
BootLoader
 ↓
Linux Kernel
 ↓
init进程
 ↓
Zygote
 ↓
SystemServer
 ↓
AMS/WMS/PMS
 ↓
Launcher
```

重点：

```text
SystemServer
```

启动所有Framework服务：

```java
startBootstrapServices();
startCoreServices();
startOtherServices();
```

启动：

```text
AMS
WMS
PMS
PowerManager
BluetoothManager
```

---

# 六、Framework如何使用

开发者每天都在使用 Framework。

---

## Activity

```java
Intent intent =
        new Intent(this,MainActivity.class);

startActivity(intent);
```

Framework：

```text
AMS
```

负责启动。

---

## Service

```java
startService(intent);
```

Framework：

```text
AMS
```

负责管理生命周期。

---

## Broadcast

```java
sendBroadcast(intent);
```

Framework：

```text
BroadcastQueue
```

负责分发。

---

## ContentProvider

```java
getContentResolver().query();
```

Framework：

```text
ContentProvider
```

负责跨进程访问数据。

---

## 蓝牙

```java
BluetoothAdapter adapter =
        BluetoothAdapter.getDefaultAdapter();
```

Framework：

```text
BluetoothManagerService
```

负责设备管理。

---

## Wifi

```java
WifiManager wifiManager =
    (WifiManager)getSystemService(WIFI_SERVICE);
```

Framework：

```text
WifiService
```

负责Wifi控制。

---

# 七、Framework源码级调用流程

以启动Activity为例：

```java
startActivity(intent);
```

源码流程：

```text
Activity.java
 ↓
Instrumentation.java
 ↓
ActivityTaskManager.java
 ↓
IActivityTaskManager.aidl
 ↓
Binder
 ↓
ActivityTaskManagerService
 ↓
RootWindowContainer
 ↓
ActivityRecord
 ↓
ActivityThread
 ↓
performLaunchActivity
 ↓
MainActivity.onCreate()
```

面试经常问：

> startActivity到底发生了什么？

回答就是上面这一套流程。

---

# 八、Android开发最常用Framework模块

|Framework模块|作用|
|---|---|
|AMS|Activity管理|
|ATMS|Activity栈管理|
|WMS|Window管理|
|PMS|应用安装管理|
|NMS|通知管理|
|PowerManager|电源管理|
|BluetoothManager|蓝牙|
|WifiManager|Wifi|
|SensorManager|传感器|
|CameraService|相机|
|AudioService|音频|

---

# 九、高级工程师面试常问

### 1. startActivity源码流程？

涉及：

```text
Instrumentation
ATMS
Binder
ActivityThread
```

---

### 2. Activity生命周期谁控制？

```text
AMS/ATMS
```

控制。

---

### 3. Framework为什么用Binder？

优点：

- 性能高
    
- 内存拷贝少
    
- 安全性高
    
- 支持跨进程
    

---

### 4. SystemServer作用？

负责启动：

```text
AMS
WMS
PMS
BluetoothManager
PowerManager
```

等所有系统服务。

---

### 5. View为什么能显示到屏幕？

流程：

```text
View
 ↓
Window
 ↓
WMS
 ↓
SurfaceFlinger
 ↓
GPU
 ↓
屏幕
```

---

# 一张图理解Framework

```text
App
 │
 ▼
Framework API
(Activity、Service、View)
 │
 ▼ Binder
System Service
(AMS/WMS/PMS)
 │
 ▼
Native Service
(SurfaceFlinger、Media)
 │
 ▼
Linux Driver
(Bluetooth/Wifi/Display)
 │
 ▼
Hardware
```

对于 Android 高级开发来说，重点掌握以下 Framework 源码：

1. Activity 启动流程（AMS/ATMS）
    
2. View 绘制流程（ViewRootImpl）
    
3. 事件分发机制（dispatchTouchEvent）
    
4. Handler 机制（Looper/MessageQueue）
    
5. Binder IPC 原理
    
6. SystemServer 启动流程
    
7. WMS 与 SurfaceFlinger
    
8. PMS 安装流程
    
9. ContentProvider 跨进程原理
    
10. ANR 产生机制与源码分析
    

这些几乎覆盖 Android 高级工程师面试中 70% 以上的 Framework 问题。