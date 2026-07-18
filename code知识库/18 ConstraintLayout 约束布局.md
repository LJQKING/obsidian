`ConstraintLayout` 是 Android Jetpack 中非常重要的布局容器（Android ConstraintLayout），它通过“约束关系”来替代传统的嵌套布局，从而提升性能。

下面把 **ConstraintLayout 常用属性按类别系统整理一版（面试 + 实战够用）**：

---

# 一、子 View（控件）约束属性（最核心）

## 1. 基础约束（四个方向）

用于确定 View 的位置关系：

```xml
app:layout_constraintLeft_toLeftOf
app:layout_constraintLeft_toRightOf
app:layout_constraintRight_toLeftOf
app:layout_constraintRight_toRightOf

app:layout_constraintTop_toTopOf
app:layout_constraintTop_toBottomOf
app:layout_constraintBottom_toTopOf
app:layout_constraintBottom_toBottomOf
```

👉 可以理解为：

> A 的左边 = B 的右边

---

## 2. 基线对齐（文本类很常用）

```xml
app:layout_constraintBaseline_toBaselineOf
```

📌 用于 TextView / EditText 对齐文字基线

---

## 3. 中心约束（居中控制）

```xml
app:layout_constraintStart_toStartOf
app:layout_constraintStart_toEndOf
app:layout_constraintEnd_toStartOf
app:layout_constraintEnd_toEndOf
```

👉 推荐使用 Start/End（支持 RTL）

---

## 4. 居中 + 偏移（非常重要）

### 居中：

```xml
app:layout_constraintHorizontal_bias="0.5"
app:layout_constraintVertical_bias="0.5"
```

📌 0 = 靠左 / 上  
📌 1 = 靠右 / 下  
📌 0.5 = 居中

---

## 5. 宽高约束模式（关键）

```xml
android:layout_width="0dp"
android:layout_height="0dp"
```

👉 代表：

> MATCH_CONSTRAINT（由约束决定大小）

---

## 6. 宽高比例（响应式布局核心）

```xml
app:layout_constraintDimensionRatio="1:1"
app:layout_constraintDimensionRatio="16:9"
```

也可以：

```xml
app:layout_constraintDimensionRatio="W,16:9"
```

---

## 7. 链式布局（Chain）

```xml
app:layout_constraintHorizontal_chainStyle="spread"
app:layout_constraintVertical_chainStyle="packed"
```

### chainStyle 三种：

- spread（均分）
    
- spread_inside（两端贴边，中间均分）
    
- packed（整体聚合）
    

---

## 8. 权重（类似 LinearLayout weight）

```xml
app:layout_constraintHorizontal_weight="1"
app:layout_constraintVertical_weight="1"
```

👉 只有在 chain 中生效

---

## 9. Margin（约束间距）

```xml
android:layout_margin
android:layout_marginTop
android:layout_marginStart
```

⚠️ ConstraintLayout 推荐配合：

```xml
app:layout_goneMarginTop
```

👉 当 View GONE 时生效

---

# 二、GuideLine（辅助线）

```xml
app:layout_constraintGuide_begin
app:layout_constraintGuide_end
app:layout_constraintGuide_percent
```

例子：

```xml
<androidx.constraintlayout.widget.Guideline
    android:orientation="vertical"
    app:layout_constraintGuide_percent="0.5"/>
```

👉 50% 屏幕分割

---

# 三、Barrier（动态边界）

```xml
app:constraint_referenced_ids="view1,view2"
app:barrierDirection="start"
```

👉 自动取多个 View 的最大/最小边界

---

# 四、Group（统一控制多个 View）

```xml
app:constraint_referenced_ids="view1,view2,view3"
android:visibility="gone"
```

👉 一次控制多个 View 显示/隐藏

---

# 五、Placeholder（占位替换）

```xml
app:content="@id/view1"
```

👉 动态替换 UI 位置

---

# 六、Flow（流式布局）

```xml
app:constraint_referenced_ids="v1,v2,v3"
app:flow_wrapMode="chain"
```

👉 类似 Flexbox（自动换行布局）

---

# 七、MotionLayout（动画扩展）

（ConstraintLayout 的进阶）

```xml
app:layoutDescription="@xml/scene"
```

👉 用于复杂动画、手势驱动 UI

---

# 八、总结（面试高频点）

你可以这样记：

### ConstraintLayout = 4大核心能力：

1. 位置约束（top/bottom/start/end）
    
2. 尺寸控制（0dp + ratio）
    
3. 复杂布局（chain / barrier / guideline）
    
4. 动态布局（group / flow / placeholder）
    

---

如果你需要，我可以帮你再整理一版：

👉 **ConstraintLayout 源码原理（onMeasure / onLayout / Solver 机制）**  
👉 或者 **面试追问版本（为什么比 LinearLayout 快）**