**Android 加密方式：AES 与 RSA（及混合使用）**

Android 中最常用的对称加密是 **AES**（Advanced Encryption Standard），非对称加密是 **RSA**。官方推荐优先使用 **AES-256**（GCM 或 CBC 模式）。

### 1. AES（对称加密）—— 推荐用于数据加密
- **特点**：同一密钥加密/解密，速度快，适合大数据（如本地存储、文件、数据库、传输内容）。
- **推荐参数**（2026 最新官方建议）：
  - 密钥长度：**AES-256**（最安全）。
  - 模式：**GCM**（推荐，带认证，防篡改）或 **CBC**。
  - 填充：**NoPadding**（GCM 模式）或 **PKCS5Padding**（CBC）。
  - 必须使用随机 **IV**（Initialization Vector，16 字节），每次加密不同。

#### AES/GCM/NoPadding 示例（Kotlin，推荐）
```kotlin
import javax.crypto.Cipher
import javax.crypto.KeyGenerator
import javax.crypto.SecretKey
import javax.crypto.spec.GCMParameterSpec
import java.security.SecureRandom
import android.util.Base64

// 生成密钥（实际应从 KeyStore 或安全方式获取）
fun generateKey(): SecretKey {
    val keyGen = KeyGenerator.getInstance("AES")
    keyGen.init(256)
    return keyGen.generateKey()
}

// 加密
fun encrypt(plaintext: ByteArray, key: SecretKey): Pair<ByteArray, ByteArray> {
    val cipher = Cipher.getInstance("AES/GCM/NoPadding")
    val iv = ByteArray(12) // GCM 推荐 12 字节 IV
    SecureRandom().nextBytes(iv)
    val spec = GCMParameterSpec(128, iv) // 128 位认证标签
    cipher.init(Cipher.ENCRYPT_MODE, key, spec)
    val ciphertext = cipher.doFinal(plaintext)
    return Pair(iv, ciphertext) // IV + 密文需一起存储/传输
}

// 解密
fun decrypt(iv: ByteArray, ciphertext: ByteArray, key: SecretKey): ByteArray {
    val cipher = Cipher.getInstance("AES/GCM/NoPadding")
    val spec = GCMParameterSpec(128, iv)
    cipher.init(Cipher.DECRYPT_MODE, key, spec)
    return cipher.doFinal(ciphertext)
}
```

**注意**：
- IV 必须随机且唯一（不要重复使用）。
- 存储时通常将 IV + 密文 + 认证标签一起 Base64 编码。
- 避免硬编码密钥，使用 **Android Keystore** 安全存储。

#### AES/CBC/PKCS5Padding（兼容旧系统）
类似以上，但使用 `IvParameterSpec` 和 `AES/CBC/PKCS5PADDING`。GCM 更安全。

### 2. RSA（非对称加密）
- **特点**：公钥加密、私钥解密，速度慢，适合小数据（如密钥交换、签名）。
- **推荐**：RSA-2048 或更高，结合 OAEP 填充（安全）。
- 通常不直接加密大文件，而是用于加密 AES 密钥（**混合加密**）。

**混合加密流程**（最常见最佳实践）：
1. 生成随机 AES 密钥。
2. 用 RSA 公钥加密 AES 密钥。
3. 用 AES 加密实际数据。
4. 传输：加密后的 AES 密钥 + AES 密文。

### 3. 密钥管理 —— 必须使用 Android Keystore
- 直接在代码中生成/存储密钥不安全。
- 使用 `AndroidKeyStore` 生成/存储密钥，支持硬件隔离（StrongBox）。
- 示例：生成 AES 密钥并存入 Keystore。

```kotlin
val keyStore = KeyStore.getInstance("AndroidKeyStore")
keyStore.load(null)
if (!keyStore.containsAlias("my_aes_key")) {
    val keyGen = KeyGenerator.getInstance(KeyProperties.KEY_ALGORITHM_AES, "AndroidKeyStore")
    val builder = KeyGenParameterSpec.Builder("my_aes_key", KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT)
        .setBlockModes(KeyProperties.BLOCK_MODE_GCM)
        .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
        .setKeySize(256)
    keyGen.init(builder.build())
    keyGen.generateKey()
}
```

### 4. 最佳实践与注意事项
- **优先 AES-256 GCM**，避免 ECB 模式（不安全）。
- **不要硬编码密钥**，使用 Keystore + 用户密码 + PBKDF2 派生（高迭代次数）。
- **完整性**：GCM 自带认证；CBC 需要额外 HMAC。
- **性能**：AES 适合大数据，RSA 适合密钥交换。
- **库推荐**：直接用 `javax.crypto`，或 Jetpack Security（注意新版本状态）。
- **常见问题**：IV 泄露、密钥管理不当、填充攻击。始终在真实设备测试。
- **线上安全**：传输时结合 TLS 1.3。

**官方文档**：  
[Android Cryptography](https://developer.android.com/privacy-and-security/cryptography)

如果需要完整 Java/Kotlin 示例、KeyStore 详细代码、数据库加密（SQLCipher）、或特定场景（如文件加密、网络传输），请告诉我，我可以进一步提供！