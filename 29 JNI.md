# Android JNI 详解（原理 + 使用 + 源码分析 + 面试）

JNI（Java Native Interface）是 Java 与 C/C++ 之间的桥梁。

Android Framework、Binder、Media、OpenGL、蓝牙协议栈等底层模块大量使用 JNI。

---

# 一、为什么需要 JNI

Android 应用代码：

```java
Activity
Service
View
```

运行在：

```text
ART虚拟机
```

而很多系统能力来自：

```text
Linux Kernel
C/C++
硬件驱动
```

例如：

```text
Camera
Bluetooth
Binder
OpenGL
MediaCodec
```

都是 Native 实现。

因此需要：

```text
Java ↔ C++
```

互相调用。

---

# JNI架构

```text
Java层
│
├── Activity
├── Framework
│
▼
JNI
│
▼
Native层
│
├── C
├── C++
│
▼
Linux Kernel
```

---

# 二、JNI能做什么

## Java调用C++

```java
public native String getName();
```

↓

```cpp
JNIEXPORT jstring JNICALL
Java_com_test_MainActivity_getName(...)
{
    return ...
}
```

---

## C++调用Java

```cpp
env->CallVoidMethod(...)
```

↓

```java
showToast();
```

---

## Framework大量使用JNI

例如：

```java
System.loadLibrary("android_runtime");
```

加载：

```text
libandroid_runtime.so
```

---

常见JNI模块：

|模块|Native|
|---|---|
|Binder|libbinder.so|
|View|android_view_View.cpp|
|Bitmap|Bitmap.cpp|
|Camera|Camera.cpp|
|Media|MediaPlayer.cpp|
|Bluetooth|Fluoride|

---

# 三、JNI开发流程

## Step1

Java声明native

```java
public class NativeUtil {

    public native String getMessage();

}
```

---

## Step2

加载so

```java
static {
    System.loadLibrary("native-lib");
}
```

对应：

```text
libnative-lib.so
```

---

## Step3

实现C++

```cpp
extern "C"
JNIEXPORT jstring JNICALL
Java_com_demo_NativeUtil_getMessage(
        JNIEnv *env,
        jobject thiz)
{
    return env->NewStringUTF(
            "Hello JNI");
}
```

---

## Step4

调用

```java
NativeUtil util = new NativeUtil();

String msg = util.getMessage();
```

输出：

```text
Hello JNI
```

---

# 四、JNI数据类型映射

Java：

```java
int
long
String
boolean
```

对应：

|Java|JNI|
|---|---|
|int|jint|
|long|jlong|
|boolean|jboolean|
|float|jfloat|
|double|jdouble|
|String|jstring|
|Object|jobject|

---

例如：

```java
public native int add(
        int a,
        int b);
```

JNI：

```cpp
JNIEXPORT jint JNICALL
Java_xxx_add(
        JNIEnv* env,
        jobject thiz,
        jint a,
        jint b)
{
    return a+b;
}
```

---

# 五、JNIEnv是什么

最重要对象：

```cpp
JNIEnv*
```

所有JNI操作都通过它完成。

类似：

```java
Context
```

作用：

```text
创建对象
调用Java方法
操作字符串
访问字段
```

---

例如：

创建字符串：

```cpp
env->NewStringUTF("Hello");
```

---

获取类：

```cpp
jclass clazz =
    env->FindClass(
        "java/lang/String");
```

---

# 六、Java调用Native原理

Java：

```java
native void test();
```

调用：

```java
test();
```

↓

ART

↓

JNI桥接

↓

Native函数

```cpp
Java_xxx_test()
```

---

流程：

```text
Java
 ↓
ART
 ↓
JNI
 ↓
Native
 ↓
返回
```

---

# 七、动态注册JNI（推荐）

实际开发很少使用：

```cpp
Java_xxx_xxx()
```

因为名称太长。

---

定义：

```cpp
jstring nativeGetName(
        JNIEnv* env,
        jobject obj)
{
    return env->NewStringUTF("Tom");
}
```

---

注册表：

```cpp
static JNINativeMethod methods[] = {

{
 "getName",
 "()Ljava/lang/String;",
 (void*)nativeGetName
}

};
```

---

注册：

```cpp
JNI_OnLoad()
```

```cpp
JNIEXPORT jint JNICALL
JNI_OnLoad(
        JavaVM* vm,
        void*)
{
    JNIEnv* env;

    vm->GetEnv(
        (void**)&env,
        JNI_VERSION_1_6);

    jclass clazz =
      env->FindClass(
      "com/demo/NativeUtil");

    env->RegisterNatives(
          clazz,
          methods,
          sizeof(methods)
          /sizeof(methods[0]));

    return JNI_VERSION_1_6;
}
```

---

# 八、JNI调用Java

Native调用Java：

---

Java：

```java
public void showToast()
{
}
```

---

获取类

```cpp
jclass clazz =
    env->GetObjectClass(thiz);
```

---

获取方法

```cpp
jmethodID method =
    env->GetMethodID(
        clazz,
        "showToast",
        "()V");
```

---

调用

```cpp
env->CallVoidMethod(
        thiz,
        method);
```

---

流程：

```text
Native
 ↓
Find Method
 ↓
CallVoidMethod
 ↓
Java执行
```

---

# 九、JNI字符串转换

Java String

↓

C String

```cpp
const char* str =
 env->GetStringUTFChars(
       jstr,
       NULL);
```

---

使用后必须释放：

```cpp
env->ReleaseStringUTFChars(
       jstr,
       str);
```

否则：

```text
内存泄漏
```

---

# 十、JNI数组操作

Java：

```java
int[] arr;
```

JNI：

```cpp
jintArray
```

获取：

```cpp
jint* data =
env->GetIntArrayElements(
        arr,
        NULL);
```

长度：

```cpp
env->GetArrayLength(arr);
```

释放：

```cpp
env->ReleaseIntArrayElements(
        arr,
        data,
        0);
```

---

# 十一、JNI线程问题

错误写法：

```cpp
std::thread([]{

    env->CallVoidMethod(...);

});
```

崩溃：

```text
JNI DETECTED ERROR
thread not attached
```

---

原因：

JNIEnv只能当前线程使用。

---

正确：

```cpp
JavaVM->AttachCurrentThread()
```

```cpp
JNIEnv* env;

gVm->AttachCurrentThread(
      &env,
      nullptr);
```

结束：

```cpp
gVm->DetachCurrentThread();
```

---

# 十二、JNI引用类型

## Local Reference

默认：

```cpp
jobject obj
```

函数结束自动释放。

---

## Global Reference

跨函数使用：

```cpp
NewGlobalRef()
```

```cpp
gObj =
env->NewGlobalRef(obj);
```

释放：

```cpp
DeleteGlobalRef()
```

---

## WeakGlobalRef

弱引用：

```cpp
NewWeakGlobalRef()
```

避免内存泄漏。

---

# 十三、Android源码中的JNI

## View JNI

Java：

```java
View.invalidate()
```

↓

JNI

```cpp
android_view_View.cpp
```

↓

Native

```cpp
RenderNode
```

---

## Bitmap JNI

Java：

```java
Bitmap.createBitmap()
```

↓

JNI

```cpp
Bitmap.cpp
```

↓

Skia

---

## Binder JNI

Java：

```java
BinderProxy.transact()
```

↓

JNI：

```cpp
android_util_Binder.cpp
```

↓

Native：

```cpp
BpBinder::transact()
```

↓

Binder Driver

---

# 十四、JNI性能优化

## 缓存Class

不要：

```cpp
FindClass()
```

每次调用。

---

缓存：

```cpp
static jclass gClazz;
```

---

## 缓存MethodID

```cpp
static jmethodID gMethod;
```

---

避免频繁：

```cpp
GetMethodID()
```

---

## 减少Java与Native切换

错误：

```text
循环10000次
每次JNI调用
```

正确：

```text
一次传数组
Native处理
```

---

# 十五、JNI面试高频题

### 1. JNI是什么？

Java 与 Native(C/C++) 通信规范。

---

### 2. JNIEnv是什么？

JNI操作接口，负责：

```text
对象创建
方法调用
字段访问
字符串处理
```

---

### 3. Java如何调用Native？

```text
native
 ↓
ART
 ↓
JNI
 ↓
C++
```

---

### 4. Native如何调用Java？

```cpp
GetMethodID()
CallVoidMethod()
```

---

### 5. JNIEnv为什么不能跨线程？

JNIEnv 与线程绑定。

跨线程必须：

```cpp
AttachCurrentThread()
```

---

### 6. 动态注册与静态注册区别？

|方式|特点|
|---|---|
|静态注册|Java_xxx_xxx|
|动态注册|RegisterNatives|

Android Framework 基本全部采用动态注册。

---

# Android Framework中的JNI调用链（面试必背）

```text
Java层

View
Binder
Bitmap
MediaPlayer

     ↓

JNI

android_view_View.cpp
android_util_Binder.cpp
android_graphics_Bitmap.cpp

     ↓

Native层

libandroid_runtime.so
libbinder.so
libgui.so

     ↓

Linux Kernel
```

掌握 JNI 后，再深入学习 Android Framework 时，就能看懂很多核心链路，例如：

- Binder：Java → JNI → Native Binder → Binder Driver
    
- SurfaceFlinger：Java → JNI → Native → GPU
    
- Camera：Java → JNI → CameraService
    
- MediaPlayer：Java → JNI → MediaCodec/FFmpeg 风格解码链路
    

这些系统能力最终都会经过 JNI 从 Java 世界进入 Native 世界。