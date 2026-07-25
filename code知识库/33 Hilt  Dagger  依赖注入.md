**Hilt / Dagger** 是 Android 开发中主流的**依赖注入（Dependency Injection，简称 DI）框架**。Dagger 是基础框架，Hilt 是 Google 基于 Dagger 专门为 Android 优化的简化版，极大减少了 Dagger 的样板代码。

### 1. 作用和能解决的问题

**核心作用**：
- **自动管理对象创建和依赖关系**：不需要手动 `new` 对象或层层传递依赖。
- **对象生命周期管理**：Hilt 内置了与 Android 组件（Application、Activity、Fragment、ViewModel、Service 等）绑定的生命周期，自动创建/销毁对象，避免内存泄漏。
- **模块解耦**：类之间只依赖接口/抽象，不依赖具体实现，便于单元测试、替换实现（比如 Mock、网络切换）。
- **提高代码可维护性和可测试性**：编译时生成代码（无反射），性能高；代码更清晰、可扩展。

**能解决的具体问题**（手动 DI 或无 DI 的痛点）：
- **构造器地狱**：一个类依赖越来越多，需要层层 `new` 并传递参数，代码膨胀、难以修改。
- **生命周期管理困难**：Activity 销毁后单例对象没释放，导致内存泄漏；或频繁创建昂贵对象（网络客户端、数据库等）。
- **紧耦合**：业务类直接依赖具体实现，难以 Mock 测试或切换模块（如从 Retrofit 换成其他）。
- **样板代码多**：手动实现 Service Locator 或工厂模式，容易出错。
- **可扩展性差**：大型项目中依赖图复杂，手动维护容易引入循环依赖或运行时崩溃。

Hilt/Dagger 在**编译期**验证依赖图，提前发现问题，运行时高效。

### 2. Hilt 的基本使用（推荐新项目直接用 Hilt）

#### **步骤 1: 添加依赖（app/build.gradle.kts 或 build.gradle）**

```kotlin
plugins {
    id("com.google.dagger.hilt.android") version "2.57.1" // 检查最新版本
    id("com.google.devtools.ksp")
}

dependencies {
    implementation("com.google.dagger:hilt-android:2.57.1")
    ksp("com.google.dagger:hilt-android-compiler:2.57.1")
}
```

#### **步骤 2: 自定义 Application**

```kotlin
@HiltAndroidApp
class MyApplication : Application() {
    // 可以在这里做初始化
}
```

在 `AndroidManifest.xml` 中声明：
```xml
<application
    android:name=".MyApplication"
    ...>
```

#### **步骤 3: 在 Android 组件中注入依赖**

```kotlin
@AndroidEntryPoint  // 支持 Activity、Fragment、ViewModel、Service 等
class MainActivity : ComponentActivity() {

    @Inject
    lateinit var repository: UserRepository  // 字段注入
}
```

**ViewModel** 使用：
```kotlin
@HiltViewModel
class MyViewModel @Inject constructor(
    private val repository: UserRepository
) : ViewModel() { ... }
```

#### **步骤 4: 定义依赖提供方式**

**方式 A: 构造函数注入（最简单，推荐）**

```kotlin
class UserRepository @Inject constructor(
    private val localDataSource: UserLocalDataSource,
    private val remoteDataSource: UserRemoteDataSource
) { ... }
```

**方式 B: 使用 @Module + @Provides / @Binds（接口、第三方库、复杂构造）**

```kotlin
@Module
@InstallIn(SingletonComponent::class)  // 全局单例生命周期
object NetworkModule {

    @Provides
    @Singleton
    fun provideRetrofit(): Retrofit {
        return Retrofit.Builder()
            .baseUrl("https://api.example.com/")
            .build()
    }

    @Provides
    fun provideApiService(retrofit: Retrofit): ApiService {
        return retrofit.create(ApiService::class.java)
    }
}
```

- `@InstallIn` 决定作用域（生命周期）：
  - `SingletonComponent`：App 整个生命周期（单例）
  - `ActivityComponent`：Activity 生命周期
  - `ViewModelComponent` 等更多内置组件。

**接口绑定**（@Binds）：
```kotlin
@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {

    @Binds
    abstract fun bindUserRepository(
        impl: UserRepositoryImpl
    ): UserRepository
}
```

#### **限定符（Qualifier）** —— 解决同一类型多种实现

```kotlin
@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class AuthOkHttpClient

@Provides
@AuthOkHttpClient
fun provideAuthClient(): OkHttpClient { ... }
```

注入时用相同注解标记。

### 3. Dagger 基础（Hilt 底层）

Hilt 简化了 Dagger。如果你直接用 Dagger，需要手动定义 `@Component`、`@Subcomponent`、模块安装等，代码量大很多。Hilt 自动生成这些。

Dagger 核心概念：
- `@Inject`：标记构造函数或字段。
- `@Component`：依赖图的入口。
- `@Module` + `@Provides`：提供绑定。
- `@Scope`（如 `@Singleton`）：控制生命周期。

### 4. 最佳实践与注意事项

- **优先用构造函数注入**，减少 `@Provides`。
- **合理使用 Scope**：不要滥用 `@Singleton`，按需选择组件。
- **测试**：Hilt 提供 `@HiltAndroidTest`、`@TestInstallIn` 等，轻松替换绑定。
- **与 Jetpack** 配合极好：ViewModel、Navigation、WorkManager 等都有原生支持。
- **学习曲线**：Hilt 入门简单，大项目收益显著。

**官方推荐**：新项目直接用 **Hilt**，它保留了 Dagger 的所有优点（编译时安全、高性能），同时大幅降低上手难度。

### 更多资源
- 官方 Hilt 指南：https://developer.android.com/training/dependency-injection/hilt-android
- Dagger 文档：https://dagger.dev/

如果你有具体场景（比如注入 Room、Retrofit、ViewModel 等），或者遇到报错，可以提供代码，我可以帮你细化示例！

==**Hilt / Dagger 完整用法指南**（2026 年最新推荐以 Hilt 为主）==

以下是 Hilt 的**全面用法**整理，从基础到高级、测试、迁移、多模块等，几乎覆盖所有常见场景。Hilt 建立在 Dagger 之上，强烈推荐新项目直接使用 Hilt。

### 1. 核心注解一览

- **`@HiltAndroidApp`**：Application 类，必备，生成应用级组件。
- **`@AndroidEntryPoint`**：Activity、Fragment、Service、BroadcastReceiver、View 等 Android 组件。
- **`@HiltViewModel`**：ViewModel。
- **`@Inject`**：构造函数注入（优先）或字段注入。
- **`@Module` + `@InstallIn`**：定义模块，指定安装到哪个组件（生命周期）。
- **`@Provides`**：提供复杂对象（如 Retrofit、Room）。
- **`@Binds`**：绑定接口到实现。
- **`@Singleton`**（或其他 Scope）：作用域。
- **`@Qualifier`**：同一类型多种实现区分。
- **`@EntryPoint`**：自定义 EntryPoint（非标准组件注入）。
- **`@InstallIn`** 的内置组件（按生命周期从长到短）：
  - `SingletonComponent`（App）
  - `ActivityRetainedComponent`
  - `ActivityComponent`
  - `FragmentComponent`
  - `ViewModelComponent`
  - `ServiceComponent` 等。

### 2. 基础用法（已在上次说明，此处补充细节）

**构造函数注入**（最推荐）：

```kotlin
class Repository @Inject constructor(
    private val api: ApiService,
    private val db: AppDatabase
) { ... }
```

**字段注入**：

```kotlin
@AndroidEntryPoint
class MainActivity : ComponentActivity() {
    @Inject lateinit var repository: Repository
}
```

**Lazy / Provider 注入**（按需创建）：

```kotlin
@Inject lateinit var lazyRepo: dagger.Lazy<Repository>  // 延迟初始化
@Inject lateinit var provider: javax.inject.Provider<Repository>  // 每次 get() 新实例
```

### 3. 模块（Module）高级用法

```kotlin
@Module
@InstallIn(SingletonComponent::class)  // 全局单例
object AppModule {

    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: Context): AppDatabase {
        return Room.databaseBuilder(context, AppDatabase::class.java, "app.db").build()
    }

    @Provides
    fun provideApiService(retrofit: Retrofit): ApiService = retrofit.create(ApiService::class.java)
}
```

**@Binds 示例**：

```kotlin
@Module
@InstallIn(ViewModelComponent::class)
abstract class RepoModule {
    @Binds
    abstract fun bindRepo(impl: RealRepository): Repository
}
```

**多绑定 / Qualifier**：

```kotlin
@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class LocalDataSource

@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class RemoteDataSource

// 在 Module 中
@Provides
@LocalDataSource
fun provideLocalRepo(): UserDataSource = LocalUserDataSource()

@Provides
@RemoteDataSource
fun provideRemoteRepo(): UserDataSource = RemoteUserDataSource()

// 注入时
class Repository @Inject constructor(
    @LocalDataSource private val local: UserDataSource,
    @RemoteDataSource private val remote: UserDataSource
)
```

**Hilt 内置 Qualifier**（Context）：
- `@ApplicationContext`
- `@ActivityContext`

### 4. 自定义 Scope（高级）

默认 `@Singleton` 不够用时：

```kotlin
@Scope
@MustBeDocumented
@Retention(AnnotationRetention.RUNTIME)
annotation class ActivityRetainedScoped  // 或自定义

@Module
@InstallIn(ActivityRetainedComponent::class)
object CustomModule { ... }
```

### 5. 多模块项目（Multi-Module）

- 每个模块都可以有自己的 `Module` 并用 `@InstallIn`。
- 公共依赖放在 `:core` 或 `:di` 模块。
- 使用 `ksp` 处理所有模块的注解处理器。

### 6. 测试用法

```kotlin
@HiltAndroidTest
class MyTest {

    @get:Rule
    var hiltRule = HiltAndroidRule(this)

    @Inject
    lateinit var repository: Repository

    @TestInstallIn(
        components = [SingletonComponent::class],
        replaces = [AppModule::class]  // 替换真实模块
    )
    object TestModule {
        @Provides
        fun provideFakeRepo(): Repository = FakeRepository()
    }
}
```

- `@UninstallModules`：卸载模块。
- 测试中可轻松替换网络、数据库等。

### 7. 与其他 Jetpack 组件集成

- **Navigation**：`@AndroidEntryPoint` 用在 NavHostActivity 即可。
- **Compose**：在 `ComponentActivity` 加 `@AndroidEntryPoint`，Composable 中用 `hiltViewModel()`。
- **WorkManager**：用 `@HiltWorker`。
- **BroadcastReceiver** 等同。

### 8. 自定义组件 / EntryPoint（非标准类注入）

```kotlin
@EntryPoint
@InstallIn(SingletonComponent::class)
interface MyEntryPoint {
    fun getRepository(): Repository
}

// 使用
val entryPoint = EntryPoints.get(application, MyEntryPoint::class.java)
val repo = entryPoint.getRepository()
```

### 9. 从 Dagger 迁移到 Hilt

1. 添加 Hilt 依赖和插件。
2. 用 `@HiltAndroidApp` 替换原有 Application。
3. 逐步添加 `@AndroidEntryPoint`。
4. 把原有 `@Component` / `@Subcomponent` 转为 Hilt Module + `@InstallIn`。
5. 参考官方迁移指南。

### 10. 常见最佳实践 & 注意事项

- **最小作用域原则**：能用 `ViewModelComponent` 就不要用 `SingletonComponent`，避免内存占用。
- **避免循环依赖**：Hilt 编译时会报错。
- **性能**：所有代码编译时生成，无运行时反射。
- **调试**：开启 `hilt` 日志或查看生成的 Dagger 代码（Build 目录）。
- **与 Koin 对比**：Hilt 编译安全、性能高；Koin 更动态轻量。
- **版本**：始终使用最新稳定版（当前约 2.5x 系列）。

### 完整示例结构（推荐）

```
di/
  ├── AppModule.kt
  ├── NetworkModule.kt
  ├── RepositoryModule.kt
  └── qualifiers/...

data/
  └── repository/RealRepository.kt
```

想看**具体某个场景的完整代码**（如 Retrofit + Room + ViewModel + Repository + 测试），或者**多模块配置**、**自定义 Scope** 详细例子，告诉我，我立刻给你贴完整可运行代码！

官方中文文档推荐：
- https://developer.android.com/training/dependency-injection/hilt-android?hl=zh-cn

Hilt 几乎覆盖了 Android 所有 DI 需求，掌握后大型项目维护性会大幅提升。有问题随时问！