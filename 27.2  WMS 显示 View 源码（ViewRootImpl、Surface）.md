WMS（WindowManagerService）相关源码是 Android Framework 面试第二高频题，仅次于 Activity 启动。

面试官经常问：

> View 是如何显示到屏幕上的？

> setContentView() 后发生了什么？

> ViewRootImpl 是什么？

> Surface 和 SurfaceFlinger 有什么关系？

---

# 一、View显示整体流程

一张图先看懂：

```text
Activity

setContentView()

    ↓

DecorView

    ↓

ViewRootImpl

    ↓

WindowManagerGlobal

    ↓

IWindowSession

    ↓ Binder

WindowManagerService

    ↓

Surface

    ↓

SurfaceFlinger

    ↓

GPU

    ↓

Display
```

---

# 二、Activity创建窗口

当Activity启动时：

```java
ActivityThread.performLaunchActivity()
```

执行：

```java
activity.attach(...)
```

源码：

```java
final void attach(...) {

    mWindow = new PhoneWindow(this);
}
```

创建：

```java
PhoneWindow
```

---

PhoneWindow：

```text
每个Activity对应一个Window
```

结构：

```text
Activity
   │
PhoneWindow
   │
DecorView
```

---

# 三、setContentView发生什么

代码：

```java
setContentView(R.layout.activity_main);
```

源码：

```java
PhoneWindow.setContentView()
```

↓

```java
installDecor();
```

↓

创建：

```java
DecorView
```

---

DecorView：

```text
整个页面根View
```

结构：

```text
DecorView
    │
FrameLayout
    │
activity_main.xml
```

例如：

```xml
LinearLayout
    TextView
    Button
```

最终：

```text
DecorView
 └── LinearLayout
      ├── TextView
      └── Button
```

---

# 四、Activity显示到屏幕

关键代码：

```java
ActivityThread.handleResumeActivity()
```

↓

```java
WindowManager wm =
    a.getWindowManager();
```

↓

```java
wm.addView(decorView);
```

这里非常重要。

---

# 五、WindowManager.addView

源码：

```java
WindowManagerImpl.addView()
```

↓

```java
WindowManagerGlobal.addView()
```

↓

创建：

```java
ViewRootImpl
```

源码：

```java
root = new ViewRootImpl(...)
```

---

# ViewRootImpl是什么

很多人误以为：

```text
ViewRootImpl是View
```

其实不是。

它：

```text
View树与WMS之间的桥梁
```

结构：

```text
DecorView
     │
ViewRootImpl
     │
WMS
```

---

# ViewRootImpl职责

负责：

```text
View测量
View布局
View绘制
事件分发
与WMS通信
Surface管理
```

所以：

```text
ViewRootImpl
=
View系统总管
```

---

# 六、建立窗口

继续：

```java
WindowManagerGlobal.addView()
```

↓

```java
root.setView(view);
```

这里：

```java
view
=
DecorView
```

---

源码：

```java
ViewRootImpl.setView()
```

↓

```java
requestLayout();
```

↓

首次绘制开始。

---

# 七、与WMS通信

ViewRootImpl：

```java
mWindowSession.addToDisplay(...)
```

其中：

```java
mWindowSession
```

实际上：

```java
IWindowSession
```

---

通过：

```java
WindowManagerGlobal.getWindowSession()
```

获取。

---

跨进程：

```text
App
 ↓
Binder
 ↓
WindowManagerService
```

---

WMS收到：

```java
addWindow()
```

---

创建：

```java
WindowState
```

---

# WindowState

对应：

```text
一个Window
```

例如：

```text
Activity窗口

Dialog窗口

PopupWindow窗口
```

都有：

```java
WindowState
```

对象。

---

结构：

```text
RootWindowContainer
    │
DisplayContent
    │
WindowState
```

---

# 八、创建Surface

WMS创建：

```java
WindowState
```

后：

```java
createSurface()
```

↓

```java
SurfaceControl
```

↓

```java
Surface
```

---

Surface是什么？

可以理解：

```text
一块共享内存
```

专门用于：

```text
保存绘制结果
```

---

结构：

```text
Canvas
   ↓
Surface
   ↓
SurfaceFlinger
```

---

# 九、首次绘制

回到：

```java
ViewRootImpl
```

调用：

```java
performTraversals()
```

这是View系统最核心方法。

---

# performTraversals

源码：

```java
private void performTraversals()
```

内部：

```java
performMeasure()
```

↓

```java
performLayout()
```

↓

```java
performDraw()
```

---

对应：

```text
Measure

Layout

Draw
```

即：

```text
测量

布局

绘制
```

---

# 十、Measure

执行：

```java
measure()
```

↓

递归：

```java
ViewGroup.measureChild()
```

↓

所有View获得：

```text
宽
高
```

---

# 十一、Layout

执行：

```java
layout()
```

↓

计算：

```text
left
top
right
bottom
```

确定位置。

---

# 十二、Draw

执行：

```java
draw()
```

↓

```java
onDraw()
```

↓

Canvas绘制

↓

保存到：

```java
Surface
```

---

流程：

```text
View
 ↓
Canvas
 ↓
Surface
```

---

# 十三、SurfaceFlinger

绘制完成后：

```text
Surface
```

交给：

```java
SurfaceFlinger
```

---

SurfaceFlinger：

```text
Android图层合成服务
```

源码：

```text
frameworks/native/services/surfaceflinger
```

---

职责：

```text
接收所有Surface

图层合成

GPU渲染

输出屏幕
```

---

例如：

```text
状态栏

Activity

Dialog
```

分别：

```text
Surface1

Surface2

Surface3
```

---

SurfaceFlinger：

```text
合成为一张图
```

↓

显示。

---

# 十四、View刷新原理

调用：

```java
textView.invalidate();
```

↓

```java
ViewRootImpl.scheduleTraversals()
```

↓

Choreographer

↓

VSYNC

↓

performTraversals()

↓

重新Draw

---

# VSYNC机制

屏幕：

```text
60Hz
```

即：

```text
16.6ms
```

刷新一次。

---

系统收到：

```text
VSYNC信号
```

后：

```java
Choreographer
```

触发：

```java
performTraversals()
```

保证：

```text
UI刷新同步屏幕
```

---

# 核心源码流程图

```text
setContentView()

      ↓

PhoneWindow

      ↓

DecorView

      ↓

WindowManager.addView()

      ↓

ViewRootImpl.setView()

      ↓

IWindowSession.addToDisplay()

      ↓ Binder

WindowManagerService

      ↓

WindowState

      ↓

Surface

      ↓

performTraversals()

      ↓

Measure

      ↓

Layout

      ↓

Draw

      ↓

Canvas

      ↓

Surface

      ↓

SurfaceFlinger

      ↓

GPU

      ↓

Display
```

---

# 面试高频问题

## 1. ViewRootImpl是什么？

回答：

```text
ViewRootImpl不是View。

它是View树与WindowManagerService之间的桥梁。

负责：

Measure
Layout
Draw
Input事件
Surface管理
Binder通信
```

---

## 2. Surface是什么？

回答：

```text
Surface是一块共享内存。

View绘制结果最终写入Surface。

SurfaceFlinger读取Surface并进行图层合成。
```

---

## 3. performTraversals干什么？

核心：

```java
performMeasure()

performLayout()

performDraw()
```

即：

```text
View三大流程入口
```

---

## 4. WMS和SurfaceFlinger区别？

### WMS

负责：

```text
窗口管理
层级管理
焦点管理
WindowState
```

### SurfaceFlinger

负责：

```text
Surface管理
图层合成
GPU渲染
屏幕显示
```

---

## 面试标准答案（2分钟）

> Activity 在 handleResumeActivity 时会调用 WindowManager.addView 添加 DecorView。WindowManagerGlobal 会创建 ViewRootImpl，并通过 IWindowSession 使用 Binder 调用 WMS。WMS 为窗口创建 WindowState 和 Surface。随后 ViewRootImpl 执行 performTraversals，完成 Measure、Layout、Draw 三大流程，将绘制结果写入 Surface。SurfaceFlinger 读取各个 Surface，通过 GPU 进行图层合成，最终显示到屏幕。ViewRootImpl 是 View 系统和 WMS 的桥梁，而 Surface 是 View 与 SurfaceFlinger 之间的数据载体。

这套流程是 Android Framework 中 View 显示机制的核心，也是高级面试必考内容。