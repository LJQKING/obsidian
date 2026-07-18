# Android 设计模式——单例模式（Singleton Pattern）

单例模式是 Android 开发面试中出现频率最高的设计模式之一。

**定义：**  确保一个类在整个应用生命周期中只有一个实例，并提供全局访问入口。

---
# 为什么要用单例

例如：

- 网络请求管理器（OkHttpClient）
    
- 数据库（Room Database）
    
- SharedPreferences工具类
    
- 蓝牙管理器
    
- Retrofit实例
    
- 图片加载框架配置对象
    

这些对象：

- 创建成本高
    
- 全局只需要一个
    
- 多次创建浪费内存
    

因此适合使用单例。

---

# 单例模式结构

```java
public class Singleton {

    private static Singleton instance;

    private Singleton() {
    }

    public static Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }
}
```

使用：

```java
Singleton singleton = Singleton.getInstance();
```

---

# 问题：线程不安全

假设两个线程同时执行：

```java
if (instance == null)
```

都成立：

```java
Thread A -> 创建对象
Thread B -> 创建对象
```

最终产生多个实例。

---

# 方案一：饿汉式（线程安全）

## 实现

```java
public class Singleton {

    private static final Singleton INSTANCE =
            new Singleton();

    private Singleton() {
    }

    public static Singleton getInstance() {
        return INSTANCE;
    }
}
```

---

## 原理

类加载时：

```java
static final Singleton INSTANCE
```

已经初始化完成。

JVM保证：

```java
ClassLoader
```

只执行一次。

因此线程安全。

---

## 优点

- 实现简单
    
- 线程安全
    

---

## 缺点

应用启动即创建

即使不用：

```java
Singleton.INSTANCE
```

也会占用内存。

---

# 方案二：懒汉式（线程不安全）

```java
public class Singleton {

    private static Singleton instance;

    private Singleton() {
    }

    public static Singleton getInstance() {

        if (instance == null) {
            instance = new Singleton();
        }

        return instance;
    }
}
```

---

## 优点

需要时才创建

```java
Lazy Load
```

---

## 缺点

线程不安全

---

# 方案三：同步锁

```java
public class Singleton {

    private static Singleton instance;

    private Singleton() {
    }

    public static synchronized Singleton getInstance() {

        if (instance == null) {
            instance = new Singleton();
        }

        return instance;
    }
}
```

---

## 优点

线程安全

---

## 缺点

每次调用：

```java
getInstance()
```

都加锁。

性能较差。

---

# 方案四：DCL（双重检查锁）

Android面试最爱问。

```java
public class Singleton {

    private static volatile Singleton instance;

    private Singleton() {
    }

    public static Singleton getInstance() {

        if (instance == null) {

            synchronized (Singleton.class) {

                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }

        return instance;
    }
}
```

---

# 为什么要两次判断

第一次：

```java
if(instance == null)
```

避免每次进入锁。

第二次：

```java
synchronized
```

防止多个线程同时创建。

---

# 为什么要 volatile

下面代码实际上分三步：

```java
instance = new Singleton();
```

等价于：

```java
1. 分配内存

2. 初始化对象

3. instance指向对象
```

正常：

```java
1 -> 2 -> 3
```

JVM可能重排序：

```java
1 -> 3 -> 2
```

此时：

```java
instance != null
```

但对象还没初始化完成。

其它线程访问：

```java
NullPointerException
```

或者数据异常。

---

## volatile作用

禁止指令重排序：

```java
private static volatile Singleton instance;
```

保证：

```java
1 -> 2 -> 3
```

顺序执行。

---

# 方案五：静态内部类（推荐）

JDK推荐实现。

```java
public class Singleton {

    private Singleton() {
    }

    private static class Holder {

        private static final Singleton INSTANCE =
                new Singleton();
    }

    public static Singleton getInstance() {
        return Holder.INSTANCE;
    }
}
```

---

## 原理

调用：

```java
Singleton.getInstance()
```

时：

```java
Holder
```

才加载。

JVM保证：

```java
ClassLoader
```

加载过程线程安全。

---

## 优点

- 懒加载
    
- 线程安全
    
- 性能高
    
- 写法优雅
    

---

# 方案六：枚举单例（最安全）

```java
public enum Singleton {

    INSTANCE;

    public void test() {

    }
}
```

使用：

```java
Singleton.INSTANCE.test();
```

---

## 优点

防止：

### 反射攻击

普通单例：

```java
Constructor constructor
```

可以强制创建对象。

```java
constructor.newInstance();
```

破坏单例。

---

### 序列化攻击

```java
ObjectInputStream
```

反序列化时会创建新对象。

---

枚举天然防止：

- 反射
    
- 序列化
    

---

# Android项目中的单例实践

## Retrofit

```java
public class RetrofitManager {

    private static volatile Retrofit retrofit;

    public static Retrofit getInstance() {

        if (retrofit == null) {

            synchronized (RetrofitManager.class) {

                if (retrofit == null) {

                    retrofit = new Retrofit.Builder()
                            .baseUrl(BASE_URL)
                            .build();
                }
            }
        }

        return retrofit;
    }
}
```

---

## Room

```java
@Database(...)
public abstract class AppDatabase extends RoomDatabase {

    private static volatile AppDatabase instance;

    public static AppDatabase getInstance(Context context) {

        if (instance == null) {

            synchronized (AppDatabase.class) {

                if (instance == null) {

                    instance =
                        Room.databaseBuilder(
                            context.getApplicationContext(),
                            AppDatabase.class,
                            "app.db")
                        .build();
                }
            }
        }

        return instance;
    }
}
```

---

# Android面试高频问题

### Q1：DCL为什么要volatile？

因为：

```java
new Singleton()
```

存在指令重排序风险。

volatile保证可见性和禁止重排序。

---

### Q2：饿汉式和懒汉式区别？

|对比|饿汉式|懒汉式|
|---|---|---|
|创建时机|类加载|第一次使用|
|线程安全|是|否|
|内存占用|较高|较低|
|推荐度|★★★|★|

---

### Q3：Android最推荐哪种单例？

通常：

1. 静态内部类
    
2. DCL + volatile
    

例如：

- Room
    
- Retrofit
    
- OkHttp
    

大量源码都采用：

```java
volatile + synchronized + DCL
```

实现。

---

# 面试总结（背诵版）

单例模式用于保证一个类全局只有一个实例。常见实现包括饿汉式、懒汉式、同步锁、DCL双重检查锁、静态内部类和枚举。Android项目中最常用的是 **DCL + volatile** 和 **静态内部类**。DCL利用两次 null 判断减少锁开销，volatile 用于防止指令重排序，保证线程安全和高性能。静态内部类利用 JVM 类加载机制实现懒加载和线程安全，是最推荐的实现方式之一。