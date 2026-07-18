RecyclerView 的“复用 ViewHolder”其实是系统帮你做的核心优化点，你只需要**正确写 Adapter 的 onCreateViewHolder / onBindViewHolder**，就能自动复用。

下面给你一版**标准 + 面试级写法（Java）**。

---

# 一、最标准 RecyclerView Adapter（ViewHolder 复用）

```java
public class MyAdapter extends RecyclerView.Adapter<MyAdapter.MyViewHolder> {

    private List<String> data;

    public MyAdapter(List<String> data) {
        this.data = data;
    }

    // 1. 创建 ViewHolder（只会创建少量，不会每次都调用）
    @Override
    public MyViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        View view = LayoutInflater.from(parent.getContext())
                .inflate(R.layout.item_layout, parent, false);
        return new MyViewHolder(view);
    }

    // 2. 绑定数据（复用的核心）
    @Override
    public void onBindViewHolder(MyViewHolder holder, int position) {
        String text = data.get(position);
        holder.textView.setText(text);
    }

    // 3. 数据数量
    @Override
    public int getItemCount() {
        return data.size();
    }

    // 4. ViewHolder（关键：复用的载体）
    static class MyViewHolder extends RecyclerView.ViewHolder {

        TextView textView;

        public MyViewHolder(View itemView) {
            super(itemView);
            textView = itemView.findViewById(R.id.textView);
        }
    }
}
```

---

# 二、RecyclerView “复用机制”本质（重点）

RecyclerView 内部有一个缓存池：

### 1. 屏幕外 View → 进入缓存

- item 滑出屏幕
    
- ViewHolder 被放入 RecyclerView Pool
    

### 2. 新 item 进入屏幕 → 复用旧 ViewHolder

- 不会 new ViewHolder
    
- 直接回收旧 ViewHolder
    

---

# 三、复用时最容易踩坑（面试重点）

## ❌ 1. 状态没重置（最常见 bug）

```java
if (item.isSelected()) {
    holder.checkBox.setChecked(true);
}
```

❌ 错误：没 else，会导致复用错乱

### ✅ 正确写法

```java
holder.checkBox.setChecked(item.isSelected());
```

---

## ❌ 2. click 事件重复绑定

```java
holder.itemView.setOnClickListener(...)
```

✔ 建议写法：

```java
holder.itemView.setOnClickListener(v -> {
    if (listener != null) {
        listener.onItemClick(holder.getAdapterPosition());
    }
});
```

---

## ❌ 3. 图片加载没清理（Glide/Coil）

复用导致旧图片残留

```java
Glide.with(holder.imageView)
     .load(url)
     .into(holder.imageView);
```

---

# 四、进阶：强制理解“复用过程”

RecyclerView 实际流程：

```
onCreateViewHolder()  →  创建少量 ViewHolder
        ↓
onBindViewHolder()    →  反复绑定数据
        ↓
View滑出屏幕 → 进入缓存池
        ↓
再次滑入 → 复用 ViewHolder（不重新创建）
```

---

# 五、Kotlin版本（如果你需要）

```kotlin
class MyAdapter(private val data: List<String>) :
    RecyclerView.Adapter<MyAdapter.VH>() {

    class VH(itemView: View) : RecyclerView.ViewHolder(itemView) {
        val tv: TextView = itemView.findViewById(R.id.textView)
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): VH {
        val view = LayoutInflater.from(parent.context)
            .inflate(R.layout.item_layout, parent, false)
        return VH(view)
    }

    override fun onBindViewHolder(holder: VH, position: Int) {
        holder.tv.text = data[position]
    }

    override fun getItemCount() = data.size
}
```

---

# 六、如果你想更深入（面试加分点）

我可以继续帮你讲这几个“源码级问题”：

- RecyclerView Pool（Recycler）
    
- ViewHolder 为什么不会重复创建
    
- scrap / recycle / detach 区别
    
- DiffUtil 如何优化复用
    
- payload 局部刷新原理
    
- itemAnimator 导致闪烁原因
    

只要说一句：**“源码级复用机制讲一下”** 👍