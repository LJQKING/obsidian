# Android App 逆向工程工具完整指南

## 🔍 **工具分类与工作流**

### **第一阶段：APK 获取与初步分析**

|工具|用途|平台|说明|
|---|---|---|---|
|**adb**|从设备提取 APK|Win/Mac/Linux|`adb pull /data/app/com.xxx/base.apk`|
|**apktool**|APK 解包|跨平台|提取资源、反汇编 Dex，支持重新打包|
|**aapt/aapt2**|资源分析|跨平台|Android SDK 自带，查看 Manifest、资源信息|
|**zipinfo**|查看 APK 结构|跨平台|APK 本质是 ZIP，快速审视内部结构|

**快速上手：**

```bash
# 解包 APK
apktool d app.apk -o app_unpacked

# 查看 AndroidManifest.xml
aapt dump badging app.apk

# 提取 DEX
unzip app.apk "*.dex"
```

---

### **第二阶段：反编译 → 源代码恢复**

#### **1. DEX 反编译器**

|工具|输出格式|优势|缺陷|
|---|---|---|---|
|**JADX**|Java 源码|UI 友好、符号表完整、支持搜索|混淆代码可读性一般|
|**CFR**|Java 源码|Lambda/Stream 还原最好|命令行，需熟悉参数|
|**Procyon**|Java 源码|处理新 Java 特性|更新不频繁|
|**Fernflower**|Java 源码|JetBrains 出品，Kotlin 还原较好|需 IntelliJ 环境|

**推荐组合：**

```bash
# Kotlin 项目优先用 JADX（GUI）
jadx -d output app.dex

# 混淆代码用 CFR（参数丰富）
cfr --classic-class-output --caseinsensitivefs app.jar
```

#### **2. 进阶：Smali 汇编分析**

- **Smali** = Android 专用汇编语言（虚拟机字节码的文本表示）
- 用途：反混淆、理解虚拟机指令、定位关键逻辑
- 工具：**apktool** 自动生成 `.smali` 文件

**Smali 示例（理解虚拟机调用）：**

```smali
# 对应 Java: this.onCreate(savedInstanceState)
invoke-super {p0, p1}, Landroid/app/Activity;->onCreate(Landroid/os/Bundle;)V

# 对应 Java: String result = method(arg1, arg2)
invoke-static {v0, v1}, Lcom/example/Utils;->process(II)Ljava/lang/String;
move-result-object v2
```

---

### **第三阶段：动态分析 & Hook**

#### **1. Frida（强大的动态插桩框架）**

**安装（Mac + Apple Silicon）：**

```bash
brew install frida
pip install frida-tools --break-system-packages
```

**核心用途：**

- 运行时拦截函数调用
- 修改返回值
- 监控参数
- Bypass 反调试

**Frida 脚本示例（Hook Java 方法）：**

```javascript
// Hook Activity.onCreate
Java.perform(function() {
    var Activity = Java.use("android.app.Activity");
    Activity.onCreate.implementation = function(savedInstanceState) {
        console.log("[+] onCreate called");
        return this.onCreate(savedInstanceState);
    };
});
```

**使用流程：**

```bash
# 1. 启动 Frida Server（设备上）
adb push frida-server /data/local/tmp/
adb shell chmod +x /data/local/tmp/frida-server
adb shell /data/local/tmp/frida-server

# 2. 列出进程
frida-ps -U

# 3. Hook 应用
frida -U -f com.example.app -l hook.js --no-pause
```

#### **2. Xposed Framework（深度系统级 Hook）**

- **原理：** 替换 app_process，在 Zygote 进程创建时注入
- **优势：** 全局 Hook，可修改系统 API 行为
- **限制：** 需要 root，仅支持特定 Android 版本

**LSPosed（Xposed 现代替代）：**

```
支持 Android 8.1 ~ 14
通过 Magisk 模块安装
比原生 Xposed 稳定性更好
```

---

### **第四阶段：调试与内存分析**

|工具|功能|使用场景|
|---|---|---|
|**Android Studio Debugger**|断点调试、变量检查|调试可编译代码|
|**IDA Pro**|反汇编 + 调试 native code|C/C++ so 文件分析|
|**Ghidra**|开源逆向工程套件|native 代码（SOC 出品）|
|**MAT（Memory Analyzer Tool）**|heap dump 分析|内存泄漏诊断|
|**Android Profiler**|CPU/内存/网络profiling|性能分析|

**Heap Dump 分析流程：**

```bash
# 1. 导出 heap
adb shell am dumpheap com.example.app /data/local/tmp/heap.dump

# 2. 转换为 MAT 格式
hprof-conv heap.dump heap.hprof

# 3. 在 MAT 中打开分析
```

---

### **第五阶段：辅助工具**

|工具|功能|
|---|---|
|**Burp Suite**|网络流量监听（HTTPS 需配置证书）|
|**Charles Proxy**|HTTP/HTTPS 代理拦截|
|**Wireshark**|低层协议分析|
|**bytecode-viewer**|多反编译器对比|
|**jadx-gui**|JADX 可视化界面|
|**ClassyShark**|APK 快速分析工具|

---

## 🛠️ **完整逆向流程示例**

假设要分析某 App 的登录逻辑：

```bash
# Step 1: 获取 APK
adb pull /data/app/com.example.app/base.apk

# Step 2: 反编译
jadx -d output app.apk

# Step 3: 定位登录代码
# 在 JADX GUI 中搜索 "login" / "password" / "authenticate"

# Step 4: 动态验证（Frida Hook 绕过）
# 创建 bypass.js
Java.perform(function() {
    var AuthUtils = Java.use("com.example.app.utils.AuthUtils");
    AuthUtils.isTokenValid.implementation = function(token) {
        console.log("Token: " + token);
        return true; // 强制返回有效
    };
});

# Step 5: 注入运行
frida -U -f com.example.app -l bypass.js --no-pause
```

---

## 🚨 **中国大陆网络限制注意**

部分工具可能需要梯子：

- GitHub releases（部分）
- NPM/PyPI 包下载（建议用国内源）
- IDA Pro 官网激活

**本地镜像方案：**

```bash
# 使用清华 PyPI 源
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple

# 下载工具后离线使用
```

---

## 📚 **建议学习路径（针对你的背景）**

1. **第一周**：JADX 反编译 + apktool 解包（理解 APK 结构）
2. **第二周**：Smali 字节码（对标你的 Java 字节码理解）
3. **第三周**：Frida Hook（实战绕过逻辑）
4. **第四周**：native 代码（IDA Pro 分析 .so）

---

**你想从哪个工具开始深入？** 比如：

- JADX 反编译 Kotlin 代码的最佳实践
- Frida Hook 原理与反反调试
- Smali 虚拟机指令详解
- native so 动态分析

告诉我，我可以提供实战代码和原理分析！