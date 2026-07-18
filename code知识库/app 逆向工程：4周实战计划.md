# Android 逆向工程：4周实战计划（详细步骤版）

---

## 📅 **第一周：JADX 反编译 + apktool 解包（理解 APK 结构）**

### **目标**

- 掌握 APK 文件结构
- 用 JADX 快速定位关键代码
- 理解 AndroidManifest.xml、res/ 资源的含义

---

### **Step 1.1：环境准备（Day 1）**

#### **A. 安装工具**

```bash
# Mac + Apple Silicon
brew install apktool
brew install jadx
brew install adb

# 验证安装
apktool --version
jadx --version
adb --version
```

#### **B. 获取测试 APK**

```bash
# 选项1：从真机提取（需 adb 连接）
adb devices  # 查看连接的设备
adb shell pm list packages  # 列出已安装应用

# 例如：获取 WeChat APK
adb shell pm path com.tencent.mm
# 输出：package:/data/app/com.tencent.mm-xxx/base.apk

adb pull /data/app/com.tencent.mm-xxx/base.apk ./wechat.apk

# 选项2：从 APK 应用商店下载（如 APKPure）
# 下载后保存为 test.apk
```

**推荐用测试 APK**（无加壳、无混淆）：

```bash
# 自己编写最简单的 Android App 打包
# 或用开源 App：
# https://github.com/SecUpwn/Android-InsecureBankv2
# https://github.com/tlamb96/dvga (Damn Vulnerable Google Android)
```

---

### **Step 1.2：APK 文件结构详解（Day 2）**

#### **A. 用 apktool 解包**

```bash
# 解包 APK（目录结构完整保留）
apktool d test.apk -o test_unpacked

# 进入解包目录
cd test_unpacked
tree -L 2  # 查看结构（需 brew install tree）
```

#### **B. 理解每个目录的含义**

```
test_unpacked/
├── AndroidManifest.xml          # 应用配置（权限、Activity、Service等）
├── apktool.yml                  # apktool 配置文件
├── original/
│   ├── AndroidManifest.xml      # 原始 XML（二进制格式的副本）
│   └── META-INF/                # 签名文件
├── res/                         # 资源文件（图片、布局、字符串等）
│   ├── drawable/                # 图片资源
│   ├── layout/                  # 布局 XML
│   ├── values/                  # 字符串、颜色等定义
│   └── ...
├── smali/                       # Smali 汇编代码（Dex 反汇编）
│   └── com/
│       └── example/
│           └── app/
│               ├── MainActivity.smali
│               └── ...
└── classes.dex                  # DEX 文件（虚拟机字节码）

```

#### **C. 关键文件解析**

**1) AndroidManifest.xml**

```bash
# 查看应用配置（直接解包后是 XML）
cat AndroidManifest.xml

# 示例输出：
# <manifest package="com.example.app">
#   <uses-permission android:name="android.permission.INTERNET" />
#   <application android:icon="@drawable/ic_launcher" ...>
#     <activity android:name=".MainActivity">
#       <intent-filter>
#         <action android:name="android.intent.action.MAIN" />
#       </intent-filter>
#     </activity>
#     <service android:name=".BackgroundService" />
#   </application>
# </manifest>

# 关键信息：
# - package: 应用包名
# - uses-permission: 权限声明
# - activity/service/receiver/provider: 组件声明
# - intent-filter: 隐式启动配置
```

**2) res/ 资源**

```bash
# 查看字符串定义
cat res/values/strings.xml

# 示例：
# <string name="app_name">My App</string>
# <string name="login_button">登录</string>

# 查看颜色
cat res/values/colors.xml

# 查看尺寸
cat res/values/dimens.xml
```

**3) smali/ 字节码预览**

```bash
# 查看一个简单的 Activity（先看目录结构）
ls -la smali/com/example/app/

# 打开一个 .smali 文件（不用完全理解，先看整体）
cat smali/com/example/app/MainActivity.smali | head -50
```

---

### **Step 1.3：JADX 反编译 → 快速定位代码（Day 3-4）**

#### **A. JADX GUI 打开 APK**

```bash
# 启动 JADX GUI（Mac 上）
jadx-gui test.apk

# 或直接命令行反编译
jadx -d output_java test.apk
# 输出：output_java/sources/com/example/app/*.java
```

#### **B. JADX GUI 操作实战**

|操作|快捷键 / 菜单|用途|
|---|---|---|
|搜索类/方法|`Cmd+Shift+O` (Mac)|找关键逻辑入口|
|搜索字符串|`Cmd+Shift+F`|定位硬编码、API Key|
|查看方法调用链|右键 → "Show Usage"|理解调用关系|
|查看反编译 Smali|右键 → "Show Smali"|对标字节码|
|导出为 Gradle|File → Export as Gradle Project|导入 AS 编译|

**实战：定位登录逻辑**

```bash
# 1. 在 JADX GUI 搜索敏感字符串
搜索：password / login / token / API

# 2. 找到 MainActivity.java
# 示例代码：
public class MainActivity extends AppCompatActivity {
    private EditText usernameInput;
    private EditText passwordInput;
    
    public void onLoginClick(View v) {
        String username = usernameInput.getText().toString();
        String password = passwordInput.getText().toString();
        
        // 关键：找这里的 HttpClient.post() 调用
        AuthService.login(username, password);
    }
}

# 3. 跟进 AuthService.login() 看网络请求
// 点击 login 方法名 → 按 Cmd+Down 进入该方法
public class AuthService {
    public static void login(String username, String password) {
        String encryptedPwd = AESUtil.encrypt(password);  // 密码加密方式
        HttpClient.post("/api/login", 
            new JSONObject()
                .put("username", username)
                .put("password", encryptedPwd)
        );
    }
}
```

#### **C. 看反编译代码的陷阱**

```java
// ❌ 错误理解：
// 反编译后的代码 != 原始代码
// 可能出现的问题：
// 1. 变量名被混淆成 a, b, c
// 2. 逻辑被优化，原始循环变成了位运算
// 3. 内联函数展开

// ✅ 对策：
// 1. 找原始 .smali 文件对标
// 2. 对比多个反编译器（JADX + CFR）
// 3. 用 Frida Hook 验证实际执行逻辑
```

---

### **Step 1.4：资源文件逆向（Day 4）**

#### **A. 提取图片、配置**

```bash
# 查看 so 库（native 代码）
find test_unpacked/lib -name "*.so"
# 输出：lib/arm64-v8a/libencrypt.so

# 提取签名信息
unzip -l test.apk "META-INF/*"

# 提取所有资源
cd test_unpacked/res
find . -name "*.xml" -o -name "*.png" | head -20
```

#### **B. 数据库、SharedPreferences 定位**

```bash
# 在反编译代码中搜索数据库名
# 示例：在 JADX 搜索 "getDatabase" 或 ".db"
搜索结果：
SharedPreferences prefs = 
    context.getSharedPreferences("app_config", MODE_PRIVATE);
// 实际存储位置：
// /data/data/com.example.app/shared_prefs/app_config.xml

// 数据库：
SQLiteDatabase db = 
    openOrCreateDatabase("user.db", MODE_PRIVATE, null);
// 实际存储位置：
// /data/data/com.example.app/databases/user.db
```

---

### **✅ 第一周验收标准**

- [ ] 能用 apktool 解包任意 APK
- [ ] 理解 AndroidManifest.xml 的 5 个关键标签
- [ ] 用 JADX 搜索并找到登录/网络请求逻辑
- [ ] 能区分 res 资源与 smali 代码
- [ ] 知道 MainActivity → AuthService → HttpClient 的调用链

**作业：**

```bash
# 任选一个开源 App（如 Signal、Telegram）
# 用 JADX 找出以下信息：
# 1. API 服务器地址
# 2. 使用的加密库
# 3. 数据库表结构
# 4. 关键权限声明

# 整理成文档：APK_Analysis_Report.md
```

---

## 📅 **第二周：Smali 字节码深入（对标 Java 字节码理解）**

### **目标**

- 掌握 Smali 虚拟机指令集
- 理解寄存器、方法调用、控制流
- 反混淆、去混淆的原理

---

### **Step 2.1：Smali 基础语法（Day 1-2）**

#### **A. 虚拟机寄存器模型**

```smali
# Dalvik VM 采用寄存器模型（不是栈）
# 寄存器：v0, v1, ..., vN（16位宽）

# 示例：对标 Java 代码
# Java:  int x = 5 + 3;
# Smali:
const/4 v0, 5       # v0 = 5 (const/4 表示常数在4位以内)
const/4 v1, 3       # v1 = 3
add-int v2, v0, v1  # v2 = v0 + v1
```

#### **B. 参数寄存器（p 寄存器）**

```smali
# 方法参数存储在高位寄存器
# 方法签名：void doSomething(String name, int age)
# 寄存器分配：
# p0 = this (对象方法)
# p1 = name (String)
# p2 = age (int)
# p3, p4, ... = 本地变量

.method public doSomething(Ljava/lang/String;I)V
    .registers 4  # 总寄存器数（参数+本地变量）
    
    move-object v0, p1  # v0 = name (参数拷贝到本地变量)
    move v1, p2         # v1 = age
    
    # 处理逻辑
    ...
.end method
```

#### **C. 方法调用指令**

```smali
# 虚拟调用（virtual call）- 对应 Java 实例方法
invoke-virtual {v0, v1}, Landroid/app/Activity;->onCreate(Landroid/os/Bundle;)V
# 拆解：
# - invoke-virtual: 调用类型（virtual/static/direct/super）
# - {v0, v1}: 寄存器参数列表（v0=this, v1=参数1）
# - Landroid/app/Activity;->onCreate(...): 完整方法签名
#   - L/L 前缀表示类类型
#   - L后面 = 包名+类名（. 替换为 /）
#   - 最后的 V = 返回值类型（V=void, I=int, Ljava/lang/String;=引用类型）

# 静态调用 (static call)
invoke-static {v0, v1}, Ljava/lang/Integer;->parseInt(Ljava/lang/String;)I
move-result v2  # 返回值保存到 v2

# 直接调用（private 方法）
invoke-direct {p0}, Lcom/example/App;-><init>()V  # 构造方法

# 超类调用
invoke-super {p0, p1}, Landroid/app/Activity;->onCreate(Landroid/os/Bundle;)V
```

#### **D. 类型签名对照表**

```
Smali 类型      Java 类型        示例
Z               boolean         Ljava/lang/Boolean;
B               byte            [B (字节数组)
S               short           [S
C               char            C
I               int             III (三个 int 参数)
J               long            J
F               float           F
D               double          D
V               void            ()V
Ljava/lang/String;  String      (Ljava/lang/String;)Ljava/lang/String;
[Ljava/lang/Object; Object[]   (Object[] arr)
```

---

### **Step 2.2：实战案例 - 反编译一个方法（Day 2-3）**

#### **A. 获取真实的 Smali 代码**

```bash
# 用 apktool 解包某个 App
apktool d some_app.apk -o app_output

# 找到一个关键类（比如加密工具类）
# 路径示例：app_output/smali/com/example/utils/CryptoUtil.smali

cat app_output/smali/com/example/utils/CryptoUtil.smali
```

#### **B. 完整 Smali 方法示例 & 翻译**

**原始 Java 代码（我们要反推）：**

```java
public class CryptoUtil {
    public static String encrypt(String plaintext, String key) {
        if (plaintext == null || plaintext.length() == 0) {
            return "";
        }
        
        Cipher cipher = Cipher.getInstance("AES/ECB/PKCS5Padding");
        SecretKeySpec keySpec = new SecretKeySpec(key.getBytes(), "AES");
        cipher.init(Cipher.ENCRYPT_MODE, keySpec);
        
        byte[] encryptedBytes = cipher.doFinal(plaintext.getBytes());
        return Base64.getEncoder().encodeToString(encryptedBytes);
    }
}
```

**对应的 Smali 代码：**

```smali
.class public Lcom/example/utils/CryptoUtil;
.super Ljava/lang/Object;

.method public static encrypt(Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;
    .registers 8
    
    # 参数：p0=plaintext, p1=key
    # 本地变量：v0-v7
    
    # ====== if (plaintext == null || plaintext.length() == 0) ======
    if-nez p0, :empty_string  # if (plaintext == null) goto empty_string
    
    invoke-virtual {p0}, Ljava/lang/String;->length()I
    move-result v0  # v0 = plaintext.length()
    
    if-nez v0, :empty_string  # if (v0 == 0) goto empty_string
    
    # ====== Cipher cipher = Cipher.getInstance(...) ======
    const-string v1, "AES/ECB/PKCS5Padding"
    invoke-static {v1}, Ljavax/crypto/Cipher;->getInstance(Ljava/lang/String;)Ljavax/crypto/Cipher;
    move-result-object v2  # v2 = cipher
    
    # ====== SecretKeySpec keySpec = new SecretKeySpec(...) ======
    new-instance v3, Ljavax/crypto/spec/SecretKeySpec;
    
    invoke-virtual {p1}, Ljava/lang/String;->getBytes()[B
    move-result-object v4  # v4 = key.getBytes()
    
    const-string v5, "AES"
    invoke-direct {v3, v4, v5}, Ljavax/crypto/spec/SecretKeySpec;-><init>([BLjava/lang/String;)V
    # v3 = new SecretKeySpec(v4, "AES")
    
    # ====== cipher.init(Cipher.ENCRYPT_MODE, keySpec) ======
    const/4 v4, 1  # ENCRYPT_MODE = 1
    invoke-virtual {v2, v4, v3}, Ljavax/crypto/Cipher;->init(ILjava/security/Key;)V
    
    # ====== byte[] encryptedBytes = cipher.doFinal(...) ======
    invoke-virtual {p0}, Ljava/lang/String;->getBytes()[B
    move-result-object v4  # v4 = plaintext.getBytes()
    
    invoke-virtual {v2, v4}, Ljavax/crypto/Cipher;->doFinal([B)[B
    move-result-object v4  # v4 = encryptedBytes
    
    # ====== Base64.getEncoder().encodeToString(...) ======
    invoke-static {}, Ljava/util/Base64;->getEncoder()Ljava/util/Base64$Encoder;
    move-result-object v5  # v5 = encoder
    
    invoke-virtual {v5, v4}, Ljava/util/Base64$Encoder;->encodeToString([B)Ljava/lang/String;
    move-result-object v6  # v6 = encoded string
    
    # ====== return v6 ======
    return-object v6
    
    # ====== :empty_string (label) ======
:empty_string
    const-string v6, ""
    return-object v6
.end method
```

**关键对照点：**

|Smali 指令|含义|对标 Java|
|---|---|---|
|`if-nez p0, :label`|if (p0 == null) goto label|if-nez = if not equal zero|
|`invoke-static {v1}`|静态方法调用|Cipher.getInstance()|
|`new-instance v3, Class;`|new 对象|new SecretKeySpec()|
|`invoke-direct {v3, ...}`|调用构造方法|`<init>` 是构造方法|
|`move-result-object v6`|保存方法返回值|返回值寄存器|
|`return-object v6`|返回对象|return|

---

### **Step 2.3：控制流分析（Day 3-4）**

#### **A. 条件分支指令**

```smali
# 对标 Java：if (x > 10) { ... } else { ... }

const/4 v0, 10
move v1, v0  # v1 = 10

const/16 v2, 20  # v2 = 20

# 比较指令
if-le v1, v2, :else_branch  # if (v1 <= v2) goto else_branch
                             # 即：if (10 <= 20) goto else
                             # 反向逻辑！

# if 为真时执行（v1 > v2）
const-string v3, "v1 > v2"
goto :end

:else_branch
const-string v3, "v1 <= v2"

:end
# 继续
```

#### **B. 循环指令**

```smali
# 对标 Java：
# for (int i = 0; i < 10; i++) { sum += i; }

const/4 v0, 0   # v0 = sum (初值)
const/4 v1, 0   # v1 = i (循环计数器)

:loop_start
const/4 v2, 10
if-ge v1, v2, :loop_end  # if (i >= 10) goto loop_end

# 循环体
add-int v0, v0, v1  # sum += i

# 递增
add-int/lit8 v1, v1, 1  # i++

goto :loop_start

:loop_end
# v0 = sum (结果)
```

---

### **Step 2.4：去混淆、找关键逻辑（Day 4）**

#### **A. 混淆代码示例**

```smali
# 混淆前（可读）
.method public isPasswordCorrect(Ljava/lang/String;)Z
    invoke-static {p1}, Lcom/example/utils/PasswordValidator;->validate(Ljava/lang/String;)Z
    move-result v0
    return v0
.end method

# 混淆后（难以理解）
.method public a(Ljava/lang/String;)Z
    const/4 v0, 0
    invoke-static {}, Lcom/a/b;->c()Ljava/lang/String;
    move-result-object v1
    
    invoke-virtual {p1}, Ljava/lang/String;->length()I
    move-result v2
    
    if-eq v2, v0, :cond_1
    const/4 v0, 1
    
    :cond_1
    # ... 复杂的位运算和跳转
    return v0
.end method
```

#### **B. 去混淆思路**

```bash
# 策略 1：搜索关键 API 调用（混淆不了）
搜索：
- Landroid/util/Base64;
- Ljavax/crypto/Cipher;
- Landroid/content/SharedPreferences;
- Landroid/database/sqlite/SQLiteDatabase;

# 策略 2：跟踪数据流（常用变量）
从入口方法追踪参数的使用
v0 = user input
v1 = 加密
v2 = 网络请求

# 策略 3：用 Frida 动态验证（第三周）
在关键点打 Hook，看运行时实际逻辑
```

#### **C. 实战：寻找加密 Key**

```bash
# Smali 中寻找硬编码 Key
grep -r "const-string" app_output/smali | grep -i "key\|secret\|password\|token"

# 输出示例：
# smali/com/example/crypto/KeyStore.smali:    const-string v0, "0x1a2b3c4d5e6f7a8b"

# 翻译到 Java：
String KEY = "0x1a2b3c4d5e6f7a8b";  # 找到了！
```

---

### **✅ 第二周验收标准**

- [ ] 能读懂简单的 Smali 方法（无混淆）
- [ ] 理解寄存器、参数、返回值的映射
- [ ] 掌握 invoke-virtual/static/direct 的区别
- [ ] 能从 Smali 反推出原始 Java 逻辑（50% 准确度）
- [ ] 能用 grep 在 Smali 中搜索关键 API

**作业：**

```bash
# 1. 反编译一个 App 的 CryptoUtil.smali
# 2. 逐行翻译成 Java（注释每一行）
# 3. 找出加密方式（AES/RSA/其他）
# 4. 提取 Key/IV（如果是硬编码）

# 整理成文档：Smali_Analysis.md
```

---

## 📅 **第三周：Frida Hook（实战绕过逻辑）**

### **目标**

- 掌握 Frida 框架与 Java.use() API
- 动态修改方法返回值、参数
- 绕过验证逻辑（密码、Token、反调试）

---

### **Step 3.1：Frida 环境搭建（Day 1）**

#### **A. 安装 Frida**

```bash
# Mac + Apple Silicon
brew install frida

# Python 工具
pip install frida-tools --break-system-packages

# 验证
frida --version
frida-ps --help
```

#### **B. Android 设备配置**

```bash
# 如果没有真机，用模拟器
# Android Studio 内置模拟器（推荐）
# 或 Genymotion

# 验证 adb 连接
adb devices

# 输出示例：
# List of attached devices
# emulator-5554  device
```

#### **C. 部署 Frida Server**

```bash
# 1. 下载 Frida Server（ARM64 版本）
# 从 https://github.com/frida/frida/releases 下载
# frida-server-16.x.x-android-arm64.xz

# 2. 解压
xz -d frida-server-16.x.x-android-arm64.xz

# 3. 推送到设备
adb push frida-server-16.x.x-android-arm64 /data/local/tmp/frida-server

# 4. 赋予执行权限
adb shell chmod +x /data/local/tmp/frida-server

# 5. 运行 Frida Server
adb shell /data/local/tmp/frida-server &

# 6. 验证（另开终端）
frida-ps -U
# 输出：应用列表
```

---

### **Step 3.2：基础 Hook - Java 方法拦截（Day 2）**

#### **A. 最简单的 Hook 脚本**

创建 `hook_demo.js`：

```javascript
// ===== 最基础：Hook 一个方法，打印调用 =====
Java.perform(function() {
    // 加载 Java 类
    var Activity = Java.use("android.app.Activity");
    
    // 替换 onCreate 方法
    Activity.onCreate.implementation = function(savedInstanceState) {
        console.log("[+] Activity.onCreate() 被调用了");
        console.log("[+] Bundle: " + savedInstanceState);
        
        // 调用原始方法（必须！否则 App 奔溃）
        return this.onCreate(savedInstanceState);
    };
});
```

#### **B. 运行 Hook**

```bash
# 方式1：先启动应用，再 Hook
adb shell am start -n com.example.app/.MainActivity
frida -U -l hook_demo.js

# 方式2：Frida 启动应用并 Hook（推荐）
frida -U -f com.example.app -l hook_demo.js --no-pause

# 输出示例：
# [Android Emulator 5554::com.example.app]-> 
# [+] Activity.onCreate() 被调用了
# [+] Bundle: null
```

---

### **Step 3.3：Hook 密码验证逻辑（实战案例）（Day 2-3）**

#### **A. 场景：绕过密码检查**

**原始 App 代码（想象的）：**

```java
public class LoginActivity extends Activity {
    public void onLoginClick(View v) {
        String password = passwordEditText.getText().toString();
        
        if (PasswordValidator.isValid(password)) {
            goToMainActivity();
        } else {
            showError("密码错误");
        }
    }
}

class PasswordValidator {
    public static boolean isValid(String pwd) {
        // 复杂验证逻辑
        return pwd.length() >= 8 && 
               pwd.matches(".*[A-Z].*") && 
               pwd.matches(".*[0-9].*");
    }
}
```

**Frida Hook 脚本 (`hook_bypass_password.js`)：**

```javascript
Java.perform(function() {
    // 加载密码验证类
    var PasswordValidator = Java.use("com.example.app.PasswordValidator");
    
    // 替换 isValid 方法，强制返回 true
    PasswordValidator.isValid.implementation = function(pwd) {
        console.log("[+] PasswordValidator.isValid() 被调用");
        console.log("[+] 输入密码: " + pwd);
        
        // 调用原始方法看结果
        var originalResult = this.isValid(pwd);
        console.log("[+] 原始结果: " + originalResult);
        
        // 强制返回 true（绕过验证）
        console.log("[+] 返回值已修改为 true");
        return true;
    };
});
```

**执行：**

```bash
frida -U -f com.example.app -l hook_bypass_password.js --no-pause

# 输出：
# [+] PasswordValidator.isValid() 被调用
# [+] 输入密码: 123456
# [+] 原始结果: false
# [+] 返回值已修改为 true
# 
# → App 认为密码正确，进入主页面
```

---

### **Step 3.4：Hook 网络请求（拦截 API）（Day 3）**

#### **A. Hook HttpURLConnection**

```javascript
Java.perform(function() {
    var HttpURLConnection = Java.use("java.net.HttpURLConnection");
    
    // Hook setRequestProperty（设置请求头）
    HttpURLConnection.setRequestProperty.implementation = function(key, value) {
        console.log("[+] HTTP 请求头: " + key + " = " + value);
        
        // 修改 Authorization Header
        if (key === "Authorization") {
            console.log("[!] 检测到认证头，原值: " + value);
            // 可以在这里修改 token
            // value = "Bearer fake_token_123";
        }
        
        return this.setRequestProperty(key, value);
    };
});
```

#### **B. Hook OkHttp（更常用）**

```javascript
Java.perform(function() {
    // OkHttp 拦截器
    var Request = Java.use("okhttp3.Request");
    
    Request.$init.overload("okhttp3.Request$Builder").implementation = function(builder) {
        console.log("[+] OkHttp Request 被构造");
        
        var build_result = this.$init(builder);
        var request = builder.build();
        
        console.log("[+] URL: " + request.url());
        console.log("[+] Method: " + request.method());
        
        // 打印所有请求头
        var headers = request.headers();
        for (var i = 0; i < headers.size(); i++) {
            console.log("[+] Header: " + headers.name(i) + " = " + headers.value(i));
        }
        
        return build_result;
    };
});
```

---

### **Step 3.5：Hook 数据库读写（窃取数据）（Day 3-4）**

#### **A. Hook SQLiteDatabase**

```javascript
Java.perform(function() {
    var SQLiteDatabase = Java.use("android.database.sqlite.SQLiteDatabase");
    
    // Hook query 方法（查询）
    SQLiteDatabase.query.overload(
        "java.lang.String",
        "[Ljava/lang/String;",
        "java.lang.String",
        "[Ljava/lang/String;",
        "java.lang.String",
        "java.lang.String",
        "java.lang.String"
    ).implementation = function(table, columns, where, whereArgs, groupBy, having, orderBy) {
        console.log("[+] SQLiteDatabase.query()");
        console.log("    Table: " + table);
        console.log("    Columns: " + columns);
        console.log("    Where: " + where);
        console.log("    WhereArgs: " + whereArgs);
        
        return this.query(table, columns, where, whereArgs, groupBy, having, orderBy);
    };
    
    // Hook insert 方法（插入）
    SQLiteDatabase.insert.implementation = function(table, nullColumnHack, values) {
        console.log("[+] SQLiteDatabase.insert()");
        console.log("    Table: " + table);
        console.log("    Values: " + values);
        
        return this.insert(table, nullColumnHack, values);
    };
});
```

---

### **Step 3.6：反反调试 - 绕过检测（Day 4）**

#### **A. App 常见反调试手段**

```java
// 1. 检查是否被 Frida Hook
// 2. 检查调试器是否连接
// 3. 检查是否在模拟器上运行
// 4. 检查 SELinux 是否关闭
// 5. 检查是否 root
```

#### **B. 绕过检测的 Hook 脚本 (`anti_anti_debug.js`)**

```javascript
Java.perform(function() {
    // ========== 绕过"检查调试器"==========
    var Debug = Java.use("android.os.Debug");
    
    Debug.isDebuggerConnected.implementation = function() {
        console.log("[+] isDebuggerConnected() 被调用，返回 false");
        return false;  // 骗程序说没有调试器
    };
    
    // ========== 绕过 "BuildProperties 检查" ==========
    var Build = Java.use("android.os.Build");
    var FINGERPRINT_val = Build.FINGERPRINT.value;
    
    console.log("[+] Build.FINGERPRINT: " + FINGERPRINT_val);
    // 如果是模拟器特征，可以修改（不推荐，太复杂）
    
    // ========== 绕过 ActivityManager 检查 ==========
    var ActivityManager = Java.use("android.app.ActivityManager");
    
    ActivityManager.getRunningAppProcesses.implementation = function() {
        console.log("[+] getRunningAppProcesses() 被调用");
        return this.getRunningAppProcesses();
    };
    
    // ========== 绕过 frida-server 进程检测 ==========
    // 修改进程名（高级技巧）
    var Runtime = Java.use("java.lang.Runtime");
    Runtime.exec.overload("java.lang.String").implementation = function(cmd) {
        console.log("[+] Runtime.exec: " + cmd);
        
        if (cmd.indexOf("ps") !== -1 || cmd.indexOf("grep") !== -1) {
            console.log("[!] 检测到进程检查命令，过滤掉 frida-server");
            // 实际上无法在这里过滤，需要其他方法
        }
        
        return this.exec(cmd);
    };
});
```

**执行：**

```bash
frida -U -f com.example.app -l anti_anti_debug.js --no-pause
```

---

### **✅ 第三周验收标准**

- [ ] 能启动 Frida Server 并连接设备
- [ ] 写出简单的 Hook 脚本（拦截方法调用）
- [ ] 绕过密码/Token 验证逻辑
- [ ] Hook 网络请求，查看 API 调用
- [ ] Hook 数据库，导出敏感数据
- [ ] 绕过反调试机制

**作业：**

```bash
# 1. 找一个有密码验证的 App
# 2. 用 Frida 绕过密码检查
# 3. 导出 SharedPreferences 中的 Token
# 4. 拦截网络请求，修改参数

# 整理成视频 + 脚本：Frida_Bypass_Demo.md
```

---

## 📅 **第四周：Native 代码分析（IDA Pro / Ghidra 分析 .so）**

### **目标**

- 理解 native 方法与 JNI 调用
- 用 IDA Pro/Ghidra 反汇编 .so 文件
- 找出关键加密逻辑、特征提取

---

### **Step 4.1：Native 代码基础（Day 1）**

#### **A. JNI 调用流程**

```
Java 层          JNI 桥接             Native 层
              (android-ndk)
                    
login()  ----→  nativeLogin()  ----→  login.so
  ↓              (Java_com_xxx_NativeLib_nativeLogin)
  ↑
  
String pwd
```

#### **B. 在 Java 中找 native 方法**

```java
// 在 JADX 反编译代码中搜索 "native" 关键字
public class CryptoUtil {
    // native 方法声明
    public static native String encryptData(String data);
    
    static {
        // 加载 .so 库
        System.loadLibrary("encrypt");  // 加载 libencrypt.so
    }
}
```

#### **C. 提取 .so 文件**

```bash
# 从 APK 中提取
unzip -l app.apk "lib/*"
unzip app.apk "lib/arm64-v8a/*.so" -d extracted

# 查看有哪些 .so 文件
ls -la extracted/lib/arm64-v8a/

# 输出示例：
# libencrypt.so
# libsecurity.so
# libc++_shared.so (依赖库)
```

---

### **Step 4.2：IDA Pro 基础操作（Day 1-2）**

#### **A. 获取与安装 IDA Pro**

```bash
# IDA Pro 是商业软件（昂贵）
# 替代方案（免费）：
# 1. Ghidra (NSA 开源，功能不错)
# 2. Radare2 (开源命令行工具)
# 3. Binary Ninja Community (免费社区版)

# 本教程用 Ghidra（免费）

# 下载 Ghidra
# https://github.com/NationalSecurityAgency/ghidra/releases

# Mac 上安装
unzip ghidra_xxx_mac.zip
cd ghidra
./ghidraRun
```

#### **B. 用 Ghidra 打开 .so**

```bash
# 启动 Ghidra GUI
cd ghidra
./ghidraRun

# 菜单：File → New Project
# 输入项目名：Android_Reversing
# 点击 OK

# 菜单：File → Import File
# 选择 libencrypt.so
# 点击 OK

# 自动反汇编开始（可能需要 5-10 分钟）
# 完成后，可以看到：
# - Listing 窗口：汇编代码
# - Decompile 窗口：伪 C 代码
```

---

### **Step 4.3：ARM64 汇编基础（Day 2）**

#### **A. ARM64 寄存器速查**

```
寄存器          用途                    说明
x0-x7          函数参数/返回值          x0/x1 = 前两个参数
x8-x15         调用者保存寄存器         函数可随意使用
x16-x17        临时寄存器              
x18-x28        调用者保存寄存器
x29            Frame Pointer (FP)       栈帧指针
x30            Link Register (LR)       返回地址
sp (x31)       Stack Pointer            栈顶
```

#### **B. 关键指令**

```arm64
; ============ 内存操作 ============
ldr x0, [x1]          ; x0 = *(x1)       (加载)
str x0, [x1]          ; *(x1) = x0       (存储)
ldp x0, x1, [sp]      ; x0,x1 = sp, sp+8 (双字加载)

; ============ 数学运算 ============
add x0, x1, x2        ; x0 = x1 + x2
sub x0, x1, x2        ; x0 = x1 - x2
mul x0, x1, x2        ; x0 = x1 * x2
lsl x0, x1, #2        ; x0 = x1 << 2     (逻辑左移)
lsr x0, x1, #3        ; x0 = x1 >> 3     (逻辑右移)

; ============ 比较与分支 ============
cmp x0, x1            ; 比较 x0 和 x1
b.eq label            ; if (x0 == x1) goto label
b.ne label            ; if (x0 != x1) goto label
b.gt label            ; if (x0 > x1) goto label
bl function_name      ; 调用函数（保存 LR）
ret                   ; 函数返回

; ============ 栈操作 ============
sub sp, sp, #16       ; sp -= 16          (分配栈空间)
add sp, sp, #16       ; sp += 16          (释放栈空间)
```

#### **C. 对标 C 代码**

**原始 C 代码：**

```c
int add(int a, int b) {
    return a + b;
}
```

**ARM64 汇编：**

```arm64
add:                           ; 函数标签
    add w0, w0, w1             ; w0 = w0 + w1 (w0/w1 是 32 位寄存器)
    ret                        ; 返回
```

---

### **Step 4.4：JNI 函数识别（Day 3）**

#### **A. JNI 函数命名规则**

```
Java_包名_类名_方法名

例子：
Java com.example.crypto.CryptoUtil.encrypt(String data)
     ↓
JNI Java_com_example_crypto_CryptoUtil_encrypt

Ghidra 中搜索：
ctrl+f → 搜索 "Java_com_example"
```

#### **B. 在 Ghidra 中找 JNI 函数**

```bash
# 1. 打开反汇编窗口
# 2. 菜单：Search → For Strings
# 3. 搜索：Java_

# 输出示例：
# Java_com_example_crypto_CryptoUtil_encrypt
# Java_com_example_crypto_CryptoUtil_decrypt

# 4. 双击函数名，跳转到该函数代码
```

#### **C. JNI 函数签名示例**

```c
// Java 代码
public class CryptoUtil {
    public static native String encrypt(String data);
}

// 对应的 JNI C 函数签名
JNIEXPORT jstring JNICALL
Java_com_example_crypto_CryptoUtil_encrypt(
    JNIEnv *env,           // JNI 环境
    jclass cls,            // Java 类引用
    jstring data            // Java String 参数
) {
    // C 实现
    const char *c_data = (*env)->GetStringUTFChars(env, data, NULL);
    
    // 处理逻辑
    char result[256];
    encrypt_impl(c_data, result);
    
    // 返回 Java String
    jstring result_str = (*env)->NewStringUTF(env, result);
    
    (*env)->ReleaseStringUTFChars(env, data, c_data);
    return result_str;
}
```

---

### **Step 4.5：实战案例 - 逆向加密算法（Day 3-4）**

#### **A. 场景：破解 App 的图像特征提取**

**假设 App 有一个 .so 库，提取用户头像的特征码**

**Ghidra 反汇编后的伪代码（简化）：**

```c
jstring Java_com_example_app_NativeLib_extractFeature(
    JNIEnv *env, 
    jclass cls, 
    jobject imageBitmap
) {
    // 1. 获取 Bitmap 像素
    AndroidBitmap_getInfo(env, imageBitmap, &bitmap_info);
    AndroidBitmap_lockPixels(env, imageBitmap, (void**) &pixels);
    
    int width = bitmap_info.width;
    int height = bitmap_info.height;
    
    // 2. 图像预处理（灰度化 + 缩放）
    uint8_t gray[224 * 224];
    resize_and_grayscale(pixels, gray, width, height, 224, 224);
    
    // 3. 特征提取（模型推理）
    float features[128];
    model_infer(gray, features);  // 调用深度学习模型
    
    // 4. 特征编码（Base64）
    char encoded[256];
    base64_encode((uint8_t*)features, 128, encoded);
    
    // 5. 返回 Java String
    return (*env)->NewStringUTF(env, encoded);
}
```

**Ghidra 中的分析步骤：**

```bash
# Step 1: 搜索 JNI 函数入口
搜索 "Java_com_example_app_NativeLib_extractFeature"

# Step 2: 查看函数调用
点击函数 → 右键 "Show References" → 看调了哪些库函数

# Step 3: 识别关键 API
- AndroidBitmap_getInfo
- AndroidBitmap_lockPixels
- model_infer (自定义函数)
- base64_encode

# Step 4: 跟进 model_infer 函数
双击 model_infer → 查看神经网络推理逻辑

# Step 5: 提取常数
查看输入大小 (224x224)
查看特征维度 (128)
查看是否有硬编码的 model weights（通常在 .rodata section）
```

---

### **Step 4.6：寻找加密密钥与魔数（Day 4）**

#### **A. 在 .so 中搜索关键常数**

```bash
# 在 Ghidra 中打开 Search → For Bytes
搜索常见的 magic numbers：

# AES 密钥长度（16/24/32 字节）
16, 24, 32

# 加密模式常量
# PKCS7 padding 特征
# IV 长度

# 搜索字符串（可能包含硬编码信息）
Search → For Strings
搜索：
- "AES"
- "RSA"
- "encrypt"
- "secret"
- "key"
- "iv"

# 示例结果：
# stringsection
# 0x4a0248: "AES/ECB/PKCS5Padding"
# 0x4a0260: "0x1a2b3c4d5e6f7a8b9c0d1e2f"  (可能的 Key)
```

#### **B. Hook native 函数验证**

```javascript
// Frida 脚本：Hook native 加密函数
Java.perform(function() {
    // 获取 native 方法引用
    var NativeLib = Java.use("com.example.app.NativeLib");
    
    // Hook Java 层的 native 调用
    NativeLib.encrypt.implementation = function(data) {
        console.log("[+] Native encrypt() 被调用");
        console.log("[+] 输入: " + data);
        
        var result = this.encrypt(data);
        
        console.log("[+] 输出: " + result);
        return result;
    };
});
```

---

### **✅ 第四周验收标准**

- [ ] 能用 Ghidra 打开 .so 文件并反汇编
- [ ] 理解 ARM64 基本指令（ldr/str/add/mov/bl/ret）
- [ ] 能找到 JNI 函数入口
- [ ] 能识别函数调用关系（调用图）
- [ ] 能搜索并提取硬编码常数
- [ ] 能用 Frida Hook native 函数验证

**作业（最终项目）：**

```bash
# 综合 4 周的学习，完成以下挑战：

# 1. 获取一个真实 App（如支付宝、微信小部分功能）
# 2. 用 JADX 找出关键业务逻辑
# 3. 用 apktool 提取资源和配置
# 4. 用 Smali 理解混淆代码
# 5. 用 Frida 绕过验证逻辑
# 6. 用 Ghidra 反汇编 native 加密库
# 7. 最终输出完整逆向报告

# 报告包含：
# - APK 结构分析
# - 关键类的逆向伪代码
# - 网络协议抓包
# - 数据库表结构
# - native 加密算法说明
# - 绕过路径与 POC
```

---

## 📊 **4周学习进度表**

|周|工具|核心能力|作业|难度|
|---|---|---|---|---|
|W1|JADX + apktool|APK 结构、代码定位|反编译报告|⭐|
|W2|Smali|字节码理解、去混淆|Smali 翻译|⭐⭐|
|W3|Frida|Hook、绕过、数据窃取|Hook 脚本|⭐⭐⭐|
|W4|Ghidra/IDA|native 逆向、密钥提取|完整项目|⭐⭐⭐⭐|

---

## 💡 **学习建议**

1. **循序渐进**：先用无加壳 App 练手，再深入混淆/加壳 APP
2. **实战优先**：不要死记指令，边做边学
3. **工具切换**：
    - JADX 快速定位 → Smali 深入理解 → Frida 动态验证
4. **记录笔记**：每个工具、每个指令都做一遍，形成自己的速查表
5. **复用脚本**：构建个人的 Frida Hook 库（template）

---

## 🔗 **资源链接**

|资源|链接|
|---|---|
|JADX GitHub|https://github.com/skylot/jadx|
|apktool|https://ibotpeaches.github.io/Apktool/|
|Frida 官网|https://frida.re/|
|Ghidra GitHub|https://github.com/NationalSecurityAgency/ghidra|
|ARM64 指令集|https://developer.arm.com/documentation|
|JNI 文档|https://docs.oracle.com/javase/8/docs/technotes/guides/jni/|

---

需要哪一周的详细讲解？或者想要我提供某个工具的具体代码示例？