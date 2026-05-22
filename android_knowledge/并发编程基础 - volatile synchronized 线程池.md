# 并发编程基础 - volatile synchronized 线程池

> 并发编程是 Android 面试的重点，必须理解 volatile、synchronized 和线程池的原理

---

## 一、volatile 关键字

### 1. 作用
- **可见性**：一个线程修改 volatile 变量，其他线程立即可见
- **有序性**：禁止指令重排序
- **不保证原子性**：不能保证复合操作的原子性

### 2. 底层原理 - 内存屏障

```
┌─────────────────────────────────────────────────────────────┐
│                    CPU 缓存一致性协议                         │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐   │
│  │  CPU 1  │    │  CPU 2  │    │  CPU 3  │    │  CPU 4  │   │
│  └────┬────┘    └────┬────┘    └────┬────┘    └────┬────┘   │
│       │              │              │              │         │
│       ▼              ▼              ▼              ▼         │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐   │
│  │ L1 Cache│    │ L1 Cache│    │ L1 Cache│    │ L1 Cache│   │
│  └────┬────┘    └────┬────┘    └────┬────┘    └────┬────┘   │
│       │              │              │              │         │
│       └──────────────┼──────────────┼──────────────┘         │
│                      │              │                         │
│                      ▼              ▼                         │
│                 ┌─────────────────────────┐                  │
│                 │      主内存 (RAM)       │                  │
│                 └─────────────────────────┘                  │
└─────────────────────────────────────────────────────────────┘
```

#### 内存屏障类型

| 屏障类型 | 说明 | 作用 |
|----------|------|------|
| LoadLoad | Load1; LoadLoad; Load2 | 保证 Load1 的读取先于 Load2 |
| StoreStore | Store1; StoreStore; Store2 | 保证 Store1 的写入先于 Store2 |
| LoadStore | Load1; LoadStore; Store2 | 保证 Load1 的读取先于 Store2 |
| StoreLoad | Store1; StoreLoad; Load2 | 保证 Store1 的写入先于 Load2 |

#### volatile 的内存屏障插入策略

```java
// volatile 写
┌─────────────────────────────────┐
│         StoreStore 屏障         │  ← 保证之前的写操作对其他处理器可见
├─────────────────────────────────┤
│      volatile 写操作            │
├─────────────────────────────────┤
│         StoreLoad 屏障          │  ← 保证 volatile 写对后续读可见
└─────────────────────────────────┘

// volatile 读
┌─────────────────────────────────┐
│         LoadLoad 屏障           │  ← 保证 volatile 读先于后续读
├─────────────────────────────────┤
│      volatile 读操作            │
├─────────────────────────────────┤
│         LoadStore 屏障          │  ← 保证 volatile 读先于后续写
└─────────────────────────────────┘
```

### 3. 可见性原理

```java
// 线程 1
boolean flag = false;

public void writer() {
    flag = true; // volatile 写
}

// 线程 2
public void reader() {
    if (flag) { // volatile 读
        // 保证能看到线程 1 的修改
    }
}
```

**原理：**
1. volatile 写时，将本地内存的值刷新到主内存
2. volatile 读时，从主内存读取最新值
3. 通过内存屏障保证顺序性

### 4. 有序性原理 - 禁止指令重排

```java
// 双重检查锁定（DCL）单例
public class Singleton {
    private volatile static Singleton instance;
    
    public static Singleton getInstance() {
        if (instance == null) {              // 第一次检查
            synchronized (Singleton.class) {
                if (instance == null) {      // 第二次检查
                    instance = new Singleton(); // 问题在这里！
                }
            }
        }
        return instance;
    }
}
```

**为什么需要 volatile？**

```java
// new Singleton() 实际是三步操作：
// 1. 分配内存空间
// 2. 初始化对象
// 3. 将引用指向内存地址

// 可能的指令重排：
// 1. 分配内存空间
// 3. 将引用指向内存地址  ← 对象还未初始化！
// 2. 初始化对象

// 线程 2 可能看到一个未初始化完成的对象
```

### 5. 不保证原子性

```java
// 错误示例：volatile 不保证原子性
private volatile int count = 0;

public void increment() {
    count++; // 实际是三步操作：读取、加1、写入
}

// 线程 1：读取 count = 0
// 线程 2：读取 count = 0
// 线程 1：count = 0 + 1 = 1，写入
// 线程 2：count = 0 + 1 = 1，写入
// 结果：count = 1（期望是 2）
```

**解决方案：**

```java
// 方案一：AtomicInteger
private AtomicInteger count = new AtomicInteger(0);

public void increment() {
    count.incrementAndGet(); // CAS 原子操作
}

// 方案二：synchronized
private int count = 0;

public synchronized void increment() {
    count++;
}

// 方案三：ReentrantLock
private int count = 0;
private Lock lock = new ReentrantLock();

public void increment() {
    lock.lock();
    try {
        count++;
    } finally {
        lock.unlock();
    }
}
```

---

## 二、synchronized 关键字

### 1. 作用
- **原子性**：保证同一时刻只有一个线程执行
- **可见性**：解锁前对变量的修改对后续加锁可见
- **有序性**：一个变量在同一时刻只允许一条线程对其进行 lock 操作

### 2. 使用方式

#### 对象锁
```java
// 1. 同步代码块
synchronized (this) {
    // 临界区
}

// 2. 同步方法
public synchronized void method() {
    // 临界区
}

// 3. 锁定任意对象
private final Object lock = new Object();
synchronized (lock) {
    // 临界区
}
```

#### 类锁
```java
// 1. 静态同步方法
public static synchronized void method() {
    // 临界区
}

// 2. Class 对象锁
synchronized (MyClass.class) {
    // 临界区
}
```

### 3. 底层原理 - 对象头 Mark Word

```
┌─────────────────────────────────────────────────────────────┐
│                    对象头 Mark Word                          │
├─────────────────────────────────────────────────────────────┤
│  无锁状态：                                                   │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ unused:25 | hashcode:31 | unused:4 | age:4 | biased:1 │ 01│
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
│  偏向锁：                                                     │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ thread:54 | epoch:2 | unused:4 | age:4 | biased:1 │ 01│
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
│  轻量级锁：                                                   │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ ptr_to_lock_record:62                            │ 00│
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
│  重量级锁：                                                   │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ ptr_to_heavyweight_monitor:62                    │ 10│
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
│  GC 标记：                                                    │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                                               │ 11│
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

### 4. 锁升级过程

```
无锁 → 偏向锁 → 轻量级锁 → 重量级锁
  ↓       ↓         ↓         ↓
 可逆    可逆      不可逆     不可逆
```

#### 偏向锁
```java
// 特点：
// - 只有一个线程进入临界区
// - 不需要 CAS 操作
// - 线程 ID 记录在对象头

// 加锁过程：
// 1. 检查 Mark Word 是否可偏向
// 2. 如果可偏向，CAS 设置线程 ID
// 3. 如果线程 ID 已经是自己，直接进入
// 4. 如果线程 ID 不是自己，撤销偏向锁

// 适用场景：几乎没有竞争
```

#### 轻量级锁
```java
// 特点：
// - 多个线程交替进入临界区
// - 使用 CAS 自旋
// - 不会阻塞线程

// 加锁过程：
// 1. JVM 在当前线程的栈帧中创建 Lock Record
// 2. CAS 将 Mark Word 复制到 Lock Record
// 3. CAS 将 Mark Word 指向 Lock Record
// 4. 如果 CAS 失败，自旋等待

// 适用场景：少量竞争、执行时间短
```

#### 重量级锁
```java
// 特点：
// - 多个线程同时竞争
// - 使用操作系统互斥量
// - 会阻塞线程

// 加锁过程：
// 1. JVM 申请 Monitor 对象
// 2. 将 Mark Word 指向 Monitor
// 3. 线程进入 EntryList 等待
// 4. 获取锁的线程成为 Owner
// 5. 释放锁时唤醒 EntryList 中的线程

// 适用场景：竞争激烈
```

### 5. 锁优化

#### 自旋锁
```java
// 线程不立即阻塞，而是循环等待
// 优点：避免线程上下文切换
// 缺点：消耗 CPU

// 自适应自旋：
// - 根据上次自旋结果调整自旋次数
// - 上次成功，增加自旋次数
// - 上次失败，减少自旋次数
```

#### 锁消除
```java
// JIT 编译器优化，消除不可能存在竞争的锁
public String concatString(String s1, String s2) {
    StringBuffer sb = new StringBuffer(); // StringBuffer 是线程安全的
    sb.append(s1);
    sb.append(s2);
    return sb.toString();
    // JIT 会消除 StringBuffer 的同步锁
}
```

#### 锁粗化
```java
// 将多个连续的锁合并为一个锁
public void method() {
    synchronized (lock) {
        // 操作 1
    }
    synchronized (lock) {
        // 操作 2
    }
    synchronized (lock) {
        // 操作 3
    }
    // JIT 会合并为一个 synchronized 块
}
```

---

## 三、CAS 原理

### 1. 什么是 CAS？
- Compare And Swap（比较并交换）
- 三个操作数：内存值 V、预期值 A、新值 B
- 当 V == A 时，将 V 更新为 B，否则不操作
- 原子操作，由 CPU 指令保证

### 2. 实现原理

```java
// Unsafe 类的 CAS 操作
public final native boolean compareAndSwapInt(Object o, long offset, int expected, int newValue);

// 底层使用 CPU 的 CAS 指令
// x86: cmpxchg 指令
// ARM: ldrex/strex 指令
```

### 3. ABA 问题

```java
// 问题场景
// 线程 1：读取值 A
// 线程 2：将 A 改为 B
// 线程 2：将 B 改回 A
// 线程 1：CAS 检查，值还是 A，认为没变化，更新成功
// 实际上值已经被修改过了！
```

### 4. ABA 问题解决方案

```java
// 方案一：AtomicStampedReference（版本号）
AtomicStampedReference<Integer> ref = new AtomicStampedReference<>(1, 0);
int stamp = ref.getStamp(); // 获取版本号
ref.compareAndSet(1, 2, stamp, stamp + 1); // 同时比较值和版本号

// 方案二：AtomicMarkableReference（布尔标记）
AtomicMarkableReference<Integer> ref = new AtomicMarkableReference<>(1, false);
boolean[] mark = new boolean[1];
int value = ref.get(mark); // 同时获取值和标记
ref.compareAndSet(1, 2, false, true);
```

---

## 四、原子类

### 1. 基本类型原子类

```java
// AtomicInteger
AtomicInteger count = new AtomicInteger(0);
count.incrementAndGet(); // ++i
count.getAndIncrement(); // i++
count.addAndGet(10);     // i += 10
count.compareAndSet(0, 1); // CAS

// AtomicLong
AtomicLong longValue = new AtomicLong(0);

// AtomicBoolean
AtomicBoolean flag = new AtomicBoolean(false);
```

### 2. 引用类型原子类

```java
// AtomicReference
AtomicReference<User> userRef = new AtomicReference<>(new User("张三"));
userRef.compareAndSet(new User("张三"), new User("李四"));

// AtomicStampedReference（解决 ABA 问题）
AtomicStampedReference<String> ref = new AtomicStampedReference<>("A", 0);
int stamp = ref.getStamp();
ref.compareAndSet("A", "B", stamp, stamp + 1);

// AtomicMarkableReference（带布尔标记）
AtomicMarkableReference<String> ref = new AtomicMarkableReference<>("A", false);
```

### 3. 数组原子类

```java
// AtomicIntegerArray
int[] arr = new int[10];
AtomicIntegerArray atomicArr = new AtomicIntegerArray(arr);
atomicArr.incrementAndGet(0); // arr[0]++

// AtomicLongArray
// AtomicReferenceArray
```

### 4. 字段更新器

```java
// AtomicIntegerFieldUpdater
public class User {
    volatile int age; // 必须 volatile
}

AtomicIntegerFieldUpdater<User> updater = 
    AtomicIntegerFieldUpdater.newUpdater(User.class, "age");
User user = new User();
updater.set(user, 25);
updater.compareAndSet(user, 25, 26);
```

### 5. 原子累加器（Java 8+）

```java
// LongAdder（高并发下比 AtomicLong 更高效）
LongAdder adder = new LongAdder();
adder.increment();
adder.add(10);
long sum = adder.sum(); // 获取结果

// LongAccumulator（自定义累加操作）
LongAccumulator acc = new LongAccumulator((x, y) -> x + y, 0);
acc.accumulate(10);
acc.accumulate(20);
long result = acc.get(); // 30
```

---

## 五、线程池

### 1. 为什么使用线程池？
- **降低资源消耗**：复用线程，减少创建销毁开销
- **提高响应速度**：任务到达时不需要等待线程创建
- **提高线程的可管理性**：统一分配、调优和监控

### 2. 核心参数

```java
public ThreadPoolExecutor(
    int corePoolSize,      // 核心线程数
    int maximumPoolSize,   // 最大线程数
    long keepAliveTime,    // 线程存活时间
    TimeUnit unit,         // 时间单位
    BlockingQueue<Runnable> workQueue,  // 任务队列
    ThreadFactory threadFactory,        // 线程工厂
    RejectedExecutionHandler handler    // 拒绝策略
)
```

### 3. 执行流程

```
提交任务
    ↓
当前线程数 < corePoolSize？
    │是
    └→ 创建核心线程执行
    ↓否
任务队列是否已满？
    │否
    └→ 任务入队等待
    ↓是
当前线程数 < maximumPoolSize？
    │是
    └→ 创建非核心线程执行
    ↓否
执行拒绝策略
```

### 4. 四种拒绝策略

```java
// 1. AbortPolicy（默认）
// 抛出 RejectedExecutionException
new ThreadPoolExecutor.AbortPolicy();

// 2. CallerRunsPolicy
// 由调用线程执行任务
new ThreadPoolExecutor.CallerRunsPolicy();

// 3. DiscardPolicy
// 静默丢弃任务
new ThreadPoolExecutor.DiscardPolicy();

// 4. DiscardOldestPolicy
// 丢弃队列中最早的任务，重新提交当前任务
new ThreadPoolExecutor.DiscardOldestPolicy();
```

### 5. 常用线程池

```java
// 1. FixedThreadPool
// 核心线程数 = 最大线程数，无超时
ExecutorService pool = Executors.newFixedThreadPool(5);
// 问题：LinkedBlockingQueue 无界，可能 OOM

// 2. CachedThreadPool
// 核心线程数为 0，最大线程数无限，60s 超时
ExecutorService pool = Executors.newCachedThreadPool();
// 问题：最大线程数无限，可能创建过多线程

// 3. SingleThreadExecutor
// 只有一个核心线程
ExecutorService pool = Executors.newSingleThreadExecutor();
// 问题：LinkedBlockingQueue 无界，可能 OOM

// 4. ScheduledThreadPool
// 支持定时和周期性任务
ScheduledExecutorService pool = Executors.newScheduledThreadPool(5);
// 问题：DelayedWorkQueue 无界，可能 OOM
```

### 6. 线程池状态

```java
// 线程池状态（高 3 位）
private static final int RUNNING    = -1 << COUNT_BITS; // 111
private static final int SHUTDOWN   =  0 << COUNT_BITS; // 000
private static final int STOP       =  1 << COUNT_BITS; // 001
private static final int TIDYING    =  2 << COUNT_BITS; // 010
private static final int TERMINATED =  3 << COUNT_BITS; // 011

// 线程数（低 29 位）
private static final int COUNT_BITS = 29;
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;
```

| 状态 | 说明 | 接受新任务 | 处理队列任务 |
|------|------|-----------|-------------|
| RUNNING | 运行状态 | ✅ | ✅ |
| SHUTDOWN | 关闭状态 | ❌ | ✅ |
| STOP | 停止状态 | ❌ | ❌ |
| TIDYING | 整理状态 | ❌ | ❌ |
| TERMINATED | 终止状态 | ❌ | ❌ |

### 7. 线程池状态转换

```
RUNNING → SHUTDOWN（调用 shutdown()）
SHUTDOWN → STOP（调用 shutdownNow()）
STOP → TIDYING（队列为空，线程数为 0）
TIDYING → TERMINATED（terminated() 执行完）
```

### 8. 合理配置线程池

```java
// CPU 密集型：线程数 = CPU 核心数 + 1
int cpuCores = Runtime.getRuntime().availableProcessors();
ExecutorService cpuPool = Executors.newFixedThreadPool(cpuCores + 1);

// IO 密集型：线程数 = CPU 核心数 * 2
ExecutorService ioPool = Executors.newFixedThreadPool(cpuCores * 2);

// 混合型：根据 IO 和 CPU 的比例调整
// 最佳线程数 = CPU 核心数 * (1 + IO 耗时 / CPU 耗时)
```

---

## 六、并发工具类

### 1. CountDownLatch（倒计时器）

```java
// 等待 N 个任务完成
CountDownLatch latch = new CountDownLatch(3);

// 启动 3 个任务
for (int i = 0; i < 3; i++) {
    new Thread(() -> {
        try {
            // 执行任务
            Thread.sleep(1000);
        } finally {
            latch.countDown(); // 计数器减 1
        }
    }).start();
}

latch.await(); // 等待计数器为 0
System.out.println("所有任务完成");
```

### 2. CyclicBarrier（循环栅栏）

```java
// N 个线程互相等待，都到达后才继续
CyclicBarrier barrier = new CyclicBarrier(3, () -> {
    System.out.println("所有线程到达栅栏");
});

for (int i = 0; i < 3; i++) {
    new Thread(() -> {
        try {
            System.out.println("线程到达栅栏");
            barrier.await(); // 等待其他线程
            System.out.println("线程继续执行");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }).start();
}
```

### 3. Semaphore（信号量）

```java
// 控制并发访问的线程数量
Semaphore semaphore = new Semaphore(3); // 最多 3 个线程同时访问

for (int i = 0; i < 10; i++) {
    new Thread(() -> {
        try {
            semaphore.acquire(); // 获取许可
            System.out.println("获取许可，当前可用: " + semaphore.availablePermits());
            Thread.sleep(1000);
        } finally {
            semaphore.release(); // 释放许可
        }
    }).start();
}
```

### 4. CountDownLatch vs CyclicBarrier

| 特性 | CountDownLatch | CyclicBarrier |
|------|----------------|---------------|
| 计数方式 | 递减 | 递增 |
| 重用 | 不可重用 | 可重用 |
| 等待方 | 一个线程等待多个线程 | 多个线程互相等待 |
| 场景 | 任务完成后通知 | 多线程同步开始 |

---

## 七、ThreadLocal

### 1. 作用
- 线程局部变量
- 每个线程有独立的变量副本
- 避免线程安全问题

### 2. 使用方式

```java
// 创建 ThreadLocal
private static ThreadLocal<User> userHolder = new ThreadLocal<>();

// 设置值
userHolder.set(new User("张三"));

// 获取值
User user = userHolder.get();

// 移除值（防止内存泄漏）
userHolder.remove();
```

### 3. 底层原理

```java
// ThreadLocalMap
public class ThreadLocal<T> {
    
    // 每个线程都有一个 ThreadLocalMap
    static class ThreadLocalMap {
        // Entry 继承 WeakReference<ThreadLocal<?>>
        static class Entry extends WeakReference<ThreadLocal<?>> {
            Object value;
        }
        
        private Entry[] table;
    }
    
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = t.threadLocals;
        if (map != null) {
            map.set(this, value);
        } else {
            createMap(t, value);
        }
    }
    
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = t.threadLocals;
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                return (T) e.value;
            }
        }
        return null;
    }
}
```

### 4. 内存泄漏问题

```
┌─────────────────────────────────────────────────────────────┐
│                    ThreadLocal 内存泄漏                      │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────┐    │
│  │                    Thread 对象                       │    │
│  │  ┌─────────────────────────────────────────────┐    │    │
│  │  │            ThreadLocalMap                   │    │    │
│  │  │  ┌───────┬───────┬───────┬───────┐        │    │    │
│  │  │  │ Entry │ Entry │ Entry │ Entry │        │    │    │
│  │  │  └───┬───┴───┬───┴───┬───┴───┬───┘        │    │    │
│  │  │      │       │       │       │              │    │    │
│  │  │      ▼       ▼       ▼       ▼              │    │    │
│  │  │   ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐        │    │    │
│  │  │   │Key  │ │Key  │ │Key  │ │Key  │        │    │    │
│  │  │   │Weak │ │Weak │ │Weak │ │Weak │        │    │    │
│  │  │   └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘        │    │    │
│  │  │      │       │       │       │              │    │    │
│  │  │      ▼       ▼       ▼       ▼              │    │    │
│  │  │   ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐        │    │    │
│  │  │   │Value│ │Value│ │Value│ │Value│        │    │    │
│  │  │   │Strong│ │Strong│ │Strong│ │Strong│       │    │    │
│  │  │   └─────┘ └─────┘ └─────┘ └─────┘        │    │    │
│  │  └─────────────────────────────────────────────┘    │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
│  问题：                                                       │
│  - Key 是弱引用，会被 GC 回收                                  │
│  - Value 是强引用，不会被 GC 回收                               │
│  - Key 被回收后，Value 无法访问，造成内存泄漏                      │
└─────────────────────────────────────────────────────────────┘
```

### 5. 解决方案

```java
// 使用后必须调用 remove()
private static ThreadLocal<User> userHolder = new ThreadLocal<>();

public void method() {
    try {
        userHolder.set(new User("张三"));
        // 使用 userHolder.get()
    } finally {
        userHolder.remove(); // 必须移除！
    }
}
```

---

## 八、面试高频问题

### Q1: volatile 和 synchronized 的区别？

| 特性 | volatile | synchronized |
|------|----------|--------------|
| 原子性 | ❌ 不保证 | ✅ 保证 |
| 可见性 | ✅ 保证 | ✅ 保证 |
| 有序性 | ✅ 保证 | ✅ 保证 |
| 阻塞 | ❌ 不会 | ✅ 会 |
| 性能 | 高 | 低 |
| 适用场景 | 状态标志、双重检查 | 临界区保护 |

### Q2: 线程池核心线程数怎么设置？

```java
// CPU 密集型：线程数 = CPU 核心数 + 1
int cpuCores = Runtime.getRuntime().availableProcessors();

// IO 密集型：线程数 = CPU 核心数 * 2

// 混合型：线程数 = CPU 核心数 * (1 + IO 耗时 / CPU 耗时)
```

### Q3: synchronized 和 ReentrantLock 的区别？

| 特性 | synchronized | ReentrantLock |
|------|--------------|---------------|
| 实现 | JVM 层面 | API 层面 |
| 释放锁 | 自动 | 手动（finally） |
| 可中断 | ❌ 不可 | ✅ 可以 |
| 公平锁 | 非公平 | 可选 |
| 条件变量 | 1 个 | 多个 |
| 锁升级 | 支持 | 不支持 |

### Q4: CAS 原理和 ABA 问题？

**CAS 原理：**
- Compare And Swap（比较并交换）
- 三个操作数：内存值 V、预期值 A、新值 B
- 当 V == A 时，将 V 更新为 B，否则不操作
- 原子操作，由 CPU 指令保证

**ABA 问题：**
- 值从 A 变为 B，再变回 A
- CAS 检查时值还是 A，认为没变化
- 解决方案：使用版本号（AtomicStampedReference）

### Q5: ThreadLocal 原理和内存泄漏？

**原理：**
- 每个线程有独立的 ThreadLocalMap
- ThreadLocal 作为 Key，值作为 Value
- Key 是弱引用，Value 是强引用

**内存泄漏：**
- Key 被 GC 回收后，Value 无法访问
- 线程不销毁，Value 一直存在

**解决方案：**
- 使用后调用 remove()

---

## 九、最佳实践

### 1. volatile 使用场景
```java
// 状态标志
private volatile boolean running = true;

// 双重检查锁定
private volatile Singleton instance;

// 独立观察
private volatile long lastTime = System.currentTimeMillis();
```

### 2. synchronized 使用场景
```java
// 保护共享变量
public synchronized void increment() {
    count++;
}

// 保护代码块
synchronized (lock) {
    // 临界区
}
```

### 3. 线程池使用
```java
// 自定义线程池
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    10,                     // 核心线程数
    20,                     // 最大线程数
    60, TimeUnit.SECONDS,   // 超时时间
    new LinkedBlockingQueue<>(1000), // 有界队列
    new ThreadFactory() {
        private AtomicInteger count = new AtomicInteger(0);
        @Override
        public Thread newThread(Runnable r) {
            return new Thread(r, "my-pool-" + count.incrementAndGet());
        }
    },
    new ThreadPoolExecutor.CallerRunsPolicy() // 拒绝策略
);
```

### 4. ThreadLocal 使用
```java
private static ThreadLocal<User> userHolder = new ThreadLocal<>();

public void method() {
    try {
        userHolder.set(new User("张三"));
        // 使用 userHolder.get()
    } finally {
        userHolder.remove();
    }
}
```
