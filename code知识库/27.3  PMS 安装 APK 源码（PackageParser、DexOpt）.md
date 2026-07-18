PMS（PackageManagerService）安装 APK 是 Android Framework 面试中的经典题目。

面试官通常会问：

> APK 从点击安装到成功安装经历了什么？

> PackageParser 是干什么的？

> DexOpt 为什么能提高启动速度？

> packages.xml 作用是什么？

> APK 安装后系统如何找到 MainActivity？

---

# 一、APK安装整体流程

```text
用户点击APK
    ↓
PackageInstaller
    ↓ Binder
PackageManagerService(PMS)
    ↓
解析APK
    ↓
解析AndroidManifest.xml
    ↓
校验签名
    ↓
分配UID
    ↓
DexOpt优化
    ↓
写入packages.xml
    ↓
安装完成
```

源码主入口：

```java
PackageManagerService.installPackage()
```

Android 8.0以后：

```java
PackageInstallerSession.commit()
```

最终进入：

```java
PackageManagerService
```

---

# 二、PackageInstaller

安装APK：

```java
Intent intent = new Intent(Intent.ACTION_VIEW);
intent.setDataAndType(uri,
    "application/vnd.android.package-archive");
```

系统启动：

```text
PackageInstaller
```

---

PackageInstaller职责：

```text
安装界面
权限确认
调用PMS
```

实际安装工作：

```text
PMS完成
```

---

# 三、PMS接收安装请求

源码：

```java
PackageInstallerSession.commit()
```

↓

```java
PackageManagerService.installPackageTracedLI()
```

↓

```java
scanPackageNewLI()
```

开始解析APK。

---

# 四、PackageParser解析APK

核心类：

```java
PackageParser
```

源码：

```text
frameworks/base/core/java/android/content/pm/
PackageParser.java
```

作用：

```text
解析APK文件
解析Manifest
生成Package对象
```

---

例如：

```xml
<manifest package="com.demo.app">

    <application>

        <activity
            android:name=".MainActivity"/>

        <service
            android:name=".UploadService"/>

    </application>

</manifest>
```

---

解析：

```java
PackageParser.parsePackage()
```

↓

生成：

```java
PackageParser.Package
```

对象。

---

# Package对象结构

简化版：

```java
class Package {

    String packageName;

    ArrayList<Activity> activities;

    ArrayList<Service> services;

    ArrayList<Provider> providers;

}
```

---

结果：

```text
Package

├── MainActivity
├── UploadService
├── Provider
└── Permission
```

---

# 五、生成ActivityInfo

解析：

```xml
<activity
    android:name=".MainActivity"/>
```

生成：

```java
ActivityInfo
```

源码：

```java
PackageParser.parseActivity()
```

↓

```java
ActivityInfo
```

保存：

```java
Package.activities
```

中。

---

后续：

```java
startActivity()
```

时：

PMS查询：

```java
ActivityInfo
```

找到：

```text
MainActivity
```

---

# 六、签名校验

解析完成：

```java
collectCertificates()
```

↓

读取：

```text
META-INF/
```

目录。

例如：

```text
META-INF/CERT.RSA
META-INF/CERT.SF
```

---

校验：

```java
SigningDetails
```

确保：

```text
APK未被篡改
```

---

升级安装时：

```text
签名必须一致
```

否则：

```text
INSTALL_FAILED_UPDATE_INCOMPATIBLE
```

---

# 七、分配UID

Android沙箱机制：

每个应用：

```text
一个UID
```

例如：

```text
com.demo.app

UID=10123
```

---

源码：

```java
Settings.addPackageLPw()
```

↓

生成：

```java
PackageSetting
```

↓

分配：

```text
UID
```

---

保存：

```text
/data/system/packages.xml
```

---

# 八、PackageSetting

核心类：

```java
PackageSetting
```

保存：

```text
包名
UID
安装路径
权限
版本号
```

例如：

```java
PackageSetting {

    packageName="com.demo.app";

    appId=10123;

}
```

---

# 九、DexOpt优化

这是面试重点。

---

APK中的代码：

```text
classes.dex
```

ART无法直接高效执行。

需要：

```text
DexOpt
```

优化。

---

源码：

```java
PackageDexOptimizer
```

↓

```java
performDexOpt()
```

---

流程：

```text
classes.dex

    ↓

dex2oat

    ↓

odex/oat

    ↓

机器码
```

---

例如：

```text
APK

classes.dex
```

↓

生成：

```text
base.odex

base.vdex
```

---

存放：

```text
/data/app/

或

/data/dalvik-cache/
```

---

# DexOpt作用

作用：

```text
提前编译
减少解释执行
提高启动速度
```

---

没有DexOpt：

```text
classes.dex
 ↓
解释执行
```

速度慢。

---

有DexOpt：

```text
机器码
 ↓
直接执行
```

速度快。

---

# Android编译模式

## Android 5.0以前

Dalvik

```text
JIT
```

边运行边编译。

---

## Android 5.0+

ART

```text
AOT
```

安装时编译。

---

## Android 7.0+

混合模式：

```text
AOT

+
JIT

+
Profile
```

---

流程：

```text
首次安装
 ↓
快速编译

运行统计热点

 ↓

后台重新编译热点代码
```

---

# 十、写入packages.xml

安装成功后：

```java
Settings.writeLPr()
```

↓

写：

```text
/data/system/packages.xml
```

---

内容类似：

```xml
<package
    name="com.demo.app"
    codePath="/data/app/com.demo.app"
    userId="10123"/>
```

---

系统重启：

```java
PackageManagerService.main()
```

↓

读取：

```text
packages.xml
```

恢复：

```text
所有已安装应用
```

---

# 十一、Launcher显示应用

安装完成：

```java
ACTION_PACKAGE_ADDED
```

广播发送。

---

Launcher收到：

```text
PACKAGE_ADDED
```

↓

查询：

```java
PackageManager
```

↓

获取：

```java
LauncherActivityInfo
```

↓

桌面显示图标。

---

# 核心源码流程图

```text
APK

 ↓

PackageInstaller

 ↓

PackageManagerService

 ↓

PackageParser

 ↓

解析Manifest

 ↓

Package对象

 ↓

ActivityInfo

 ↓

签名校验

 ↓

分配UID

 ↓

PackageSetting

 ↓

DexOpt

 ↓

dex2oat

 ↓

odex/vdex

 ↓

packages.xml

 ↓

安装完成
```

---

# 面试高频问题

## 1. PackageParser作用

回答：

```text
PackageParser负责解析APK。

主要解析：

AndroidManifest.xml

Activity

Service

Receiver

Provider

Permission
```

生成：

```java
PackageParser.Package
```

对象。

---

## 2. DexOpt作用

回答：

```text
DexOpt负责把classes.dex优化成ART可执行格式。

通过dex2oat生成odex/oat文件。

减少解释执行，提高启动速度。
```

---

## 3. packages.xml作用

路径：

```text
/data/system/packages.xml
```

保存：

```text
包名
UID
安装路径
权限
```

系统重启后：

```text
PMS读取恢复应用信息
```

---

## 4. APK安装后如何找到MainActivity

PMS安装时：

```java
PackageParser.parseActivity()
```

生成：

```java
ActivityInfo
```

保存在内存结构中。

启动：

```java
startActivity()
```

时：

```java
PMS.resolveIntent()
```

根据 Intent 查询匹配的 ActivityInfo，再交给 ATMS/AMS 启动。

---

# 面试标准答案（3分钟）

> APK 安装时首先由 PackageInstaller 发起安装请求，通过 Binder 调用 PackageManagerService。PMS 使用 PackageParser 解析 APK 和 AndroidManifest.xml，生成 Package 对象以及 ActivityInfo、ServiceInfo 等组件信息。随后进行签名校验、分配 UID，并生成 PackageSetting 保存安装信息。接着执行 DexOpt，通过 dex2oat 将 classes.dex 编译成 oat/odex 文件，提高运行效率。最后将安装信息写入 `/data/system/packages.xml`，并发送 PACKAGE_ADDED 广播通知系统。后续启动 Activity 时，PMS 会根据解析好的 ActivityInfo 完成 Intent 匹配和组件查找。

---

### Framework源码面试三件套

掌握下面三条链路，基本覆盖 Android Framework 面试核心：

```text
Activity启动
startActivity
→ ATMS
→ ActivityStarter
→ ActivityRecord
→ ActivityThread

View显示
DecorView
→ ViewRootImpl
→ WMS
→ Surface
→ SurfaceFlinger

APK安装
PackageInstaller
→ PMS
→ PackageParser
→ DexOpt
→ packages.xml
```

这三条链路是高级 Android 工程师面试最常被追问的源码主线。