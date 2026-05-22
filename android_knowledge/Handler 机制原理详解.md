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

### 1. 发送消息

```java
// Handler 发送消息
handler.sendMessage(message);
handler.sendMessageDelayed(message, 1000);
handler.post(runnable);

// 最终都调用 sendMessageAtTime()
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}

private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this; // 设置消息的 target 为当前 Handler
    
    // 设置发送时间
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

### 2. MessageQueue 入队

```java
boolean enqueueMessage(Message msg, long when) {
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

### 3. Looper 循环

```java
public static void loop() {
    final Looper me = myLooper();
    final MessageQueue queue = me.mQueue;
    
    for (;;) {
        // 1. 取消息（可能会阻塞）
        Message msg = queue.next();
        
        if (msg == null) {
            return; // 退出循环
        }
        
        // 2. 分发消息给 Handler
        msg.target.dispatchMessage(msg);
        
        // 3. 回收消息
        msg.recycleUnchecked();
    }
}
```

### 4. 处理消息

```java
// Handler.dispatchMessage()
public void dispatchMessage(Message msg) {
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

---

## 三、同步屏障

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
```

### 3. 取消息时的处理

```java
Message next() {
    for (;;) {
        synchronized (this) {
            Message msg = mMessages;
            
            // 如果队头是同步屏障
            if (msg != null && msg.target == null) {
                // 跳过同步消息，找到异步消息
                do {
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            
            if (msg != null) {
                if (now < msg.when) {
                    // 还没到执行时间，计算等待时间
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // 取出消息返回
                    mMessages = msg.next;
                    msg.next = null;
                    return msg;
                }
            } else {
                nextPollTimeoutMillis = -1; // 无限等待
            }
        }
        
        // 阻塞等待（native 层使用 epoll）
        nativePollOnce(ptr, nextPollTimeoutMillis);
    }
}
```

### 4. 应用场景
- ViewRootImpl.scheduleTraversals()
- 确保 UI 绘制（异步消息）优先执行

---

## 四、Handler 内存泄漏

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
            if (activity != null) {
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
    mHandler.removeCallbacksAndMessages(null); // 移除所有消息
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

---

## 五、面试高频问题

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

---

## 六、最佳实践

1. **避免在主线程执行耗时操作**：使用子线程或协程
2. **及时移除消息**：在 onDestroy 中调用 removeCallbacksAndMessages
3. **使用静态内部类**：避免内存泄漏
4. **合理使用 postDelayed**：避免过多延迟消息
5. **线程间通信**：优先考虑 Handler 而非共享变量
