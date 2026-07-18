
### 1. **View 动画（Tween Animation）**

**类型：**

- `TranslateAnimation` - 平移
- `ScaleAnimation` - 缩放
- `RotateAnimation` - 旋转
- `AlphaAnimation` - 透明度
- `AnimationSet` - 组合

**能干什么：**

- 改变 View 在屏幕上的**视觉位置**（但不改变真实坐标）
- 轻量级，性能好，但只是"看起来移动"

**关键参数：**

```kotlin
// 平移 (起点 → 终点)
TranslateAnimation(
    fromXType, fromXValue,  // 起始X坐标
    toXType, toXValue,      // 终点X坐标
    fromYType, fromYValue,
    toYType, toYValue
)

// 坐标类型（三种）：
// Animation.ABSOLUTE - 绝对像素值
// Animation.RELATIVE_TO_SELF - 相对自身 (0.0 ~ 1.0)
// Animation.RELATIVE_TO_PARENT - 相对父布局 (0.0 ~ 1.0)

// 例子：向右移动自身宽度
TranslateAnimation(0f, 1f, 0f, 0f)  // X轴相对自身移动100%
```

**缩放参数：**

```kotlin
ScaleAnimation(
    fromX, toX,      // X轴缩放因子 (1.0 = 原大小)
    fromY, toY,      // Y轴缩放因子
    pivotXType, pivotXValue,  // 缩放中心X
    pivotYType, pivotYValue   // 缩放中心Y
)

// 例子：从中心点放大到2倍
ScaleAnimation(1f, 2f, 1f, 2f, 
    Animation.RELATIVE_TO_SELF, 0.5f,
    Animation.RELATIVE_TO_SELF, 0.5f)
```

**通用参数：**

```kotlin
animation.apply {
    duration = 500              // 动画时长（毫秒）
    startOffset = 100           // 开始延迟
    repeatCount = 0             // 重复次数 (-1 = 无限)
    repeatMode = Animation.RESTART  // RESTART 或 REVERSE
    interpolator = AccelerateInterpolator()  // 加速器
    fillAfter = true            // 动画结束后保持最终状态
    setAnimationListener(object : Animation.AnimationListener {
        override fun onAnimationStart(animation: Animation?) {}
        override fun onAnimationEnd(animation: Animation?) {}
        override fun onAnimationRepeat(animation: Animation?) {}
    })
}
```

### 2. **属性动画（Property Animation）**

**主要类：**

- `ObjectAnimator` - 直接改变对象属性
- `ValueAnimator` - 只生成数值，需要自己赋值
- `AnimatorSet` - 组合动画

**能干什么：**

- **真正改变对象属性**（如真实坐标、背景色等）
- 可以作用于任何对象，不仅是 View
- 更灵活，性能开销更大

**ObjectAnimator 参数：**

```kotlin
ObjectAnimator.ofFloat(
    view,           // 目标对象
    "translationX", // 属性名（必须有setter）
    0f, 500f        // 起始值 ~ 终点值
).apply {
    duration = 1000
    interpolator = BounceInterpolator()
    start()
}

// 常用属性：
// translationX, translationY - 平移
// scaleX, scaleY - 缩放
// rotationX, rotationY, rotation - 旋转
// alpha - 透明度
// backgroundColor - 背景色
```

**ValueAnimator（自定义属性）：**

```kotlin
ValueAnimator.ofFloat(0f, 100f).apply {
    duration = 1000
    addUpdateListener { animator ->
        val value = animator.animatedValue as Float
        // 自己应用这个值
        view.customProperty = value
    }
    start()
}
```

**组合动画：**

```kotlin
AnimatorSet().apply {
    val anim1 = ObjectAnimator.ofFloat(view, "translationX", 0f, 500f)
    val anim2 = ObjectAnimator.ofFloat(view, "scaleX", 1f, 1.5f)
    
    playTogether(anim1, anim2)      // 同时播放
    // playSequentially(anim1, anim2) // 顺序播放
    
    // 更精细的组合
    play(anim1).with(anim2)          // anim1和anim2同时
    play(anim1).before(anim2)        // anim1结束后anim2
    play(anim1).after(anim2)         // anim1在anim2之后
    
    duration = 1000
    start()
}
```

### 3. **过渡动画（Transition）**- API 19+

**类型：**

- `ChangeBounds` - 布局边界变化
- `ChangeImageTransform` - 图片缩放
- `ChangeScroll` - 滚动位置
- `Fade` - 淡入淡出
- `Slide` - 滑动

**能干什么：**

- Activity/Fragment 进出场景
- Shared Element Transition（共享元素）
- 自动追踪布局变化

**代码示例：**

```kotlin
// Activity 转换
val options = ActivityOptions.makeSceneTransitionAnimation(
    this,
    view,           // 共享元素
    "shared"        // 转换名称
)
startActivity(intent, options.toBundle())

// Fragment 转换
val transition = Fade().apply {
    duration = 500
}
exitTransition = transition
enterTransition = transition
```

### 4. **Drawable 动画（Frame Animation）**

**能干什么：**

- 逐帧播放一系列图像
- 类似 GIF 动画

```xml
<!-- res/drawable/animation.xml -->
<animation-list xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:drawable="@drawable/frame1" android:duration="100" />
    <item android:drawable="@drawable/frame2" android:duration="100" />
</animation-list>
```

```kotlin
val drawable = imageView.drawable as AnimationDrawable
drawable.start()
```

## 插值器（Interpolator）对比

```kotlin
// 线性
LinearInterpolator()

// 加速/减速
AccelerateInterpolator()      // 加速
DecelerateInterpolator()      // 减速
AccelerateDecelerateInterpolator()  // 先加后减

// 弹性效果
BounceInterpolator()          // 反弹
OvershootInterpolator()       // 超出后回退

// 预设
CycleInterpolator(2f)         // 循环N次
AnticipateInterpolator()      // 先回退后前进
```

## 选择建议

|场景|选择|理由|
|---|---|---|
|简单的 UI 动效|View Animation|性能好，代码简洁|
|改变真实属性|Property Animation|需要点击交互|
|Activity 进出|Transition|系统集成好|
|逐帧动画|Drawable Animation|少用，容易OOM|

你这个问题更适合通过深度代码示例理解。要我详解某个具体动画的**源码原理**或者**性能对比**吗？