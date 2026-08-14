- [[0  Kotlin  函数｜MVVM｜Jetpack｜协程｜Retrofit + OkHttp]]  协程与Flow
- [[24.1  Kotlin 协程源码面试题]] Activity Fragment Hilt依赖注入 MVVM架构 View与UI 协程与Flow 网络 蓝牙与BLE
- [[24.3  LiveData｜RxJava｜Flow 不同场景实现]] Fragment Hilt依赖注入 MVVM架构 Navigation导航 View与UI 协程与Flow 网络 蓝牙与BLE
- [[31.3 KMP Dispatcher  Repository  StateFlow vs SharedFlow]] Activity Compose Fragment Hilt依赖注入 MVVM架构 Navigation导航 View与UI 协程与Flow 网络 蓝牙与BLE
- [[Android 开发高级岗位]] Activity Compose Fragment Hilt依赖注入 MVVM架构 Navigation导航 View与UI 协程与Flow 网络 蓝牙与BLE

### 一、Navigation是什么

**Navigation** 是 Android Jetpack 中的一个导航管理组件，旨在**简化应用内页面（目的地）之间的跳转、返回栈管理、参数传递和深层链接**等复杂逻辑。它提供了一套统一的 API 和可视化工具，让开发者能够以更清晰、安全、可维护的方式构建应用导航。

#### 1. 核心价值：从“散乱跳转”到“集中管理”

在传统的 Android 开发中，页面跳转通常通过 `Intent`（Activity 间）或 `FragmentTransaction`（Fragment 间）来实现。这种方式的主要问题是**跳转逻辑分散**在各个页面中，当页面增多时，维护成本急剧上升。

Navigation 组件的核心思路是**将导航关系抽取出来，集中定义在一个“导航图”中**，由一个统一的“导航控制器”来执行所有跳转。页面只需告诉控制器“我要去哪里”，而**返回栈管理、参数传递、过渡动画**等复杂工作全部由 Navigation 组件自动处理。

#### 2. 三大核心概念

| 概念 | 作用 | 通俗理解 | 主要类型 |
| :--- | :--- | :--- | :--- |
| **NavHost（导航宿主）** | 一个**容器**，用于显示当前导航目的地的内容 | “**舞台**”，页面在这里展示 | Fragment: `NavHostFragment`<br>Compose: `NavHost` |
| **NavGraph（导航图）** | 一个 **XML 资源文件**，定义了所有目的地及其之间的连接关系 | “**地图**”，描述了所有页面和它们之间的路径 | `res/navigation/nav_graph.xml` |
| **NavController（导航控制器）** | 负责执行导航跳转、管理返回栈的核心对象 | “**导航员**”，根据地图指挥跳转 | 通过 `NavHost` 获取 |

---

### 二、核心使用方式及区别

Navigation 没有“启动”的概念，其核心是**如何定义导航关系**和**如何执行跳转**。

#### 1. 导航图的两种定义方式

| 方式 | 特点 | 适用场景 |
| :--- | :--- | :--- |
| **XML 导航图（传统方式）** | 使用 `res/navigation/*.xml` 文件，通过可视化的 **Navigation Editor** 编辑 | 基于 **View/Fragment** 的项目，是最常见的方式 |
| **代码导航图（Compose 方式）** | 在 Kotlin 代码中直接使用 `NavGraphBuilder` 的扩展函数定义目的地 | 基于 **Jetpack Compose** 的项目 |

#### 2. 跳转方式的核心区别

| 跳转方式 | 核心方法 | 特点与适用场景 |
| :--- | :--- | :--- |
| **通过 `action` ID 跳转** | `navController.navigate(R.id.action_home_to_detail)` | 基于 XML 中预定义的 `action`，**类型安全**，只能跳转到 action 指定的目标 |
| **通过 `destination` ID 直接跳转** | `navController.navigate(R.id.detailFragment)` | 不依赖预定义的 action，**更灵活**，可跳转到导航图中的任意目的地 |
| **通过深层链接 (Deep Link) 跳转** | `navController.navigate(uri)` | 支持从外部（如通知、网页链接）直接跳转到应用内指定页面 |
| **Compose 类型安全跳转** | `navController.navigate("profile/user123")` | 使用 `@Serializable` 数据类定义路由，**编译时类型安全** |

#### 3. `navigate()` 与 `NavOptions`：定制跳转行为

`NavOptions` 可以精细控制跳转行为，其核心区别如下：

| 配置项 | 作用 | 典型场景 |
| :--- | :--- | :--- |
| **`launchSingleTop`** | 如果目标目的地已在返回栈栈顶，则**复用**而不创建新实例 | 避免重复点击底部导航按钮时创建多个相同页面 |
| **`popUpTo`** | 跳转前**弹出**返回栈中直到指定目的地的所有页面 | 从详情页返回首页时，清空中间的所有页面 |
| **`popUpToInclusive`** | 与 `popUpTo` 配合，是否**同时弹出指定的目的地本身** | 登录成功后清空整个登录栈 |

#### 4. `NavigationUI`：自动集成 UI 组件

`NavigationUI` 类提供了静态方法，可以将 `NavController` 与标准的 Material Design UI 组件**自动关联**。当用户点击底部导航项或菜单项时，**无需手动设置点击监听器**，`NavigationUI` 会根据菜单项的 `id` 自动匹配导航图中的目的地并执行跳转。

| UI 组件 | 关联方法 | 自动功能 |
| :--- | :--- | :--- |
| **Toolbar / ActionBar** | `NavigationUI.setupWithNavController()` | 自动更新标题、显示“向上”按钮 |
| **BottomNavigationView** | `NavigationUI.setupWithNavController()` | 菜单项点击自动导航 |
| **NavigationView (抽屉)** | `NavigationUI.setupWithNavController()` | 菜单项点击自动导航 |
| **CollapsingToolbarLayout** | `NavigationUI.setupWithNavController()` | 自动更新标题 |

---

### 三、具体场景代码实现

以下示例基于 **View/Fragment** 和 **XML 导航图**，这是目前最主流的方式。

#### 场景一：基础跳转（首页 → 详情页）

**1. 添加依赖**

在模块的 `build.gradle` 中添加:
```groovy
dependencies {
    implementation "androidx.navigation:navigation-fragment-ktx:2.7.7"
    implementation "androidx.navigation:navigation-ui-ktx:2.7.7"
}
```

**2. 创建导航图 (`res/navigation/nav_graph.xml`)**

右键 `res` 目录 → `New` → `Android Resource File`，选择 `Resource type` 为 `Navigation`。在可视化编辑器中添加两个 Fragment 并连接:

```xml
<navigation xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:id="@+id/nav_graph"
    app:startDestination="@id/homeFragment">  <!-- 默认显示的页面 -->

    <fragment
        android:id="@+id/homeFragment"
        android:name="com.example.app.HomeFragment"
        android:label="首页">
        <action
            android:id="@+id/action_home_to_detail"
            app:destination="@id/detailFragment" />  <!-- 定义跳转动作 -->
    </fragment>

    <fragment
        android:id="@+id/detailFragment"
        android:name="com.example.app.DetailFragment"
        android:label="详情页" />
</navigation>
```

**3. Activity 布局中添加 NavHost**

```xml
<!-- activity_main.xml -->
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <androidx.appcompat.widget.Toolbar
        android:id="@+id/toolbar"
        android:layout_width="match_parent"
        android:layout_height="?attr/actionBarSize"
        android:background="?attr/colorPrimary" />

    <androidx.fragment.app.FragmentContainerView
        android:id="@+id/nav_host_fragment"
        android:name="androidx.navigation.fragment.NavHostFragment"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:defaultNavHost="true"
        app:navGraph="@navigation/nav_graph" />  <!-- 关联导航图 -->

</LinearLayout>
```

**4. Activity 中设置 Toolbar**

```kotlin
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        val toolbar = findViewById<Toolbar>(R.id.toolbar)
        setSupportActionBar(toolbar)

        // 获取 NavController
        val navHostFragment = supportFragmentManager
            .findFragmentById(R.id.nav_host_fragment) as NavHostFragment
        val navController = navHostFragment.navController

        // 将 Toolbar 与 NavController 关联，自动处理标题和返回按钮
        NavigationUI.setupWithNavController(toolbar, navController)
    }
}
```

**5. 在 HomeFragment 中触发跳转**

```kotlin
class HomeFragment : Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        // 获取 NavController
        val navController = Navigation.findNavController(view)

        view.findViewById<Button>(R.id.btn_go_detail).setOnClickListener {
            // 通过 action ID 跳转
            navController.navigate(R.id.action_home_to_detail)
        }
    }
}
```

#### 场景二：带参数传递的跳转

**1. 在导航图中为目标 Fragment 声明参数**

```xml
<fragment
    android:id="@+id/detailFragment"
    android:name="com.example.app.DetailFragment"
    android:label="详情页">
    <argument
        android:name="userId"
        app:argType="integer"
        android:defaultValue="-1" />
</fragment>
```

**2. 使用 `Safe Args` 插件安全传参**（Gradle 插件会生成类型安全的 `Directions` 和 `Args` 类）

```kotlin
// 在 HomeFragment 中
val action = HomeFragmentDirections.actionHomeToDetail(userId = 123)
navController.navigate(action)

// 在 DetailFragment 中获取参数
val args: DetailFragmentArgs by navArgs()
val userId = args.userId
```

#### 场景三：与底部导航栏集成

```kotlin
// 在 Activity 的 onCreate 中
val navController = findNavController(R.id.nav_host_fragment)
val bottomNav = findViewById<BottomNavigationView>(R.id.bottom_nav)

// 一行代码完成关联：点击底部导航项自动跳转，自动高亮
NavigationUI.setupWithNavController(bottomNav, navController)

// 配置 AppBarConfiguration，使这些底部导航目的地不显示返回按钮
val appBarConfiguration = AppBarConfiguration(
    setOf(R.id.homeFragment, R.id.profileFragment, R.id.settingsFragment)
)
NavigationUI.setupWithNavController(toolbar, navController, appBarConfiguration)
```

---

### 四、实现原理

#### 1. 工作流程

Navigation 的工作流程可以概括为:

1.  **定义**：开发者在 `NavGraph`（XML 或代码）中定义所有目的地和它们之间的连接关系。
2.  **初始化**：`NavHost`（如 `NavHostFragment`）在创建时，会读取关联的 `NavGraph`，并初始化 `NavController`。
3.  **跳转**：当调用 `navController.navigate()` 时：
    - `NavController` 根据传入的 ID 或 action，在 `NavGraph` 中查找目标目的地。
    - 自动处理 `NavOptions`（如 `popUpTo`、`launchSingleTop`）。
    - 根据目的地类型（Fragment、Activity 等），执行相应的**事务**（如 `FragmentTransaction`）。
    - 自动管理**返回栈**（Back Stack）。
4.  **状态恢复**：在配置变更（如屏幕旋转）时，`NavController` 会自动保存和恢复当前导航状态。

#### 2. 与 `FragmentTransaction` 的关系

Navigation 组件**底层仍然是使用 `FragmentTransaction` 来管理 Fragment 的切换**。但它**封装了** `FragmentTransaction` 的复杂性，自动处理了：
- **添加/替换/隐藏/显示**的逻辑选择
- **返回栈**的正确管理
- **共享元素过渡动画**的配置
- **Fragment 生命周期**的正确触发

#### 3. 生命周期与 `ViewModel` 的作用域

Navigation 组件**支持将 `ViewModel` 的作用域限定到导航图**（`NavGraph`）。这意味着，同一个导航图下的多个 Fragment 可以**共享同一个 `ViewModel` 实例**，实现数据共享，而该 `ViewModel` 的生命周期会跟随导航图的生命周期，在导航图的最后一个 Fragment 被销毁时才被清理。

---

### 五、重要补充与未来趋势

- **Navigation 3.0**：Google 在 2025 年推出了专为 Compose 全新设计的 Navigation 3.0。与旧版不同，Navigation 3 将返回栈作为**可观察状态**（Observable State）暴露给开发者，提供了更大的灵活性和控制力。
- **类型安全**：自 Navigation 2.8.0 起，Compose 和 Views 都支持使用 `@Serializable` 数据类定义**类型安全的目的地（Type-Safe Destinations）**。
- **与 Compose 的集成**：在 Compose 中，使用 `navigation-compose` 依赖，通过 `NavHost` 和 `composable()` 扩展函数定义目的地。