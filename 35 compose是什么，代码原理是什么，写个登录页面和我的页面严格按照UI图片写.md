# 一、什么是 Jetpack Compose

**Jetpack Compose** 是 Google 推出的 Android 声明式 UI 框架，用来替代 XML + View 开发模式。

传统开发：

```text
Activity
   ↓
setContentView()
   ↓
XML布局
   ↓
findViewById()
   ↓
更新View
```

Compose开发：

```text
Activity
   ↓
setContent {
    LoginScreen()
}
```

直接通过 Kotlin 函数描述 UI。

---

# 二、Compose原理

例如：

```kotlin
@Composable
fun Greeting(name: String) {
    Text(text = "Hello $name")
}
```

实际上：

```text
Greeting()
   ↓
Compose Compiler
   ↓
生成 SlotTable
   ↓
Compose Runtime
   ↓
生成 UI Tree
   ↓
Render
```

核心组件：

```text
Compose Compiler
        ↓
Compose Runtime
        ↓
Recomposer
        ↓
Snapshot State
        ↓
UI更新
```

---

## State触发重组

```kotlin
var count by remember {
    mutableStateOf(0)
}
```

点击：

```kotlin
count++
```

流程：

```text
count改变
    ↓
Snapshot检测
    ↓
Recomposer
    ↓
重新执行Composable
    ↓
局部刷新
```

只刷新：

```kotlin
Text("$count")
```

不会刷新整个页面。

这也是 Compose 比 XML 更高效的原因。

---

# 三、项目结构

按照你的UI拆分：

```text
ui
├── login
│   ├── LoginScreen
│   ├── PhoneInput
│   ├── PasswordInput
│
├── profile
│   ├── MineScreen
│   ├── Header
│   ├── UserInfo
│   ├── TabLayout
│
├── component
│   ├── SocialLoginButton
│   ├── CommonButton
```

---

# 四、登录页（严格按照第一张图）

## LoginScreen.kt

```kotlin
@Composable
fun LoginScreen() {

    var phone by remember {
        mutableStateOf("")
    }

    var password by remember {
        mutableStateOf("")
    }

    var checked by remember {
        mutableStateOf(false)
    }

    Column(
        modifier = Modifier
            .fillMaxSize()
            .background(Color.White)
            .padding(horizontal = 24.dp)
    ) {

        Spacer(modifier = Modifier.height(100.dp))

        Text(
            text = "手机号密码登录",
            fontSize = 28.sp,
            fontWeight = FontWeight.Bold
        )

        Spacer(modifier = Modifier.height(32.dp))

        Row(
            modifier = Modifier
                .fillMaxWidth()
                .height(56.dp)
                .background(
                    Color(0xFFF7F7F7),
                    RoundedCornerShape(12.dp)
                ),
            verticalAlignment = Alignment.CenterVertically
        ) {

            Text(
                "+81",
                modifier = Modifier.padding(start = 16.dp)
            )

            Text(
                "▼",
                modifier = Modifier.padding(start = 4.dp)
            )

            Divider(
                modifier = Modifier
                    .height(20.dp)
                    .width(1.dp)
                    .padding(horizontal = 8.dp)
            )

            TextField(
                value = phone,
                onValueChange = {
                    phone = it
                },
                placeholder = {
                    Text("请输入手机号")
                },
                colors = TextFieldDefaults.colors(
                    focusedContainerColor = Color.Transparent,
                    unfocusedContainerColor = Color.Transparent
                )
            )
        }

        Spacer(modifier = Modifier.height(16.dp))

        TextField(
            value = password,
            onValueChange = {
                password = it
            },
            placeholder = {
                Text("请输入密码")
            },
            visualTransformation = PasswordVisualTransformation(),
            modifier = Modifier
                .fillMaxWidth()
                .height(56.dp),
            shape = RoundedCornerShape(12.dp)
        )

        Spacer(modifier = Modifier.height(16.dp))

        Row(
            modifier = Modifier.fillMaxWidth(),
            horizontalArrangement = Arrangement.SpaceBetween
        ) {

            Text(
                "⇌ 验证码登录",
                color = Color(0xFF1565C0)
            )

            Text(
                "忘记密码",
                color = Color(0xFF1565C0)
            )
        }

        Spacer(modifier = Modifier.height(24.dp))

        Button(
            onClick = {},
            colors = ButtonDefaults.buttonColors(
                containerColor = Color(0xFFFFC1CF)
            ),
            modifier = Modifier
                .fillMaxWidth()
                .height(56.dp),
            shape = RoundedCornerShape(12.dp)
        ) {

            Text(
                "登录",
                fontSize = 20.sp
            )
        }

        Spacer(modifier = Modifier.height(20.dp))

        Row(
            verticalAlignment = Alignment.CenterVertically,
            horizontalArrangement = Arrangement.Center,
            modifier = Modifier.fillMaxWidth()
        ) {

            Checkbox(
                checked = checked,
                onCheckedChange = {
                    checked = it
                }
            )

            Text("已阅读并同意 ")

            Text(
                "用户协议",
                color = Color.Blue
            )

            Text(" 和 ")

            Text(
                "隐私政策",
                color = Color.Blue
            )
        }

        Spacer(modifier = Modifier.weight(1f))

        Row(
            modifier = Modifier.fillMaxWidth(),
            horizontalArrangement = Arrangement.SpaceEvenly
        ) {

            SocialIcon(
                Icons.Default.AccountCircle
            )

            SocialIcon(
                Icons.Default.Face
            )

            SocialIcon(
                Icons.Default.Email
            )

            SocialIcon(
                Icons.Default.MoreHoriz
            )
        }

        Spacer(modifier = Modifier.height(30.dp))
    }
}
```

---

## 社交按钮

```kotlin
@Composable
fun SocialIcon(
    imageVector: ImageVector
) {

    Box(
        modifier = Modifier
            .size(56.dp)
            .border(
                1.dp,
                Color.LightGray,
                CircleShape
            ),
        contentAlignment = Alignment.Center
    ) {

        Icon(
            imageVector,
            contentDescription = null
        )
    }
}
```

---

# 五、我的页面（严格按照第二张图）

## MineScreen.kt

```kotlin
@Composable
fun MineScreen() {

    Scaffold(
        bottomBar = {
            NavigationBar {

                NavigationBarItem(
                    selected = false,
                    onClick = {},
                    icon = {
                        Icon(Icons.Default.Home, null)
                    },
                    label = {
                        Text("首页")
                    }
                )

                NavigationBarItem(
                    selected = false,
                    onClick = {},
                    icon = {
                        Icon(Icons.Default.Person, null)
                    },
                    label = {
                        Text("朋友")
                    }
                )

                NavigationBarItem(
                    selected = false,
                    onClick = {},
                    icon = {
                        Icon(Icons.Default.Add, null)
                    },
                    label = {
                        Text("")
                    }
                )

                NavigationBarItem(
                    selected = false,
                    onClick = {},
                    icon = {
                        Icon(Icons.Default.MailOutline, null)
                    },
                    label = {
                        Text("消息")
                    }
                )

                NavigationBarItem(
                    selected = true,
                    onClick = {},
                    icon = {
                        Icon(Icons.Default.Person, null)
                    },
                    label = {
                        Text("我")
                    }
                )
            }
        }
    ) {

        Column(
            modifier = Modifier
                .fillMaxSize()
                .background(Color.Black)
        ) {

            Box(
                modifier = Modifier
                    .fillMaxWidth()
                    .height(260.dp)
            ) {

                AsyncImage(
                    model = "背景图",
                    contentDescription = null,
                    modifier = Modifier.fillMaxSize(),
                    contentScale = ContentScale.Crop
                )

                Column(
                    modifier = Modifier.padding(
                        top = 120.dp,
                        start = 20.dp
                    )
                ) {

                    Row {

                        AsyncImage(
                            model = "头像",
                            contentDescription = null,
                            modifier = Modifier
                                .size(90.dp)
                                .clip(CircleShape)
                        )

                        Spacer(
                            modifier = Modifier.width(16.dp)
                        )

                        Column {

                            Text(
                                "卯金刀",
                                color = Color.White,
                                fontSize = 28.sp
                            )

                            Text(
                                "抖音号：2184967098",
                                color = Color.Gray
                            )
                        }
                    }
                }
            }

            Row(
                modifier = Modifier
                    .fillMaxWidth()
                    .padding(20.dp),
                horizontalArrangement =
                Arrangement.SpaceEvenly
            ) {

                StatItem("0", "获赞")
                StatItem("2", "互关")
                StatItem("11", "关注")
                StatItem("3", "粉丝")
            }

            Text(
                "中华人民共和国万岁！",
                color = Color.White,
                modifier = Modifier.padding(
                    horizontal = 20.dp
                )
            )

            Spacer(
                modifier = Modifier.height(24.dp)
            )

            Row(
                horizontalArrangement =
                Arrangement.SpaceEvenly,
                modifier = Modifier.fillMaxWidth()
            ) {

                MenuItem("我的订单")
                MenuItem("我的钱包")
                MenuItem("我的互动")
                MenuItem("全部功能")
            }

            Spacer(
                modifier = Modifier.height(20.dp)
            )

            TabRow(
                selectedTabIndex = 0
            ) {

                Tab(
                    selected = true,
                    onClick = {},
                    text = {
                        Text("作品")
                    }
                )

                Tab(
                    selected = false,
                    onClick = {},
                    text = {
                        Text("日常")
                    }
                )

                Tab(
                    selected = false,
                    onClick = {},
                    text = {
                        Text("推荐")
                    }
                )

                Tab(
                    selected = false,
                    onClick = {},
                    text = {
                        Text("收藏")
                    }
                )

                Tab(
                    selected = false,
                    onClick = {},
                    text = {
                        Text("喜欢")
                    }
                )
            }

            LazyVerticalGrid(
                columns = GridCells.Fixed(3)
            ) {

                items(20) {

                    Box(
                        modifier = Modifier
                            .padding(4.dp)
                            .aspectRatio(1f)
                            .background(Color.DarkGray)
                    )
                }
            }
        }
    }
}
```

---

## 统计组件

```kotlin
@Composable
fun StatItem(
    count: String,
    title: String
) {

    Column(
        horizontalAlignment =
        Alignment.CenterHorizontally
    ) {

        Text(
            count,
            color = Color.White,
            fontWeight = FontWeight.Bold
        )

        Text(
            title,
            color = Color.Gray
        )
    }
}
```

---

## 功能菜单

```kotlin
@Composable
fun MenuItem(
    title: String
) {

    Column(
        horizontalAlignment =
        Alignment.CenterHorizontally
    ) {

        Icon(
            Icons.Default.Star,
            null,
            tint = Color.White
        )

        Spacer(
            modifier = Modifier.height(8.dp)
        )

        Text(
            title,
            color = Color.White
        )
    }
}
```

# 六、Compose 与 XML 对比

|XML|Compose|
|---|---|
|findViewById|不需要|
|RecyclerView|LazyColumn|
|Fragment|Navigation|
|LiveData|StateFlow|
|DataBinding|State|
|XML布局|Kotlin函数|

例如：

XML：

```xml
<TextView
    android:text="Hello"/>
```

Compose：

```kotlin
Text("Hello")
```

代码量通常减少 30%~50%。

---

如果你想做到**和图片 100% 像素级还原（抖音风格登录页+个人中心页）**，下一步可以继续细化：

- ConstraintLayout Compose 还原布局
    
- 渐变背景
    
- 毛玻璃（Blur）
    
- 状态栏沉浸式
    
- TabRow 下划线动画
    
- 底部导航栏抖音样式
    
- 头像加号悬浮效果
    
- LazyVerticalGrid 作品瀑布流
    
- MVVM + Navigation + Hilt 完整项目架构
    

这样可以直接达到大厂项目级实现。