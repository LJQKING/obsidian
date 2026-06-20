# Android MVC、MVP、MVVM 框架原理及区别

这三个架构模式本质上都是为了解决：

> **UI层、业务逻辑层、数据层耦合严重，导致代码难维护、难测试的问题。**

---

# 一、MVC（Model-View-Controller）

## 架构图

```text
┌─────────┐
│  View   │
└────┬────┘
     │
     ▼
┌─────────┐
│Controller│
└────┬────┘
     │
     ▼
┌─────────┐
│  Model  │
└─────────┘
```

---

## 各层职责

### Model（数据层）

负责：

- 网络请求
    
- 数据库操作
    
- 数据封装
    

例如：

```java
public class UserModel {

    public User getUser() {
        return new User("Tom");
    }
}
```

---

### View（视图层）

负责：

```java
Activity
Fragment
XML布局
```

展示UI。

---

### Controller（控制层）

负责：

- 接收用户操作
    
- 调用Model
    
- 更新View
    

Android早期：

```java
Activity
```

既是：

```text
View + Controller
```

---

## MVC代码示例

```java
public class MainActivity extends AppCompatActivity {

    TextView tvName;
    UserModel model;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        model = new UserModel();

        User user = model.getUser();

        tvName.setText(user.getName());
    }
}
```

---

## MVC问题

Activity中：

```java
网络请求
数据库
业务逻辑
UI更新
```

全写一起。

最终：

```java
MainActivity.java
3000+
5000+
8000+
行
```

业内称：

```text
God Activity（上帝类）
```

---

# 二、MVP（Model-View-Presenter）

Google早期推荐架构。

---

## 架构图

```text
          ┌─────────┐
          │  Model  │
          └────┬────┘
               │
               ▼
┌─────────┐ ◄────► ┌─────────┐
│  View   │        │Presenter│
└─────────┘        └─────────┘
```

---

## 核心思想

把业务逻辑从Activity抽离。

---

## View

一般：

```java
Activity
Fragment
```

定义接口：

```java
public interface UserView {

    void showUser(String name);

    void showLoading();
}
```

---

## Presenter

核心控制层。

```java
public class UserPresenter {

    private UserView view;
    private UserModel model;

    public UserPresenter(UserView view) {
        this.view = view;
        model = new UserModel();
    }

    public void getUser() {

        User user = model.getUser();

        view.showUser(user.getName());
    }
}
```

---

## Activity

```java
public class MainActivity extends AppCompatActivity
        implements UserView {

    UserPresenter presenter;

    @Override
    protected void onCreate(Bundle savedInstanceState) {

        presenter = new UserPresenter(this);

        presenter.getUser();
    }

    @Override
    public void showUser(String name) {
        tvName.setText(name);
    }
}
```

---

# MVP调用流程

```text
用户点击按钮
      │
      ▼
Activity(View)
      │
      ▼
Presenter
      │
      ▼
Model
      │
      ▼
Presenter
      │
      ▼
View更新UI
```

---

## MVP优点

### 1. 解耦

Activity变轻。

```java
只负责UI
```

---

### 2. 易测试

Presenter不依赖Android。

JUnit直接测试：

```java
@Test
public void testGetUser() {

}
```

---

## MVP缺点

### 1. 接口过多

一个页面：

```text
LoginView
LoginPresenter

UserView
UserPresenter

HomeView
HomePresenter
```

几十个页面：

```text
大量模板代码
```

---

### 2. Presenter容易膨胀

```java
LoginPresenter
2000+
3000+
行
```

又变成：

```text
God Presenter
```

---

# 三、MVVM（Model-View-ViewModel）

目前Android官方推荐架构。

Android Jetpack核心：

- ViewModel
    
- LiveData
    
- DataBinding
    
- StateFlow
    
- Compose
    

---

## 架构图

```text
┌───────────┐
│   View    │
└─────┬─────┘
      │观察
      ▼
┌───────────┐
│ ViewModel │
└─────┬─────┘
      │
      ▼
┌───────────┐
│   Model   │
└───────────┘
```

---

# 核心思想

使用：

```java
观察者模式
```

替代：

```java
Presenter主动更新View
```

---

# ViewModel

```java
public class UserViewModel
        extends ViewModel {

    MutableLiveData<String> userName
            = new MutableLiveData<>();

    public void getUser() {

        userName.setValue("Tom");
    }
}
```

---

# Activity

```java
UserViewModel vm;

vm.userName.observe(this, name -> {

    tvName.setText(name);

});
```

---

# 数据流

```text
View
 ↓
ViewModel
 ↓
Model
 ↓
LiveData
 ↓
View自动刷新
```

---

# MVVM源码原理

## LiveData

观察者模式。

```text
LiveData
      │
      ▼
Observer
```

源码结构：

```java
LiveData
   ├─ Observer
   ├─ ObserverWrapper
   ├─ SafeIterableMap
   └─ LifecycleBoundObserver
```

---

### observe()

```java
liveData.observe(this, observer);
```

源码：

```java
mObservers.put(observer);
```

保存观察者。

---

### setValue()

```java
liveData.setValue(data);
```

源码：

```java
dispatchingValue();
```

通知所有Observer。

---

### Observer收到通知

```java
observer.onChanged(data);
```

UI自动刷新。

---

# ViewModel源码原理

## 为什么屏幕旋转不丢数据？

Activity重建：

```java
onDestroy()
onCreate()
```

重新执行。

---

但ViewModel不会立即销毁。

系统：

```java
ActivityThread
      ↓
ViewModelStore
      ↓
HashMap
```

缓存ViewModel。

源码：

```java
HashMap<String, ViewModel>
```

---

获取：

```java
new ViewModelProvider(this)
```

实际：

```java
从ViewModelStore取
```

不是重新new。

---

# MVC、MVP、MVVM区别

|对比项|MVC|MVP|MVVM|
|---|---|---|---|
|View更新|Controller|Presenter|数据绑定|
|耦合度|高|中|低|
|测试性|差|好|很好|
|Activity代码量|非常多|少|最少|
|学习成本|低|中|高|
|开发效率|高|中|高|
|官方推荐|❌|❌|✅|

---

# Android发展历程

```text
Android 4.x
     ↓
MVC

Android 5.x~7.x
     ↓
MVP

Android 8.x+
     ↓
MVVM + Jetpack

Android 12+
     ↓
MVVM + Compose
```

---

# 面试高频问题

### Q1：为什么MVVM比MVP更流行？

答：

因为MVVM利用：

- LiveData
    
- Flow
    
- DataBinding
    

实现自动更新UI。

ViewModel不持有View引用：

```java
不会出现Presenter持有Activity导致内存泄漏
```

耦合更低。

---

### Q2：ViewModel为什么不会因为旋转屏幕销毁？

答：

因为ViewModel存储在：

```java
ViewModelStore
```

中。

Activity重建时会重新获取同一个ViewModel实例。

---

### Q3：LiveData为什么感知生命周期？

核心源码：

```java
LifecycleOwner
LifecycleObserver
```

LiveData内部监听：

```java
ON_START
ON_STOP
ON_DESTROY
```

生命周期。

页面销毁：

```java
自动移除Observer
```

避免内存泄漏。

---

# Android高级工程师面试答案

如果面试官问：

> 项目中为什么选择MVVM？

可以这样回答：

> MVVM通过 ViewModel 将 View 和 Model 解耦，利用 LiveData/StateFlow 实现响应式数据更新。ViewModel 不持有 Activity 或 Fragment 引用，降低了内存泄漏风险。同时结合 Repository、Retrofit、Room、Coroutine 可以形成完整的 Jetpack 架构体系，代码更易维护、扩展和单元测试，因此目前主流 Android 项目基本采用 MVVM 架构。

对于现代 Android 项目，推荐架构通常是：

```text
UI(Compose/Fragment)
        ↓
ViewModel
        ↓
Repository
      ↙     ↘
Remote      Local
Retrofit    Room
```

这是目前大厂（如 Google 官方推荐）的主流架构方案。

![[Pasted image 20260616160427.png]]