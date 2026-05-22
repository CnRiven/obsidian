# Handler 机制原理详解

> Handler 是 Android 消息机制的核心，面试必问！

---

## 一、核心组件

```
┌─────────────────────────────────────────────────────────────┐
│                      Handler 机制                            │
├─────────────────────────────────────────────────────────────┤
│  ┌──────────┐    ┌───────────────┐    ┌──────────────────┐  │
│  │ Handler  │───→│ MessageQueue  │───→│     Looper       │  │
│  │ 发送消息 │    │  消息队列      │    │  消息循环         │  │
│  └──────────┘    └───────────────┘    └──────────────────┘  │
│                                                             │
│  Message: 消息对象，包含 what、arg1、arg2、obj 等           │
│                                                             │
│  一个线程只能有一个 Looper                                    │
│  一个 Looper 只能绑定一个 MessageQueue                        │
│  一个 MessageQueue 可以有多个 Handler 发送消息                 │
└─────────────────────────────────────────────────────────────┘
```

### 1. Handler
- 发送消息（sendMessage、post）
- 处理消息（handleMessage）

### 2. MessageQueue
- 消息队列（单链表实现）
- 按时间排序（when 字段）
- 底层使用 epoll 等待消息

### 3. Looper
- 消息循环（loop() 方法）
- 一个线程只能有一个 Looper
- 主线程 Looper 在 ActivityThread.main() 中创建

### 4. Message
- 消息载体
- 对象池复用（maxPoolSize = 50）

---

## 二、核心流程

### 1. Looper.prepare()

```java
public static void prepare() {
    prepare(true);
}

private static void prepare(boolean quitAllowed) {
    // 检查是否已经有 Looper
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    // 设置 Looper 到 ThreadLocal
    sThreadLocal.set(new Looper(quitAllowed));
}

private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed); // 创建 MessageQueue
    mThread = Thread.currentThread();       // 记录当前线程
}
```

**关键点：**
- 一个线程只能有一个 Looper
- Looper 存储在 ThreadLocal 中
- 创建 Looper 时会创建 MessageQueue

### 2. Looper.loop()

```java
public static void loop() {
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    
    final MessageQueue queue = me.mQueue;
    
    // 死循环，不断取消息
    for (;;) {
        // 1. 取消息（可能会阻塞）
        Message msg = queue.next();
        
        if (msg == null) {
            // 没有消息，退出循环
            return;
        }
        
        // 2. 分发消息给 Handler
        final long dispatchStart = SystemClock.uptimeMillis();
        msg.target.dispatchMessage(msg); // msg.target 就是发送消息的 Handler
        
        // 3. 回收消息到对象池
        msg.recycleUnchecked();
    }
}

public void dispatchMessage(Message msg) {
    // 优先级：Message.callback > Handler.mCallback > Handler.handleMessage
    
    // 1. Message 的 callback（post 方式）
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        // 2. Handler 的 mCallback
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        // 3. Handler 的 handleMessage（子类重写）
        handleMessage(msg);
    }
}
```

**关键点：**
- 死循环，不断取消息
- queue.next() 可能会阻塞（没有消息时）
- 消息分发给对应的 Handler 处理

### 3. Handler 发送消息

```java
// sendMessage
public final boolean sendMessage(Message msg) {
    return sendMessageDelayed(msg, 0);
}

public final boolean sendMessageDelayed(Message msg, long delayMillis) {
    if (delayMillis < 0) {
        delayMillis = 0;
    }
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}

public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}

private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this; // 设置消息的 target 为当前 Handler
    
    if (mAsynchronous) {
        msg.setAsynchronous(true); // 设置异步消息
    }
    
    return queue.enqueueMessage(msg, uptimeMillis);
}

// post
public final boolean post(Runnable r) {
    return sendMessageDelayed(getPostMessage(r), 0);
}

private static Message getPostMessage(Runnable r) {
    Message m = Message.obtain();
    m.callback = r; // Runnable 作为 callback
    return m;
}
```

### 4. MessageQueue 入队

```java
boolean enqueueMessage(Message msg, long when) {
    // 检查消息是否有效
    if (msg.target == null) {
        throw new IllegalArgumentException("Message must have a target.");
    }
    
    synchronized (this) {
        msg.when = when;
        
        Message prev = null;
        Message p = mMessages; // 链表头
        
        // 按时间排序插入
        if (when == 0 || when < p.when) {
            // 插入队头
            msg.next = p;
            mMessages = msg;
        } else {
            // 找到合适位置插入
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
            }
            msg.next = p;
            prev.next = msg;
        }
    }
    return true;
}
```

### 5. MessageQueue.next()

```java
Message next() {
    int nextPollTimeoutMillis = 0;
    
    for (;;) {
        // 1. 阻塞等待消息
        nativePollOnce(ptr, nextPollTimeoutMillis);
        
        synchronized (this) {
            final long now = SystemClock.uptimeMillis();
            Message msg = mMessages;
            
            // 2. 如果队头是同步屏障
            if (msg != null && msg.target == null) {
                // 跳过同步消息，找到异步消息
                do {
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            
            if (msg != null) {
                if (now < msg.when) {
                    // 3. 还没到执行时间，计算等待时间
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // 4. 取出消息返回
                    mMessages = msg.next;
                    msg.next = null;
                    return msg;
                }
            } else {
                // 5. 没有消息，无限等待
                nextPollTimeoutMillis = -1;
            }
            
            // 6. 处理 IdleHandler
            if (pendingIdleHandlerCount < 0 && (mMessages == null || now < mMessages.when)) {
                pendingIdleHandlerCount = mIdleHandlers.size();
            }
            
            if (pendingIdleHandlerCount > 0) {
                IdleHandler[] handlers = mIdleHandlers.toArray(mIdleHandlers.size());
                for (int i = 0; i < pendingIdleHandlerCount; i++) {
                    IdleHandler idler = handlers[i];
                    boolean keep = idler.queueIdle();
                    if (!keep) {
                        mIdleHandlers.remove(idler);
                    }
                }
                pendingIdleHandlerCount = 0;
                nextPollTimeoutMillis = 0;
            }
        }
    }
}
```

---

## 三、epoll 机制

### 1. 什么是 epoll？
- Linux 内核的 IO 多路复用机制
- 用于高效处理大量文件描述符
- MessageQueue 底层使用 epoll 等待消息

### 2. 工作原理

```
┌─────────────────────────────────────────────────────────────┐
│                    epoll 工作原理                            │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────┐    │
│  │                    用户空间                          │    │
│  │  ┌─────────┐    ┌─────────┐    ┌─────────┐        │    │
│  │  │ Thread 1│    │ Thread 2│    │ Thread 3│        │    │
│  │  └────┬────┘    └────┬────┘    └────┬────┘        │    │
│  │       │              │              │              │    │
│  │       └──────────────┼──────────────┘              │    │
│  │                      │                              │    │
│  │                      ▼                              │    │
│  │               ┌─────────────┐                      │    │
│  │               │  epoll_wait │                      │    │
│  │               └──────┬──────┘                      │    │
│  └──────────────────────┼──────────────────────────────┘    │
│                         │                                    │
│  ───────────────────────┼───────────────────────────────────│
│                         │                                    │
│  ┌──────────────────────┼──────────────────────────────┐    │
│  │                 内核空间                              │    │
│  │                      │                              │    │
│  │                      ▼                              │    │
│  │               ┌─────────────┐                      │    │
│  │               │   epoll     │                      │    │
│  │               │   实例      │                      │    │
│  │               └──────┬──────┘                      │    │
│  │                      │                              │    │
│  │         ┌────────────┼────────────┐                │    │
│  │         ▼            ▼            ▼                │    │
│  │    ┌─────────┐  ┌─────────┐  ┌─────────┐        │    │
│  │    │  fd 1   │  │  fd 2   │  │  fd 3   │        │    │
│  │    │(eventfd)│  │(socket) │  │(pipe)   │        │    │
│  │    └─────────┘  └─────────┘  └─────────┘        │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

### 3. MessageQueue 中的 epoll

```java
// native 层使用 epoll
private native static void nativePollOnce(long ptr, int timeoutMillis);
private native static void nativeWake(long ptr);

// 当调用 next() 时：
// 1. 如果没有消息，调用 epoll_wait 阻塞等待
// 2. 当有新消息时，调用 nativeWake 唤醒

// Java 层
Message msg = queue.next(); // 可能阻塞
```

---

## 四、同步屏障

### 1. 什么是同步屏障？
- 一种特殊的消息（target 为 null）
- 优先处理异步消息
- 用于保证 UI 绘制的及时性

### 2. 发送同步屏障

```java
// MessageQueue
public int postSyncBarrier() {
    return postSyncBarrier(SystemClock.uptimeMillis());
}

private int postSyncBarrier(long when) {
    synchronized (this) {
        final int token = mNextBarrierToken++;
        
        // 创建同步屏障消息（target 为 null）
        final Message msg = Message.obtain();
        msg.markInUse();
        msg.when = when;
        msg.arg1 = token;
        
        // 插入队列（按时间排序）
        Message prev = null;
        Message p = mMessages;
        if (when != 0) {
            while (p != null && p.when <= when) {
                prev = p;
                p = p.next;
            }
        }
        if (prev != null) {
            msg.next = p;
            prev.next = msg;
        } else {
            msg.next = p;
            mMessages = msg;
        }
        
        return token;
    }
}

// 移除同步屏障
public void removeSyncBarrier(int token) {
    synchronized (this) {
        Message prev = null;
        Message p = mMessages;
        while (p != null && (p.target != null || p.arg1 != token)) {
            prev = p;
            p = p.next;
        }
        if (p == null) {
            throw new IllegalStateException("The specified message queue synchronization  barrier token has not been posted or has already been removed.");
        }
        if (prev != null) {
            prev.next = p.next;
        } else {
            mMessages = p.next;
        }
        p.recycleUnchecked();
    }
}
```

### 3. 同步屏障的应用 - ViewRootImpl.scheduleTraversals()

```java
// ViewRootImpl.java
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        
        // 1. 发送同步屏障
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        
        // 2. 发送异步消息（Choreographer）
        mChoreographer.postCallback(
            Choreographer.CALLBACK_TRAVERSAL,
            mTraversalRunnable,
            null
        );
    }
}

// Choreographer 发送的是异步消息
// 同步屏障保证异步消息（UI 绘制）优先执行
```

### 4. 同步屏障的作用

```
消息队列：
[同步消息 A] → [同步消息 B] → [同步屏障] → [同步消息 C] → [异步消息 D]

处理顺序：
1. 检测到同步屏障
2. 跳过同步消息 A、B
3. 找到异步消息 D
4. 处理异步消息 D
5. 移除同步屏障
6. 处理同步消息 A、B、C
```

---

## 五、IdleHandler

### 1. 什么是 IdleHandler？
- 空闲时执行的任务
- 在 MessageQueue.next() 中调用
- 适用于延迟初始化、GC 等

### 2. 使用方式

```java
Looper.myQueue().addIdleHandler(new MessageQueue.IdleHandler() {
    @Override
    public boolean queueIdle() {
        // 空闲时执行
        // 返回 false 表示只执行一次
        // 返回 true 表示每次空闲都执行
        return false;
    }
});
```

### 3. 应用场景

```java
// 延迟初始化
Looper.myQueue().addIdleHandler(() -> {
    initHeavyComponents(); // 初始化重量级组件
    return false;
});

// GC 触发
Looper.myQueue().addIdleHandler(() -> {
    System.gc(); // 空闲时触发 GC
    return false;
});

// 延迟加载
Looper.myQueue().addIdleHandler(() -> {
    loadData(); // 空闲时加载数据
    return false;
});
```

---

## 六、HandlerThread

### 1. 什么是 HandlerThread？
- 继承 Thread
- 内部创建了 Looper
- 适用于后台任务

### 2. 使用方式

```java
// 创建 HandlerThread
HandlerThread handlerThread = new HandlerThread("MyHandlerThread");
handlerThread.start();

// 获取 Handler
Handler handler = new Handler(handlerThread.getLooper()) {
    @Override
    public void handleMessage(Message msg) {
        // 在后台线程处理消息
    }
};

// 发送消息
handler.sendEmptyMessage(1);

// 退出
handlerThread.quitSafely();
```

### 3. 原理分析

```java
public class HandlerThread extends Thread {
    Looper mLooper;
    
    @Override
    public void run() {
        // 创建 Looper
        Looper.prepare();
        
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll(); // 通知等待的线程
        }
        
        // 开始消息循环
        Looper.loop();
    }
    
    public Looper getLooper() {
        if (!isAlive()) {
            return null;
        }
        
        // 等待 Looper 创建完成
        synchronized (this) {
            while (isAlive() && mLooper == null) {
                try {
                    wait();
                } catch (InterruptedException e) {
                }
            }
        }
        return mLooper;
    }
}
```

---

## 七、IntentService

### 1. 什么是 IntentService？
- 继承 Service
- 内部使用 HandlerThread
- 任务执行完自动停止

### 2. 使用方式

```java
public class MyIntentService extends IntentService {
    public MyIntentService() {
        super("MyIntentService");
    }
    
    @Override
    protected void onHandleIntent(Intent intent) {
        // 在后台线程处理任务
        String data = intent.getStringExtra("data");
        // 处理数据
    }
}
```

### 3. 原理分析

```java
public abstract class IntentService extends Service {
    private Looper mServiceLooper;
    private ServiceHandler mServiceHandler;
    
    private final class ServiceHandler extends Handler {
        public ServiceHandler(Looper looper) {
            super(looper);
        }
        
        @Override
        public void handleMessage(Message msg) {
            // 处理任务
            onHandleIntent((Intent) msg.obj);
            
            // 任务完成，停止服务
            stopSelf(msg.arg1);
        }
    }
    
    @Override
    public void onCreate() {
        super.onCreate();
        
        // 创建 HandlerThread
        HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
        thread.start();
        
        mServiceLooper = thread.getLooper();
        mServiceHandler = new ServiceHandler(mServiceLooper);
    }
    
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        // 发送消息到 HandlerThread
        Message msg = mServiceHandler.obtainMessage();
        msg.arg1 = startId;
        msg.obj = intent;
        mServiceHandler.sendMessage(msg);
        
        return START_REDELIVER_INTENT;
    }
}
```

### 4. HandlerThread vs IntentService

| 特性 | HandlerThread | IntentService |
|------|---------------|---------------|
| 类型 | Thread | Service |
| 生命周期 | 手动管理 | 自动停止 |
| 使用场景 | 后台任务 | 异步任务 |
| 消息处理 | 自定义 | onHandleIntent |

---

## 八、Handler 内存泄漏

### 1. 为什么会泄漏？

```java
// 错误示例
public class MainActivity extends Activity {
    private Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            // 处理消息
        }
    };
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        mHandler.sendMessageDelayed(Message.obtain(), 60000);
    }
}
```

**泄漏链路：**
```
主线程 → Looper → MessageQueue → Message → Handler → Activity
```

- Message 持有 Handler 的引用（target 字段）
- 非静态内部类 Handler 持有 Activity 的引用
- Message 在队列中延迟 60 秒
- Activity 销毁后，Message 仍在队列中，导致 Activity 无法回收

### 2. 解决方案

#### 方案一：静态内部类 + 弱引用

```java
public class MainActivity extends Activity {
    private static class MyHandler extends Handler {
        private WeakReference<MainActivity> mActivity;
        
        public MyHandler(MainActivity activity) {
            mActivity = new WeakReference<>(activity);
        }
        
        @Override
        public void handleMessage(Message msg) {
            MainActivity activity = mActivity.get();
            if (activity != null && !activity.isFinishing()) {
                // 处理消息
            }
        }
    }
    
    private MyHandler mHandler = new MyHandler(this);
}
```

#### 方案二：onDestroy 中移除消息

```java
@Override
protected void onDestroy() {
    super.onDestroy();
    mHandler.removeCallbacksAndMessages(null); // 移除所有消息和回调
}
```

#### 方案三：使用 Lifecycle 感知

```java
public class MainActivity extends AppCompatActivity {
    private Handler mHandler = new Handler(Looper.getMainLooper());
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        
        getLifecycle().addObserver(new LifecycleEventObserver() {
            @Override
            public void onStateChanged(LifecycleOwner source, Lifecycle.Event event) {
                if (event == Lifecycle.Event.ON_DESTROY) {
                    mHandler.removeCallbacksAndMessages(null);
                }
            }
        });
    }
}
```

#### 方案四：使用 WeakHandler（不推荐）

```java
// 第三方库
WeakHandler handler = new WeakHandler();
```

---

## 九、Handler 在系统中的应用

### 1. ActivityThread.H

```java
// ActivityThread 中的 Handler
class H extends Handler {
    public static final int LAUNCH_ACTIVITY = 100;
    public static final int PAUSE_ACTIVITY = 101;
    public static final int STOP_ACTIVITY = 102;
    public static final int DESTROY_ACTIVITY = 103;
    
    @Override
    public void handleMessage(Message msg) {
        switch (msg.what) {
            case LAUNCH_ACTIVITY:
                handleLaunchActivity((ActivityClientRecord) msg.obj);
                break;
            case PAUSE_ACTIVITY:
                handlePauseActivity((ActivityClientRecord) msg.obj);
                break;
            // ...
        }
    }
}
```

### 2. ViewRootImpl.ViewRootHandler

```java
// ViewRootImpl 中的 Handler
final class ViewRootHandler extends Handler {
    @Override
    public void handleMessage(Message msg) {
        switch (msg.what) {
            case MSG_INVALIDATE:
                handleInvalidate((View) msg.obj);
                break;
            case MSG_INVALIDATE_RECT:
                handleInvalidateRect((View) msg.obj);
                break;
            // ...
        }
    }
}
```

### 3. WindowManagerService

```java
// WMS 使用 Handler 处理窗口消息
final class H extends Handler {
    public static final int ADD_WINDOW = 1;
    public static final int REMOVE_WINDOW = 2;
    public static final int RELAYOUT_WINDOW = 3;
    
    @Override
    public void handleMessage(Message msg) {
        switch (msg.what) {
            case ADD_WINDOW:
                addWindow((Session) msg.obj, ...);
                break;
            // ...
        }
    }
}
```

---

## 十、面试高频问题

### Q1: Handler、Looper、MessageQueue 的关系？
- 一个线程只能有一个 Looper
- 一个 Looper 只能绑定一个 MessageQueue
- 一个 Handler 可以发送消息到多个 MessageQueue
- 一个 MessageQueue 可以有多个 Handler 发送消息

### Q2: 主线程的 Looper 为什么不会导致 ANR？
- Looper.loop() 是一个无限循环
- ANR 是因为消息处理超时，不是因为循环
- 没有消息时，epoll 会阻塞，不消耗 CPU

### Q3: Handler 的 post 和 sendMessage 的区别？
- post：传入 Runnable，包装成 Message（callback 字段）
- sendMessage：直接发送 Message
- 处理优先级：Message.callback > Handler.mCallback > Handler.handleMessage

### Q4: 如何在子线程创建 Handler？
```java
new Thread(() -> {
    Looper.prepare(); // 创建 Looper
    Handler handler = new Handler(Looper.myLooper());
    Looper.loop(); // 开始循环
}).start();
```

### Q5: IdleHandler 是什么？
- 空闲时执行的任务
- 在 MessageQueue.next() 中调用
- 应用场景：延迟初始化、GC 等

```java
Looper.myQueue().addIdleHandler(() -> {
    // 空闲时执行
    return false; // 返回 false 表示只执行一次
});
```

### Q6: 同步屏障是什么？
- 一种特殊的消息（target 为 null）
- 优先处理异步消息
- 用于保证 UI 绘制的及时性

### Q7: HandlerThread 和 IntentService 的区别？
- HandlerThread：Thread，手动管理生命周期
- IntentService：Service，自动停止
- 都使用 Handler + Looper 处理消息

### Q8: 为什么 Handler 会导致内存泄漏？
- Message 持有 Handler 引用（target 字段）
- 非静态内部类 Handler 持有 Activity 引用
- Message 在队列中延迟执行
- Activity 销毁后，Message 仍在队列中

---

## 十一、最佳实践

### 1. 避免在主线程执行耗时操作
```java
// 使用子线程
new Thread(() -> {
    // 耗时操作
}).start();

// 使用线程池
executor.execute(() -> {
    // 耗时操作
});

// 使用协程
lifecycleScope.launch {
    withContext(Dispatchers.IO) {
        // 耗时操作
    }
}
```

### 2. 及时移除消息
```java
@Override
protected void onDestroy() {
    super.onDestroy();
    handler.removeCallbacksAndMessages(null);
}
```

### 3. 使用静态内部类
```java
private static class MyHandler extends Handler {
    private WeakReference<Activity> activity;
    
    public MyHandler(Activity activity) {
        this.activity = new WeakReference<>(activity);
    }
    
    @Override
    public void handleMessage(Message msg) {
        Activity a = activity.get();
        if (a != null) {
            // 处理消息
        }
    }
}
```

### 4. 合理使用 postDelayed
```java
// 避免过多延迟消息
handler.postDelayed(() -> {
    // 任务
}, 1000);

// 及时移除
handler.removeCallbacks(runnable);
```

### 5. 线程间通信
```java
// 优先考虑 Handler 而非共享变量
// 子线程发送消息到主线程
handler.post(() -> {
    // 更新 UI
});
```
