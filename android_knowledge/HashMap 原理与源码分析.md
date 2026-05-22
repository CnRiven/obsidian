# HashMap 原理与源码分析

> HashMap 是 Android 面试中最常问的数据结构之一，必须深入理解其底层原理

---

## 一、底层数据结构

### JDK 1.7
- **数组 + 链表**
- 使用头插法（多线程下会造成死循环）

### JDK 1.8 / Android
- **数组 + 链表 + 红黑树**
- 使用尾插法
- 链表长度 >= 8 且数组长度 >= 64 时，转为红黑树
- 红黑树节点 <= 6 时，退化为链表

---

## 二、核心参数

```java
// 默认初始容量 16
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;

// 最大容量 2^30
static final int MAXIMUM_CAPACITY = 1 << 30;

// 默认负载因子 0.75
static final float DEFAULT_LOAD_FACTOR = 0.75f;

// 链表转红黑树的阈值
static final int TREEIFY_THRESHOLD = 8;

// 红黑树退化为链表的阈值
static final int UNTREEIFY_THRESHOLD = 6;

// 最小树形化容量
static final int MIN_TREEIFY_CAPACITY = 64;
```

---

## 三、核心流程

### 1. Put 流程

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    
    // 1. 如果数组为空，先扩容
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    
    // 2. 计算索引，如果该位置为空，直接插入
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        
        // 3. 如果是同一个 key，覆盖
        if (p.hash == hash && 
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        
        // 4. 如果是红黑树节点
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        
        // 5. 链表遍历
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    // 链表长度 >= 8，转红黑树
                    if (binCount >= TREEIFY_THRESHOLD - 1)
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash && 
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        
        // 6. 覆盖旧值
        if (e != null) {
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    
    ++modCount;
    
    // 7. 超过阈值，扩容
    if (++size > threshold)
        resize();
    
    afterNodeInsertion(evict);
    return null;
}
```

### 2. Hash 计算

```java
static final int hash(Object key) {
    int h;
    // 高 16 位与低 16 位异或
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

**为什么这样做？**
- 减少哈希碰撞
- 让高位也参与运算
- 结合 `(n-1) & hash` 计算索引

### 3. 扩容机制 resize()

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    
    if (oldCap > 0) {
        // 已达到最大容量
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 容量翻倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1;
    }
    
    // ... 初始化或扩容
    threshold = newThr;
    Node<K,V>[] newTab = new Node[newCap];
    table = newTab;
    
    // 重新分配节点
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)
                    // 单节点，直接放入新位置
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    // 红黑树拆分
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else {
                    // 链表拆分（高低位）
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        } else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

---

## 四、面试高频问题

### Q1: HashMap 为什么线程不安全？
- 多线程 put 会造成数据覆盖
- JDK 1.7 头插法会造成死循环
- size 不是原子操作，可能不准确

### Q2: 为什么容量是 2 的幂次？
- `(n-1) & hash` 等价于 `hash % n`，位运算更快
- 保证哈希分布均匀

### Q3: 负载因子为什么是 0.75？
- 时间和空间的折中
- 太小：频繁扩容，浪费空间
- 太大：哈希碰撞多，查询慢

### Q4: 红黑树转链表阈值为什么是 6？
- 避免频繁转换（链表和红黑树之间）
- 7 作为中间值，8 转红黑树，6 退链表

### Q5: HashMap 和 HashTable 的区别？
| 特性 | HashMap | HashTable |
|------|---------|-----------|
| 线程安全 | 否 | 是（synchronized） |
| null 键值 | 允许 | 不允许 |
| 初始容量 | 16 | 11 |
| 扩容 | 2倍 | 2倍+1 |

### Q6: ConcurrentHashMap 原理？
- JDK 1.7: Segment 分段锁
- JDK 1.8: CAS + synchronized（锁单个桶）

---

## 五、源码关键点

### 1. 索引计算
```java
// 等价于 hash % n，但要求 n 是 2 的幂
i = (n - 1) & hash
```

### 2. 链表转红黑树
```java
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    // 数组长度 < 64，先扩容而不是转红黑树
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        TreeNode<K,V> hd = null, tl = null;
        do {
            TreeNode<K,V> p = replacementTreeNode(e, null);
            if (tl == null)
                hd = p;
            else {
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);
        if ((tab[index] = hd) != null)
            hd.treeify(tab);
    }
}
```

---

## 六、最佳实践

1. **预估容量**：初始化时指定容量，避免频繁扩容
   ```java
   // 预计存储 100 个元素
   new HashMap<>(100 / 0.75 + 1)
   ```

2. **不可变对象作为 Key**：确保 hashCode 和 equals 正确实现

3. **线程安全场景**：使用 ConcurrentHashMap 或 Collections.synchronizedMap()
