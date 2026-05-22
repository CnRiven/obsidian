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

```
┌─────────────────────────────────────────────────────────────┐
│                    HashMap 底层结构                          │
├─────────────────────────────────────────────────────────────┤
│  ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐  │
│  │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │ 8 │ 9 │...│n-2│n-1│   │
│  └─┬─┴─┬─┴───┴─┬─┴───┴───┴───┴─┬─┴───┴───┴───┴───┴───┘   │
│    │   │       │               │                            │
│    ▼   ▼       ▼               ▼                            │
│   [A] [B]→[C]  [D]→[E]→[F]    红黑树                        │
│              ↓                                               │
│             [G]                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 二、核心参数

```java
// 默认初始容量 16（必须是 2 的幂）
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;

// 最大容量 2^30
static final int MAXIMUM_CAPACITY = 1 << 30;

// 默认负载因子 0.75
static final float DEFAULT_LOAD_FACTOR = 0.75f;

// 链表转红黑树的阈值
static final int TREEIFY_THRESHOLD = 8;

// 红黑树退化为链表的阈值
static final int UNTREEIFY_THRESHOLD = 6;

// 最小树形化容量（数组长度至少 64 才会转红黑树）
static final int MIN_TREEIFY_CAPACITY = 64;
```

### 关键参数说明

| 参数 | 值 | 说明 |
|------|-----|------|
| DEFAULT_INITIAL_CAPACITY | 16 | 默认桶数量 |
| DEFAULT_LOAD_FACTOR | 0.75f | 扩容阈值 = 容量 × 负载因子 |
| TREEIFY_THRESHOLD | 8 | 链表长度 ≥ 8 转红黑树 |
| UNTREEIFY_THRESHOLD | 6 | 红黑树节点 ≤ 6 退化为链表 |
| MIN_TREEIFY_CAPACITY | 64 | 数组长度 < 64 时优先扩容而非树化 |

---

## 三、Node 内部类

```java
// 链表节点
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;    // 哈希值
    final K key;       // 键
    V value;           // 值
    Node<K,V> next;    // 下一个节点
    
    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }
}

// 红黑树节点（继承 LinkedHashMap.Entry，间接继承 Node）
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
    TreeNode<K,V> parent;  // 父节点
    TreeNode<K,V> left;    // 左子节点
    TreeNode<K,V> right;   // 右子节点
    TreeNode<K,V> prev;    // 前驱节点（用于删除后恢复链表）
    boolean red;           // 颜色标记
}
```

---

## 四、核心流程

### 1. Put 流程

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    
    // 1. 如果数组为空或长度为 0，先扩容（懒加载）
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    
    // 2. 计算索引 (n-1) & hash，如果该位置为空，直接插入新节点
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        
        // 3. 桶的第一个节点就是要找的 key
        if (p.hash == hash && 
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        
        // 4. 如果是红黑树节点，走红黑树的插入逻辑
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        
        // 5. 链表遍历（尾插法）
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    // 到达链表尾部，插入新节点
                    p.next = newNode(hash, key, value, null);
                    // 链表长度 >= 8，尝试转红黑树
                    if (binCount >= TREEIFY_THRESHOLD - 1)
                        treeifyBin(tab, hash);
                    break;
                }
                // 找到相同的 key
                if (e.hash == hash && 
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        
        // 6. key 已存在，覆盖旧值
        if (e != null) {
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e); // LinkedHashMap 回调
            return oldValue;
        }
    }
    
    ++modCount; // 修改计数器（fail-fast 机制）
    
    // 7. 超过阈值，扩容
    if (++size > threshold)
        resize();
    
    afterNodeInsertion(evict); // LinkedHashMap 回调
    return null;
}
```

### Put 流程图

```
put(key, value)
    ↓
计算 hash(key)
    ↓
数组是否为空？ ─是→ resize() 初始化
    ↓否
计算索引 i = (n-1) & hash
    ↓
tab[i] 是否为空？ ─是→ 直接插入新节点
    ↓否
tab[i] 的 key 是否相同？ ─是→ 覆盖旧值
    ↓否
tab[i] 是否是红黑树？ ─是→ 红黑树插入
    ↓否
遍历链表（尾插法）
    ↓
找到相同 key？ ─是→ 覆盖旧值
    ↓否
插入到链表尾部
    ↓
链表长度 >= 8？ ─是→ treeifyBin()
    ↓否
size > threshold？ ─是→ resize() 扩容
    ↓否
结束
```

### 2. Hash 计算

```java
static final int hash(Object key) {
    int h;
    // 高 16 位与低 16 位异或（扰动函数）
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

**为什么这样做？**
- 减少哈希碰撞
- 让高位也参与索引运算
- 结合 `(n-1) & hash` 计算索引
- 扰动函数：将高 16 位的特征混合到低 16 位

**索引计算原理：**
```java
// 假设 n = 16 (0001 0000)
// n - 1 = 15 (0000 1111)
// hash = xxxxxxxx xxxxxxxx xxxxxxxx xxxx???? 
// (n-1) & hash = 取 hash 的低 4 位

// 所以只有低 4 位参与索引计算
// 高 16 位的信息通过扰动函数混合到低 16 位
```

### 3. Get 流程

```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}

final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    
    // 1. 数组不为空且桶不为空
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        
        // 2. 检查第一个节点
        if (first.hash == hash && 
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        
        // 3. 有后续节点
        if ((e = first.next) != null) {
            // 4. 如果是红黑树，走红黑树查找
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            
            // 5. 链表遍历查找
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

### 4. Resize 扩容流程

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    
    if (oldCap > 0) {
        // 已达到最大容量，不再扩容
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 容量翻倍，阈值翻倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1;
    }
    // 使用 threshold 作为初始容量（带参构造函数）
    else if (oldThr > 0)
        newCap = oldThr;
    // 无参构造函数，使用默认值
    else {
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    
    // 创建新数组
    Node<K,V>[] newTab = new Node[newCap];
    table = newTab;
    
    // 重新分配节点
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null; // help GC
                
                // 单节点，直接放入新位置
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                // 红黑树拆分
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                // 链表拆分（高低位）
                else {
                    Node<K,V> loHead = null, loTail = null; // 低位链表
                    Node<K,V> hiHead = null, hiTail = null; // 高位链表
                    Node<K,V> next;
                    do {
                        next = e.next;
                        // 判断是低位还是高位
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
                    
                    // 低位链表放在原位置
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    // 高位链表放在原位置 + oldCap
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

### 扩容时链表拆分原理

```
原数组长度 n = 16 (0001 0000)
新数组长度 n = 32 (0010 0000)

原索引计算：hash & 0x0F (取低 4 位)
新索引计算：hash & 0x1F (取低 5 位)

区别在于第 5 位（oldCap 位置）

如果 hash & oldCap == 0：
    新索引 = 原索引（低位）
如果 hash & oldCap != 0：
    新索引 = 原索引 + oldCap（高位）

示例：
hash = 0b1010 0110
oldCap = 16 = 0b0001 0000
hash & oldCap = 0b0000 0000 = 0 → 低位

hash = 0b1011 0110
hash & oldCap = 0b0001 0000 ≠ 0 → 高位
```

---

## 五、红黑树特性

### 红黑树的五个性质
1. 每个节点是红色或黑色
2. 根节点是黑色
3. 每个叶子节点（NIL）是黑色
4. 红色节点的两个子节点都是黑色
5. 从任一节点到其每个叶子节点的所有路径包含相同数目的黑色节点

### 为什么用红黑树而不是 AVL 树？
- AVL 树更严格平衡，查询更快
- 红黑树插入删除时旋转次数更少
- HashMap 插入删除频繁，红黑树更合适

---

## 六、LinkedHashMap

### 1. 特点
- 继承 HashMap
- 维护一个双向链表
- 支持插入顺序或访问顺序

### 2. 底层结构

```java
public class LinkedHashMap<K,V> extends HashMap<K,V> {
    // 双向链表的头尾节点
    transient LinkedHashMap.Entry<K,V> head;
    transient LinkedHashMap.Entry<K,V> tail;
    
    // 排序模式：false 插入顺序，true 访问顺序
    final boolean accessOrder;
    
    // Entry 继承 HashMap.Node，增加了 before/after 指针
    static class Entry<K,V> extends HashMap.Node<K,V> {
        Entry<K,V> before, after;
    }
}
```

### 3. 使用场景
- LRU 缓存（accessOrder = true）
- 保持插入顺序

### 4. 实现 LRU 缓存

```java
public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int maxSize;
    
    public LRUCache(int maxSize) {
        // accessOrder = true 表示访问顺序
        super(maxSize, 0.75f, true);
        this.maxSize = maxSize;
    }
    
    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > maxSize; // 超过容量时删除最久未使用的
    }
}
```

---

## 七、HashSet

### 1. 特点
- 基于 HashMap 实现
- 元素作为 key，value 是固定的 PRESENT 对象
- 不允许重复元素

### 2. 源码

```java
public class HashSet<E> extends AbstractSet<E> {
    private transient HashMap<E,Object> map;
    
    // 所有 value 共用这个对象
    private static final Object PRESENT = new Object();
    
    public HashSet() {
        map = new HashMap<>();
    }
    
    public boolean add(E e) {
        return map.put(e, PRESENT) == null;
    }
    
    public boolean remove(Object o) {
        return map.remove(o) == PRESENT;
    }
}
```

---

## 八、ConcurrentHashMap

### JDK 1.7 实现（Segment 分段锁）

```java
// 结构：Segment[] + HashEntry[]
// 每个 Segment 继承 ReentrantLock
// 并发度：默认 16 个 Segment

static final class Segment<K,V> extends ReentrantLock {
    transient HashEntry<K,V>[] table;
    transient int count;
}

// 锁的粒度：Segment 级别
// 不同 Segment 可以并发写入
```

### JDK 1.8 实现（CAS + synchronized）

```java
// 结构：Node[] + 链表/红黑树（与 HashMap 相同）
// 锁的粒度：桶级别（更细）

final V putVal(K key, V value, boolean onlyIfAbsent) {
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        
        // 数组为空，初始化
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        
        // 桶为空，CAS 插入
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null)))
                break;
        }
        // 正在扩容，帮助扩容
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            synchronized (f) { // 锁住桶的第一个节点
                // 链表或红黑树插入逻辑
            }
        }
    }
}
```

### ConcurrentHashMap vs HashMap

| 特性 | HashMap | ConcurrentHashMap |
|------|---------|-------------------|
| 线程安全 | 否 | 是 |
| null 键值 | 允许 | 不允许 |
| 锁粒度 | 无锁 | 桶级别 |
| 性能 | 单线程最优 | 并发最优 |
| 迭代器 | fail-fast | 弱一致性 |

---

## 九、面试高频问题

### Q1: HashMap 为什么线程不安全？
- **数据覆盖**：多线程 put 时，可能同时写入同一个桶
- **死循环（JDK 1.7）**：头插法在扩容时可能导致环形链表
- **size 不准确**：size++ 不是原子操作

### Q2: 为什么容量是 2 的幂次？
- `(n-1) & hash` 等价于 `hash % n`，位运算更快
- 保证哈希分布均匀
- 扩容时可以快速定位新位置

### Q3: 负载因子为什么是 0.75？
- **数学推导**：泊松分布下，桶中节点数为 8 的概率约为 0.00000006
- **时间和空间的折中**：
  - 太小（如 0.5）：频繁扩容，浪费空间
  - 太大（如 1.0）：哈希碰撞多，查询慢

### Q4: 红黑树转链表阈值为什么是 8？
- **泊松分布**：正常情况下，链表长度达到 8 的概率极小
- **避免频繁转换**：7 作为中间缓冲，8 转红黑树，6 退链表
- **红黑树开销**：节点占用空间是链表的 2 倍

### Q5: HashMap 和 HashTable 的区别？

| 特性 | HashMap | HashTable |
|------|---------|-----------|
| 线程安全 | 否 | 是（synchronized） |
| null 键值 | 允许（1 个 null key） | 不允许 |
| 初始容量 | 16 | 11 |
| 扩容 | 2 倍 | 2 倍 + 1 |
| 迭代器 | fail-fast | enumerator |
| 性能 | 高 | 低 |
| 推荐 | 是 | 否（用 ConcurrentHashMap） |

### Q6: ConcurrentHashMap 原理？
- **JDK 1.7**：Segment 分段锁（默认 16 个 Segment）
- **JDK 1.8**：CAS + synchronized（锁单个桶，粒度更细）

### Q7: HashMap 的 put 流程？
1. 计算 key 的 hash 值
2. 如果数组为空，先扩容
3. 计算索引 `(n-1) & hash`
4. 如果桶为空，直接插入
5. 如果桶不为空：
   - key 相同，覆盖旧值
   - 是红黑树，红黑树插入
   - 是链表，尾插法插入
6. 如果链表长度 >= 8，尝试转红黑树
7. 如果 size > threshold，扩容

### Q8: HashMap 的 get 流程？
1. 计算 key 的 hash 值
2. 计算索引 `(n-1) & hash`
3. 检查第一个节点
4. 如果是红黑树，走红黑树查找
5. 如果是链表，遍历查找

### Q9: 为什么重写 equals 必须重写 hashCode？
```java
// 如果只重写 equals，不重写 hashCode
class User {
    String name;
    
    @Override
    public boolean equals(Object o) {
        return this.name.equals(((User)o).name);
    }
}

// 问题：
User u1 = new User("张三");
User u2 = new User("张三");
u1.equals(u2); // true

HashMap<User, String> map = new HashMap<>();
map.put(u1, "value");
map.get(u2); // 可能返回 null！
// 因为 u1 和 u2 的 hashCode 不同，计算出的索引不同
```

### Q10: HashMap 死循环问题（JDK 1.7）？

```java
// JDK 1.7 头插法扩容导致
// 线程 1 扩容时，A -> B 变成 B -> A
// 线程 2 同时扩容，可能导致 A -> B -> A 的环形链表
// get 时陷入死循环

// JDK 1.8 使用尾插法解决了这个问题
```

---

## 十、常见误区和踩坑点

### 误区 1：HashMap 的 size 是准确的
```java
// 多线程环境下，size 可能不准确
// 应该使用 ConcurrentHashMap.size()
```

### 误区 2：遍历时可以删除元素
```java
// 错误：会抛出 ConcurrentModificationException
for (Map.Entry<K, V> entry : map.entrySet()) {
    if (shouldRemove(entry)) {
        map.remove(entry.getKey()); // 错误！
    }
}

// 正确：使用迭代器的 remove
Iterator<Map.Entry<K, V>> it = map.entrySet().iterator();
while (it.hasNext()) {
    Map.Entry<K, V> entry = it.next();
    if (shouldRemove(entry)) {
        it.remove(); // 正确！
    }
}

// 或者使用 removeIf
map.entrySet().removeIf(entry -> shouldRemove(entry));
```

### 误区 3：初始容量设置不合理
```java
// 错误：预计存 100 个元素，设置容量为 100
new HashMap<>(100); // 实际容量 128，阈值 96

// 正确：考虑负载因子
new HashMap<>(100 / 0.75 + 1); // 容量 134，阈值 100
```

### 误区 4：使用可变对象作为 Key
```java
// 错误：使用可变对象作为 Key
List<String> list = new ArrayList<>();
list.add("item");
map.put(list, "value");

list.add("new item"); // 修改了 Key
map.get(list); // 可能返回 null！hashCode 变了

// 正确：使用不可变对象作为 Key
// String、Integer 等包装类
```

---

## 十一、最佳实践

### 1. 预估容量
```java
// 预计存储 100 个元素
int capacity = (int) (100 / 0.75) + 1;
Map<String, String> map = new HashMap<>(capacity);
```

### 2. 不可变对象作为 Key
```java
// String、Integer、Long 等不可变类作为 Key
Map<String, Object> map = new HashMap<>();
map.put("key", "value");
```

### 3. 线程安全场景
```java
// 方案一：ConcurrentHashMap（推荐）
ConcurrentHashMap<String, String> map = new ConcurrentHashMap<>();

// 方案二：Collections.synchronizedMap
Map<String, String> map = Collections.synchronizedMap(new HashMap<>());

// 方案三：Hashtable（不推荐，性能差）
Hashtable<String, String> map = new Hashtable<>();
```

### 4. 使用合适的初始容量
```java
// 根据预期元素数量计算
int expectedSize = 1000;
int capacity = (int) (expectedSize / 0.75f) + 1;
HashMap<String, String> map = new HashMap<>(capacity);
```

### 5. 遍历方式选择
```java
// 方式一：entrySet（推荐，一次获取 key 和 value）
for (Map.Entry<String, String> entry : map.entrySet()) {
    String key = entry.getKey();
    String value = entry.getValue();
}

// 方式二：keySet（需要再次 get value，不推荐）
for (String key : map.keySet()) {
    String value = map.get(key); // 额外的查找开销
}

// 方式三：forEach（Java 8+）
map.forEach((key, value) -> {
    // 处理
});
```

---

## 十二、性能对比

### 不同 Map 实现的性能

| 操作 | HashMap | LinkedHashMap | TreeMap | ConcurrentHashMap |
|------|---------|---------------|---------|-------------------|
| put | O(1) | O(1) | O(log n) | O(1) |
| get | O(1) | O(1) | O(log n) | O(1) |
| remove | O(1) | O(1) | O(log n) | O(1) |
| 遍历 | 无序 | 插入顺序 | 排序顺序 | 无序 |
| 线程安全 | 否 | 否 | 否 | 是 |

### 选择建议
- **单线程，快速访问**：HashMap
- **保持插入顺序**：LinkedHashMap
- **需要排序**：TreeMap
- **多线程**：ConcurrentHashMap
