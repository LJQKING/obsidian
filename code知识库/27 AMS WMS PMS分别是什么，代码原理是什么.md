AMS、WMS、PMS 是 Android Framework 最核心的三个系统服务，几乎所有 App 的运行都离不开它们。

面试中经常问：

> Activity启动流程？

> Window是怎么显示的？

> APK安装过程？

本质上就是考察 AMS、WMS、PMS。

---

# 一、AMS（ActivityManagerService）

## 什么是AMS

AMS负责管理：

```text
Activity
Service
Broadcast
ContentProvider
进程(Process)
```

源码位置：

```text
frameworks/base/services/core/java/
    com/android/server/am/
        ActivityManagerService.java
```

---

## AMS架构

```text
App
 ↓
Activity.startActivity()
 ↓
Instrumentation
 ↓
ActivityTaskManager
 ↓ Binder
ActivityTaskManagerService
 ↓
AMS
 ↓
启动目标Activity
```

Android 10以后：

```text
AMS
负责进程管理

ATMS(ActivityTaskManagerService)
负责Activity栈管理
```

---

# AMS启动Activity原理

调用：

```java
startActivity(intent);
```

源码：

```java
// Activity.java
public void startActivity(Intent intent){
    Instrumentation.execStartActivity(...)
}
```

↓

```java
// Instrumentation.java
execStartActivity()
```

↓

```java
ActivityTaskManager.getService()
    .startActivity(...)
```

↓

Binder跨进程

↓

```java
ActivityTaskManagerService
```

↓

```java
ActivityStartController
```

↓

```java
ActivityStarter
```

↓

```java
ActivityRecord
```

↓

启动目标进程

↓

```java
ActivityThread.performLaunchActivity()
```

↓

```java
onCreate()
```

---

## AMS核心数据结构

### ActivityRecord

类似：

```java
class ActivityRecord {
    Intent intent;
    ActivityInfo info;
}
```

表示一个Activity。

---

### Task

```text
任务栈
```

例如：

```text
A
↓
B
↓
C
```

一个Task。

---

### ProcessRecord

```java
class ProcessRecord
```

记录：

```text
PID
UID
包名
进程状态
```

---

# 二、WMS（WindowManagerService）

---

## 什么是WMS

负责：

```text
窗口管理
View显示
Dialog
Toast
屏幕旋转
Window层级
```

源码：

```text
frameworks/base/services/core/java/
    com/android/server/wm/
        WindowManagerService.java
```

---

## WMS架构

```text
Activity
 ↓
Window
 ↓
WindowManager
 ↓ Binder
WindowManagerService
 ↓
SurfaceFlinger
 ↓
屏幕
```

---

## View显示原理

调用：

```java
setContentView(...)
```

实际上：

```text
DecorView
 ↓
ViewRootImpl
 ↓
WindowManagerGlobal
 ↓
WindowManagerService
```

---

### addView流程

```java
windowManager.addView(view);
```

源码：

```java
WindowManagerImpl
```

↓

```java
WindowManagerGlobal.addView()
```

↓

```java
ViewRootImpl
```

↓

```java
IWindowSession.addToDisplay()
```

↓

Binder

↓

```java
WindowManagerService
```

↓

创建WindowState

↓

Surface

↓

SurfaceFlinger

↓

显示到屏幕

---

# WMS核心类

---

## WindowState

一个窗口：

```java
WindowState
```

例如：

```text
MainActivity窗口

Dialog窗口

PopupWindow窗口
```

都对应一个：

```java
WindowState
```

---

## DisplayContent

表示屏幕：

```java
DisplayContent
```

例如：

```text
主屏幕
外接屏幕
```

---

## RootWindowContainer

管理所有窗口。

```text
RootWindowContainer
  ↓
DisplayContent
  ↓
WindowState
```

---

# 三、PMS（PackageManagerService）

---

## 什么是PMS

负责：

```text
APK安装
APK卸载
权限管理
组件解析
应用信息查询
```

源码：

```text
frameworks/base/services/core/java/
    com/android/server/pm/
        PackageManagerService.java
```

---

# PMS架构

```text
APK
 ↓
PackageInstaller
 ↓ Binder
PackageManagerService
 ↓
解析AndroidManifest.xml
 ↓
写入packages.xml
 ↓
安装完成
```

---

# APK安装流程

点击安装：

```text
PackageInstaller
 ↓
PMS.installPackage()
 ↓
PackageParser
 ↓
解析Manifest
 ↓
创建Package对象
 ↓
dexopt
 ↓
生成odex
 ↓
写入packages.xml
 ↓
安装成功
```

---

# PMS解析APK原理

Manifest：

```xml
<activity
    android:name=".MainActivity"/>
```

PMS解析：

```java
PackageParser.parsePackage()
```

生成：

```java
ActivityInfo
```

保存到：

```java
PackageSetting
```

---

# PMS核心数据结构

---

## PackageParser.Package

表示一个APK。

```java
Package package
```

包含：

```text
Activity
Service
Receiver
Provider
Permission
```

---

## PackageSetting

安装信息：

```text
安装路径
版本号
UID
权限
```

---

## packages.xml

路径：

```text
/data/system/packages.xml
```

记录：

```xml
<package
    name="com.demo.app"
    codePath="/data/app/..."/>
```

系统重启后：

```text
PMS读取packages.xml
```

恢复应用信息。

---

# 三大服务关系图

```text
                 App
                  │
     ┌────────────┼────────────┐
     ▼            ▼            ▼

    AMS          WMS          PMS
(Activity)   (Window)     (Package)

     │            │            │
     ▼            ▼            ▼

 Activity栈     Surface     APK安装

     │            │            │
     └──────┬─────┴──────┬─────┘
            ▼
         Binder
            ▼
      SystemServer
```

---

# 面试高频问题

## 1. AMS和ATMS区别

Android 10后：

```text
AMS
负责进程管理

ATMS
负责Activity任务栈管理
```

---

## 2. WMS和SurfaceFlinger区别

### WMS

负责：

```text
窗口管理
窗口层级
焦点管理
```

### SurfaceFlinger

负责：

```text
图层合成
GPU渲染
显示到屏幕
```

流程：

```text
View
 ↓
WMS
 ↓
Surface
 ↓
SurfaceFlinger
 ↓
屏幕
```

---

## 3. PMS安装APK做了什么

```text
解析APK
解析Manifest
校验签名
分配UID
Dex优化
写入packages.xml
```

---

## 4. Activity启动涉及哪些服务

```text
Activity
 ↓
Instrumentation
 ↓
ATMS
 ↓
AMS
 ↓
Zygote
 ↓
Application
 ↓
ActivityThread
```

---

# 源码级记忆图

```text
startActivity()
      ↓
Instrumentation
      ↓
ATMS
      ↓
AMS
      ↓
Zygote
      ↓
ActivityThread

--------------------------------

setContentView()
      ↓
ViewRootImpl
      ↓
WMS
      ↓
SurfaceFlinger

--------------------------------

安装APK
      ↓
PackageInstaller
      ↓
PMS
      ↓
PackageParser
      ↓
packages.xml
```

如果你准备 Android 高级工程师面试，建议深入掌握：

- AMS 启动 Activity 源码（ActivityStarter、ActivityRecord）
    
- WMS 显示 View 源码（ViewRootImpl、Surface）
    
- PMS 安装 APK 源码（PackageParser、DexOpt）
    
- Binder IPC 原理（这些服务都是通过 Binder 调用）
    

这四部分是 Framework 面试的核心内容。