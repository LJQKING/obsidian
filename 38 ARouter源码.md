# ARouter源码级详解

ARouter 是阿里开源的组件化路由框架，核心作用：

```text
页面跳转
参数传递
组件解耦
服务发现
跨模块调用
```

很多开发者只会用：

```java
ARouter.getInstance()
       .build("/user/main")
       .navigation();
```

但高级面试通常会问：

```text
ARouter为什么比反射快？

ARouter如何找到目标Activity？

ARouter如何自动注入参数？

APT生成了什么代码？

Warehouse是什么？

LogisticsCenter作用是什么？
```

---

# 一、为什么需要ARouter

传统方式：

```java
Intent intent =
    new Intent(this, UserActivity.class);

startActivity(intent);
```

问题：

```text
模块间强依赖

组件化困难

跨模块调用困难
```

---

ARouter：

```java
ARouter.getInstance()
       .build("/user/main")
       .navigation();
```

模块间无需直接依赖。

---

# 二、ARouter整体架构

```text
编译期
   ↓
APT扫描 @Route

   ↓
生成路由表

   ↓
运行期加载

   ↓
Warehouse缓存

   ↓
navigation()

   ↓
目标页面
```

---

# 三、Route注解

Activity：

```java
@Route(path = "/user/main")
public class UserActivity
        extends AppCompatActivity {

}
```

---

APT扫描：

```java
@Route
```

生成：

```java
ARouter$$Group$$user
```

---

# 四、APT生成代码

生成：

```java
public class ARouter$$Group$$user
        implements IRouteGroup {

    @Override
    public void loadInto(
        Map<String, RouteMeta> atlas) {

        atlas.put(
            "/user/main",

            RouteMeta.build(
                RouteType.ACTIVITY,
                UserActivity.class,
                "/user/main",
                "user",
                0,
                0
            )
        );
    }
}
```

---

RouteMeta：

```java
public final class RouteMeta {

    private RouteType type;

    private Class<?> destination;

    private String path;

    private String group;
}
```

保存：

```text
路径

对应Class

分组信息
```

---

# 五、Root节点

APT还生成：

```java
ARouter$$Root$$app
```

源码：

```java
public class ARouter$$Root$$app
       implements IRouteRoot {

    @Override
    public void loadInto(
       Map<String,
       Class<? extends IRouteGroup>> routes) {

       routes.put(
           "user",
           ARouter$$Group$$user.class
       );
    }
}
```

---

结构：

```text
Root
 ↓
Group
 ↓
RouteMeta
 ↓
Activity
```

---

# 六、初始化流程

Application：

```java
ARouter.init(application);
```

源码：

```java
LogisticsCenter.init(context);
```

---

扫描：

```text
com.alibaba.android.arouter.routes
```

包下所有：

```text
ARouter$$Root$$*
```

类。

---

保存到：

```java
Warehouse.groupsIndex
```

---

# 七、Warehouse

ARouter核心缓存中心。

源码：

```java
public class Warehouse {

    static Map<String,
            Class<? extends IRouteGroup>>
            groupsIndex;

    static Map<String,
            RouteMeta>
            routes;

    static Map<Class,
            IProvider>
            providers;
}
```

---

缓存结构：

```text
groupsIndex

"user"
  ↓
ARouter$$Group$$user
```

---

routes

```text
/user/main
   ↓
RouteMeta
```

---

providers

```text
LoginService
    ↓
LoginServiceImpl
```

---

# 八、navigation流程

代码：

```java
ARouter.getInstance()
       .build("/user/main")
       .navigation();
```

---

源码流程：

```text
build()
 ↓
Postcard

navigation()
 ↓
LogisticsCenter.completion()

 ↓
Warehouse

 ↓
RouteMeta

 ↓
startActivity()
```

---

源码：

```java
protected Object navigation(
        Context context,
        Postcard postcard,
        int requestCode,
        NavigationCallback callback)
```

---

核心：

```java
LogisticsCenter.completion(
        postcard);
```

---

# 九、LogisticsCenter

ARouter最核心类。

职责：

```text
路由查找

路由填充

Provider查找
```

---

源码：

```java
public synchronized static void completion(
        Postcard postcard)
```

---

查找：

```java
RouteMeta routeMeta =
    Warehouse.routes.get(path);
```

---

如果为空：

```java
动态加载Group
```

```java
group.loadInto(
    Warehouse.routes
);
```

---

然后再次查找。

---

# 十、参数自动注入

页面：

```java
@Route(path="/user/main")
public class UserActivity {

    @Autowired
    String userId;
}
```

---

跳转：

```java
ARouter.getInstance()
       .build("/user/main")
       .withString("userId","1001")
       .navigation();
```

---

注入：

```java
ARouter.getInstance()
       .inject(this);
```

---

APT生成：

```java
public class UserActivity$$Autowired {

    public void inject(
        Object target) {

        UserActivity substitute =
            (UserActivity) target;

        substitute.userId =
            substitute.getIntent()
            .getStringExtra("userId");
    }
}
```

---

本质：

```text
编译时代码生成

替代反射
```

---

# 十一、Provider服务发现

定义：

```java
public interface LoginService {

    boolean isLogin();
}
```

实现：

```java
@Route(path="/service/login")
public class LoginServiceImpl
        implements LoginService {

}
```

---

获取：

```java
LoginService service =
    (LoginService)
    ARouter.getInstance()
           .navigation(LoginService.class);
```

---

流程：

```text
Provider
 ↓
Warehouse.providers
 ↓
单例缓存
```

---

# 十二、为什么ARouter快

很多人回答：

```text
因为用了缓存
```

不完整。

---

传统反射：

```java
Class.forName()
```

↓

```java
newInstance()
```

每次执行。

---

ARouter：

编译期：

```text
APT生成映射表
```

运行期：

```text
HashMap查找
```

---

复杂度：

```text
反射
O(n)

HashMap
O(1)
```

---

# 十三、APT与ASM

面试高频。

---

APT

Annotation Processing Tool

编译期：

```java
@Route
```

↓

```java
生成Java代码
```

---

ASM

字节码修改

直接操作：

```text
.class文件
```

---

例如：

```text
组件自动注册

Plugin注册

埋点
```

很多大厂：

```text
ARouter + ASM
```

组合使用。

---

# 十四、ARouter源码面试题

### Q1 ARouter为什么不用反射？

答：

```text
APT编译期生成路由表

运行期HashMap查找

避免大量反射开销
```

---

### Q2 Warehouse是什么？

答：

```text
ARouter路由仓库

缓存：

groupsIndex
routes
providers
```

---

### Q3 LogisticsCenter作用？

答：

```text
路由中心

负责：

路由加载
路由匹配
参数填充
Provider查找
```

---

### Q4 navigation流程？

```text
build()

↓

Postcard

↓

LogisticsCenter

↓

Warehouse

↓

RouteMeta

↓

Activity跳转
```

---

# 面试总结

ARouter源码最重要的 5 个类：

```text
★★★★★
ARouter

★★★★★
LogisticsCenter

★★★★★
Warehouse

★★★★★
Postcard

★★★★★
RouteMeta
```

源码流程图：

```text
@Route
   ↓
APT
   ↓
ARouter$$Root
   ↓
ARouter$$Group
   ↓
Warehouse
   ↓
LogisticsCenter
   ↓
Postcard
   ↓
navigation()
   ↓
Activity
```

掌握到这个深度，基本可以应对高级 Android 面试中关于 ARouter 的源码追问。