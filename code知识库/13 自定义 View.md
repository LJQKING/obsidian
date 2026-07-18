Android 自定义 View 是 Android 开发里非常核心的一块，面试和实战都经常用到。可以从**基础流程 → 三大核心方法 → 自定义分类 → 实战示例 → 源码理解**来掌握。

---

# 一、自定义 View 是什么

自定义 View 就是：**自己继承 View 或 ViewGroup，重写绘制/测量/布局逻辑，实现系统控件无法满足的 UI 效果。**

常见场景：

- 自定义进度条
    
- 圆形头像
    
- 仪表盘
    
- 雷达图
    
- 波浪动画
    
- 图表组件
    

---

# 二、自定义 View 三大核心方法（必须掌握）

## 1. onMeasure（测量）

👉 决定 View 的大小

```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    int width = MeasureSpec.getSize(widthMeasureSpec);
    int height = MeasureSpec.getSize(heightMeasureSpec);

    setMeasuredDimension(width, height);
}
```

### MeasureSpec 三种模式：

- EXACTLY：确定大小（match_parent / 100dp）
    
- AT_MOST：最大值（wrap_content）
    
- UNSPECIFIED：不限制（ScrollView 内部）
    

---

## 2. onLayout（布局）

👉 只在 ViewGroup 中使用

决定子 View 的位置

```java
@Override
protected void onLayout(boolean changed, int l, int t, int r, int b) {
    // child.layout(left, top, right, bottom)
}
```

---

## 3. onDraw（绘制）

👉 核心中的核心（画 UI）

```java
@Override
protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);

    Paint paint = new Paint();
    paint.setColor(Color.RED);
    paint.setStrokeWidth(5);

    canvas.drawCircle(200, 200, 100, paint);
}
```

---

# 三、自定义 View 分类

## 1. 继承 View（最常见）

适合：单一控件绘制

- 圆形头像
    
- 进度条
    
- 图表
    

---

## 2. 继承 ViewGroup

适合：自定义布局

- FlowLayout（流式布局）
    
- TagLayout
    
- 自定义 Recycler 容器
    

---

## 3. 继承现有控件

例如：

```java
public class MyButton extends AppCompatButton
```

适合：

- 修改按钮样式
    
- 增强点击效果
    

---

# 四、完整自定义 View 示例（圆形进度条）

```java
public class CircleProgressView extends View {

    private Paint paint;
    private int progress = 0;

    public CircleProgressView(Context context, AttributeSet attrs) {
        super(context, attrs);

        paint = new Paint();
        paint.setAntiAlias(true);
        paint.setStrokeWidth(20);
        paint.setStyle(Paint.Style.STROKE);
    }

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int size = Math.min(
                MeasureSpec.getSize(widthMeasureSpec),
                MeasureSpec.getSize(heightMeasureSpec)
        );
        setMeasuredDimension(size, size);
    }

    @Override
    protected void onDraw(Canvas canvas) {
        int center = getWidth() / 2;
        int radius = center - 30;

        // 背景圆
        paint.setColor(Color.LTGRAY);
        canvas.drawCircle(center, center, radius, paint);

        // 进度弧
        paint.setColor(Color.BLUE);
        RectF rectF = new RectF(30, 30, getWidth() - 30, getHeight() - 30);

        canvas.drawArc(rectF, -90, progress * 3.6f, false, paint);
    }

    public void setProgress(int progress) {
        this.progress = progress;
        invalidate(); // 触发重绘
    }
}
```

---

# 五、invalidate vs requestLayout（面试重点）

## invalidate()

👉 只重绘（onDraw）

适合：

- 改颜色
    
- 改进度
    

---

## requestLayout()

👉 重新测量 + 布局 + 绘制

适合：

- 改大小
    
- 改位置
    

---

# 六、Canvas & Paint 核心能力

## Canvas（画布）

- drawLine
    
- drawRect
    
- drawCircle
    
- drawText
    
- drawBitmap
    
- drawPath
    

## Paint（画笔）

- 颜色
    
- 粗细
    
- 抗锯齿
    
- 填充/描边
    

---

# 七、进阶知识（面试加分）

## 1. View 的绘制流程

```
measure → layout → draw
```

## 2. 事件分发（重要）

```
dispatchTouchEvent
→ onInterceptTouchEvent (ViewGroup)
→ onTouchEvent
```

## 3. 硬件加速

Canvas 在 GPU 上绘制

---

# 八、常见面试问题

### 1. 自定义 View 流程？

测量 → 布局 → 绘制

---

### 2. wrap_content 怎么处理？

需要自己在 onMeasure 中处理 AT_MOST

---

### 3. invalidate 和 postInvalidate 区别？

- invalidate：主线程
    
- postInvalidate：子线程
    

---

### 4. View 和 ViewGroup 区别？

- View：绘制
    
- ViewGroup：布局 + 子 View 管理
    

---

# 九、如果你想继续深入（建议）

我可以帮你继续讲：

### 🔥 进阶方向

- 自定义 View 源码（ViewRootImpl）
    
- MotionEvent 事件分发源码
    
- 自定义 ViewGroup（FlowLayout）
    
- 仪表盘 / 雷达图实战
    
- 属性动画 + View 联动
    
- Matrix / Shader 高级绘制
    

==**自定义 View 画“表格**==
# 一、核心思路

表格 = 网格结构

- 横线：row + 1 条
    
- 竖线：column + 1 条
    
- 内容：每个 cell 居中绘制 text
    

---

# 二、自定义 View 实现（Java）

## 1️⃣ TableView.java

```java
public class TableView extends View {

    private Paint linePaint;
    private Paint textPaint;

    private int rows = 5;
    private int cols = 4;

    private String[][] data;

    public TableView(Context context) {
        super(context);
        init();
    }

    public TableView(Context context, AttributeSet attrs) {
        super(context, attrs);
        init();
    }

    private void init() {
        linePaint = new Paint();
        linePaint.setColor(Color.BLACK);
        linePaint.setStrokeWidth(3);
        linePaint.setStyle(Paint.Style.STROKE);

        textPaint = new Paint();
        textPaint.setColor(Color.BLACK);
        textPaint.setTextSize(40);
        textPaint.setAntiAlias(true);

        // 初始化数据
        data = new String[rows][cols];
        for (int i = 0; i < rows; i++) {
            for (int j = 0; j < cols; j++) {
                data[i][j] = "R" + i + "C" + j;
            }
        }
    }

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int width = MeasureSpec.getSize(widthMeasureSpec);
        int height = MeasureSpec.getSize(heightMeasureSpec);
        setMeasuredDimension(width, height);
    }

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);

        int width = getWidth();
        int height = getHeight();

        float cellWidth = width * 1f / cols;
        float cellHeight = height * 1f / rows;

        drawGrid(canvas, width, height, cellWidth, cellHeight);
        drawText(canvas, cellWidth, cellHeight);
    }

    private void drawGrid(Canvas canvas, float width, float height,
                          float cellWidth, float cellHeight) {

        // 画横线
        for (int i = 0; i <= rows; i++) {
            float y = i * cellHeight;
            canvas.drawLine(0, y, width, y, linePaint);
        }

        // 画竖线
        for (int j = 0; j <= cols; j++) {
            float x = j * cellWidth;
            canvas.drawLine(x, 0, x, height, linePaint);
        }
    }

    private void drawText(Canvas canvas, float cellWidth, float cellHeight) {

        Paint.FontMetrics fm = textPaint.getFontMetrics();

        float textHeight = fm.descent - fm.ascent;

        for (int i = 0; i < rows; i++) {
            for (int j = 0; j < cols; j++) {

                String text = data[i][j];

                float x = j * cellWidth + cellWidth / 2;
                float y = i * cellHeight + cellHeight / 2
                        + textHeight / 4;

                float textWidth = textPaint.measureText(text);

                canvas.drawText(text,
                        x - textWidth / 2,
                        y,
                        textPaint);
            }
        }
    }
}
```

---

# 三、XML 使用

```xml
<com.xxx.TableView
    android:layout_width="match_parent"
    android:layout_height="300dp"/>
```

---

# 四、效果特点

✔ 自动平均分列  
✔ 自动绘制网格  
✔ 文字居中  
✔ 支持动态数据

---

# 五、进阶优化（面试加分点🔥）

## 1️⃣ 支持自定义列宽

```java
float[] colWidths
```

替代平均分

---

## 2️⃣ 支持点击单元格

```java
@Override
public boolean onTouchEvent(MotionEvent event) {
    int col = (int)(event.getX() / cellWidth);
    int row = (int)(event.getY() / cellHeight);
}
```

---

## 3️⃣ 支持滚动（重要）

- 用 `Scroller`
    
- 或直接继承 `HorizontalScrollView + View`
    

---

## 4️⃣ 性能优化

- Paint 只初始化一次
    
- 避免 onDraw new 对象
    
- 数据变化用 `invalidate()`
    

---

# 六、面试回答模板（很关键）

如果面试官问：

### 👉 “自定义View画表格怎么做？”

你可以这样答：

> 使用 Canvas 在 onDraw 中绘制网格线（横线+竖线），根据 View 宽高计算单元格大小，然后遍历二维数组绘制文本，通过 Paint.measureText 和 FontMetrics 实现文本居中，最后通过 invalidate 更新数据。

---

# 七、如果你想继续进阶👇

我可以帮你升级成：

- 📊 类 Excel 可滑动表格
    
- 📌 RecyclerView + 自定义表格列
    
- 📈 支持冻结首行/首列
    
- 🎯 类似“钉钉审批表”UI
    

直接说你要哪种版本 👍