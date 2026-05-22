# 并发编程基础 - volatile synchronized 线程池

> 并发编程是 Android 面试的重点，必须理解 volatile、synchronized 和线程池的原理

---

## 一、volatile 关键字

### 1. 作用
- **可见性**：一个线程修改 volatile 变量，其他线程立即可见
- **有序性**：禁止指令重排序
- **不保证原子性**：不能保证复合操作的原子性

### 2. 底层原理
- 使用内存屏障（Memory Barrier）实现
- 写操作后加 StoreLoad 屏障
- 读操作前加 LoadLoad 屏障

### 3. 代码示例
```java
// 典型用法：状态标志
private volatile boolean running = true;

public void stop() {
    running = false; // 其他线程立即可见
}

public void run() {
    while (running) {
        // 执行任务
    }
}
```

### 4. 不适用场景
```java
// 非原子操作，volatile 无法保证线程安全
private volatile int count = 0;

public void increment() {
    count++; // 实际是三步操作：读取、加1、写入
}

// 解决方案：使用 AtomicInteger
private AtomicInteger count = new AtomicInteger(0);

public void increment() {
    count.incrementAndGet(); // 原子操作
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

### 3. 底层原理

#### 对象头 Mark Word
- 存储 hashCode、GC 分代年龄、锁状态标志、线程持有的锁

#### 锁升级过程
```
无锁 → 偏向锁 → 轻量级锁 → 重量级锁
```

| 锁状态 | 存储内容 | 标志位 |
|--------|----------|--------|
| 无锁 | HashCode、GC年龄 | 01 |
| 偏向锁 | 线程ID、Epoch | 01 |
| 轻量级锁 | 指向栈中锁记录的指针 | 00 |
| 重量级锁 | 指向 Monitor 的指针 | 10 |

### 4. 锁优化

#### 偏向锁
- 只有一个线程进入临界区
- 不需要 CAS 操作
- 适用场景：几乎没有竞争

#### 轻量级锁
- 多个线程交替进入临界区
- 使用 CAS 自旋
- 适用场景：少量竞争、执行时间短

#### 重量级锁
- 多个线程同时竞争
- 使用操作系统互斥量
- 适用场景：竞争激烈

---

## 三、线程池

### 1. 为什么使用线程池？
- 降低资源消耗（复用线程）
- 提高响应速度
- 提高线程的可管理性

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
核心线程是否已满？ ─否→ 创建核心线程执行
    ↓是
任务队列是否已满？ ─否→ 任务入队等待
    ↓是
最大线程数是否已满？ ─否→ 创建非核心线程执行
    ↓是
执行拒绝策略
```

### 4. 四种拒绝策略

| 策略 | 说明 |
|------|------|
| AbortPolicy | 抛出异常（默认） |
| CallerRunsPolicy | 调用者执行 |
| DiscardPolicy | 静默丢弃 |
| DiscardOldestPolicy | 丢弃最早任务 |

### 5. 常用线程池

#### FixedThreadPool
```java
ExecutorService pool = Executors.newFixedThreadPool(5);
// 特点：核心线程数 = 最大线程数，无超时
// 适用：并发量稳定的场景
```

#### CachedThreadPool
```java
ExecutorService pool = Executors.newCachedThreadPool();
// 特点：核心线程数为 0，最大线程数无限，60s 超时
// 适用：并发量波动大的场景
```

#### SingleThreadExecutor
```java
ExecutorService pool = Executors.newSingleThreadExecutor();
// 特点：只有一个核心线程
// 适用：需要顺序执行的场景
```

#### ScheduledThreadPool
```java
ScheduledExecutorService pool = Executors.newScheduledThreadPool(5);
// 特点：支持定时和周期性任务
// 适用：定时任务场景
```

### 6. 线程池状态

| 状态 | 说明 |
|------|------|
| RUNNING | 接受新任务，处理队列任务 |
| SHUTDOWN | 不接受新任务，处理队列任务 |
| STOP | 不接受新任务，不处理队列任务，中断正在执行的任务 |
| TIDYING | 所有任务终止，线程数为 0 |
| TERMINATED | terminated() 方法执行完 |

---

## 四、面试高频问题

### Q1: volatile 和 synchronized 的区别？
| 特性 | volatile | synchronized |
|------|----------|--------------|
| 原子性 | 不保证 | 保证 |
| 可见性 | 保证 | 保证 |
| 有序性 | 保证 | 保证 |
| 阻塞 | 不会 | 会 |
| 适用场景 | 状态标志、双重检查 | 临界区保护 |

### Q2: 线程池核心线程数怎么设置？
- **CPU 密集型**：核心线程数 = CPU 核心数 + 1
- **IO 密集型**：核心线程数 = CPU 核心数 * 2

### Q3: synchronized 和 ReentrantLock 的区别？
| 特性 | synchronized | ReentrantLock |
|------|--------------|---------------|
| 实现 | JVM 层面 | API 层面 |
| 释放锁 | 自动 | 手动（finally） |
| 可中断 | 不可 | 可以 |
| 公平锁 | 非公平 | 可选 |
| 条件变量 | 1 个 | 多个 |

### Q4: CAS 原理？
- Compare And Swap（比较并交换）
- 三个操作数：内存值 V、预期值 A、新值 B
- 当 V == A 时，将 V 更新为 B，否则不操作
- ABA 问题：使用版本号解决（AtomicStampedReference）

---

## 五、最佳实践

1. **避免过度使用 synchronized**：优先考虑并发容器
2. **合理配置线程池**：根据业务场景调整参数
3. **volatile 使用场景**：状态标志、双重检查锁定
4. **线程安全的单例**：
```java
public class Singleton {
    private volatile static Singleton instance;
    
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
