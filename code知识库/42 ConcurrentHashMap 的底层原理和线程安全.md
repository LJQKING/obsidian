我来深入讲解 ConcurrentHashMap 的底层原理和线程安全机制。

## 1. 整体设计思路

ConcurrentHashMap 采用**分段锁**（Segment Lock）或**哈希桶级别锁**的设计，相比 Hashtable 的全表锁，大大提高了并发性能。

### JDK 1.7 vs 1.8+ 的演进

|版本|结构|锁机制|
|---|---|---|
|JDK 1.7|Segment + HashEntry 数组|Segment 级别的分段锁|
|JDK 1.8+|Node 数组 + 链表 + 红黑树|哈希桶级别的 CAS + synchronized|

## 2. JDK 1.8+ 的数据结构

```java
// ConcurrentHashMap 的核心字段
transient volatile Node<K,V>[] table;
private transient volatile long baseCount;
private transient volatile int sizeCtl;  // -1 表示初始化中，-N 表示有 N-1 个线程在扩容

// Node 节点定义
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    volatile V val;  // volatile 保证可见性
    volatile Node<K,V> next;
}

// 树节点（红黑树）
static final class TreeNode<K,V> extends Node<K,V> {
    TreeNode<K,V> parent, left, right;
    boolean red;
}
```

## 3. 初始化过程（Lazy Initialize）

```java
public ConcurrentHashMap(int initialCapacity) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException();
    int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
               MAXIMUM_CAPACITY :
               tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
    this.sizeCtl = cap;  // 延迟初始化，标记初始容量
}

// 真正的初始化发生在第一次 put 时
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        if ((sc = sizeCtl) < 0)
            Thread.yield();  // 其他线程正在初始化，让出 CPU
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            // CAS 操作修改 sizeCtl 为 -1，标记初始化进行中
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2);  // 0.75 * n
                }
            } finally {
                sizeCtl = sc;  // 设置扩容阈值
            }
            break;
        }
    }
    return tab;
}
```

**线程安全保证**：使用 CAS 操作确保只有一个线程成功进行初始化。

## 4. PUT 操作详解

```java
public V put(K key, V value) {
    return putVal(key, value, false);
}

final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    
    int hash = spread(key.hashCode());  // 计算哈希值
    int binCount = 0;
    
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        
        // 1. 表未初始化
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        
        // 2. 目标桶为空，尝试 CAS 直接插入
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null)))
                break;  // CAS 成功，跳出循环
            // 失败则继续重试（另一个线程也在操作该桶）
        }
        
        // 3. 正在扩容，帮助扩容
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        
        // 4. 桶位置有元素，加锁处理
        else {
            V oldVal = null;
            synchronized (f) {  // 锁住链表头节点
                if (tabAt(tab, i) == f) {
                    // 4.1 链表处理
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key, value, null);
                                ++binCount;
                                break;
                            }
                        }
                    }
                    // 4.2 红黑树处理
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key, value)) == null)
                            oldVal = null;
                        else
                            oldVal = p.val;
                    }
                }
            }
            
            // 5. 链表长度超过 8，转换为红黑树
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    
    addCount(1L, binCount);  // 更新计数
    return null;
}

// CAS 操作原语
static final <K,V> Node<K,V> casTabAt(Node<K,V>[] tab, int i,
                                       Node<K,V> c, Node<K,V> v) {
    return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
}
```

**核心线程安全机制**：

- 空桶：用 **CAS** 操作原子化插入
- 有元素：用 **synchronized** 锁住链表头节点（只锁单个桶，不是全表）

## 5. GET 操作（无锁）

```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());
    
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
        
        // 直接比较第一个节点
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        // 哈希冲突或树节点，继续查找
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
        
        // 遍历链表
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}

// tabAt 使用 volatile 读
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
    return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
}
```

**关键特点**：**完全无锁**，利用 volatile 的可见性保证读到最新数据。

## 6. 扩容机制（多线程协作）

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    // stride 是每个线程处理的桶数，至少 16
    int stride = (NCPU > 1) ? (tab.length >>> 3) / NCPU : tab.length;
    
    for (int i = nextIndex;;) {
        // 多个线程可以共同参与扩容，通过 CAS 竞争处理不同的区间
        for (int bound;;) {
            // 线程安全地分配扩容任务区间
        }
    }
}
```

## 7. 线程安全保证总结

|场景|实现方式|说明|
|---|---|---|
|初始化|CAS|只有一个线程初始化成功|
|空桶插入|CAS|原子化操作，无锁|
|冲突桶操作|synchronized|锁住单个桶的头节点|
|读操作|volatile|无锁，利用可见性|
|计数更新|CounterCell + CAS|LongAdder 模式|
|扩容|CAS + synchronized|多线程协作|

## 8. size() 计数实现

```java
// 类似 LongAdder 的设计
private transient volatile long baseCount;
private transient volatile CounterCell[] counterCells;

public int size() {
    long n = sumCount();
    return ((n < 0L) ? 0 :
            (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
            (int)n);
}

final long sumCount() {
    CounterCell[] as = counterCells; CounterCell a;
    long sum = baseCount;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```

## 9. 与其他实现的对比

```
┌─────────────────┬────────────────┬──────────────────┬──────────┐
│                 │  HashMap       │  Hashtable       │ CHM      │
├─────────────────┼────────────────┼──────────────────┼──────────┤
│ 线程安全        │ 否             │ 是（全表锁）     │ 是       │
│ 锁粒度          │ -              │ 表级              │ 桶级      │
│ 并发读写        │ 否             │ 低               │ 高       │
│ Get 无锁        │ 是             │ 否               │ 是       │
│ 初始化方式      │ 急进式         │ 急进式           │ 延迟式   │
└─────────────────┴────────────────┴──────────────────┴──────────┘
```

## 10. 关键参数配置

```java
// 默认参数
static final int DEFAULT_CAPACITY = 16;
static final int MAXIMUM_CAPACITY = 1 << 30;
static final int DEFAULT_CONCURRENCY_LEVEL = 16;  // JDK 1.8 后已弃用
static final float LOAD_FACTOR = 0.75f;
static final int TREEIFY_THRESHOLD = 8;   // 链表转红黑树
static final int UNTREEIFY_THRESHOLD = 6; // 红黑树转链表
```

**总结**：ConcurrentHashMap 通过细粒度的锁（桶级别）、无锁的读操作、CAS 操作和 volatile 关键字的配合，以及多线程协作扩容等机制，实现了高效的线程安全。相比 Hashtable 的全表锁，性能提升显著。

有没有想进一步深入某个环节？比如红黑树的转换、扩容的详细流程，或者 CounterCell 的 LongAdder 实现？