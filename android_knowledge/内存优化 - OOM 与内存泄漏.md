# 内存优化 - OOM 与内存泄漏

> 性能优化是 Android 面试的重点，内存优化尤为重要！

---

## 一、内存基础

### 1. Android 内存区域

| 区域 | 说明 | 特点 |
|------|------|------|
| Java 堆 | 对象分配 | GC 管理，容易 OOM |
| Native 堆 | C/C++ 对象 | 手动管理 |
| 栈 | 方法调用 | 自动管理，容易 StackOverflow |
| 方法区 | 类信息 | 存储类元数据 |
| 常量池 | 常量 | 存储字符串常量等 |

### 2. 内存抖动
- 短时间内大量创建和销毁对象
- 导致频繁 GC，造成卡顿
- 典型场景：在循环或 onDraw 中创建对象

### 3. 内存泄漏
- 对象不再使用，但无法被 GC 回收
- 长生命周期对象持有短生命周期对象的引用
- 导致内存逐渐增加，最终 OOM

---

## 二、常见内存泄漏场景

### 1. Handler 泄漏

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

**泄漏原因：**
- Message 持有 Handler 引用
- Handler 是非静态内部类，持有 Activity 引用
- Message 在队列中延迟 60 秒
- Activity 销毁后，Message 仍在队列中

**解决方案：**
```java
// 使用静态内部类 + 弱引用
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
```

### 2. 单例模式泄漏

```java
// 错误示例
public class AppManager {
    private static AppManager instance;
    private Context context;
    
    private AppManager(Context context) {
        this.context = context; // 持有 Activity Context
    }
    
    public static AppManager getInstance(Context context) {
        if (instance == null) {
            instance = new AppManager(context);
        }
        return instance;
    }
}
```

**解决方案：**
```java
private AppManager(Context context) {
    this.context = context.getApplicationContext(); // 使用 Application Context
}
```

### 3. 匿名内部类泄漏

```java
// 错误示例
public class MainActivity extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        
        new Thread(new Runnable() {
            @Override
            public void run() {
                // 耗时操作
                SystemClock.sleep(60000);
            }
        }).start();
    }
}
```

**泄漏原因：**
- 匿名内部类持有 Activity 引用
- Thread 生命周期比 Activity 长

**解决方案：**
```java
// 使用静态内部类
private static class MyTask implements Runnable {
    private WeakReference<MainActivity> mActivity;
    
    public MyTask(MainActivity activity) {
        mActivity = new WeakReference<>(activity);
    }
    
    @Override
    public void run() {
        // 耗时操作
    }
}
```

### 4. 注册监听未注销

```java
// 错误示例
public class MainActivity extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        
        SensorManager manager = (SensorManager) getSystemService(SENSOR_SERVICE);
        Sensor sensor = manager.getDefaultSensor(Sensor.TYPE_ACCELEROMETER);
        manager.registerListener(this, sensor, SensorManager.SENSOR_DELAY_NORMAL);
    }
    
    // 忘记注销
}
```

**解决方案：**
```java
@Override
protected void onDestroy() {
    super.onDestroy();
    sensorManager.unregisterListener(this);
}
```

### 5. 集合类泄漏

```java
// 错误示例
public class MainActivity extends Activity {
    private static List<Object> sList = new ArrayList<>();
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        sList.add(new Object()); // 静态集合持有对象引用
    }
}
```

**解决方案：**
- 使用 WeakHashMap
- 及时清理集合
- 避免静态集合持有 Activity 引用

### 6. WebView 泄漏

```java
// 错误示例
public class WebViewActivity extends Activity {
    private WebView mWebView;
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        mWebView = new WebView(this);
        setContentView(mWebView);
    }
}
```

**解决方案：**
```java
@Override
protected void onDestroy() {
    if (mWebView != null) {
        mWebView.loadDataWithBaseURL(null, "", "text/html", "utf-8", null);
        mWebView.clearHistory();
        
        ((ViewGroup) mWebView.getParent()).removeView(mWebView);
        mWebView.destroy();
        mWebView = null;
    }
    super.onDestroy();
}
```

---

## 三、内存分析工具

### 1. Android Profiler

**使用步骤：**
1. 打开 Android Studio
2. View → Tool Windows → Profiler
3. 选择设备和进程
4. 查看 Memory 实时曲线
5. 点击 Record 按钮录制内存分配
6. 分析堆转储

### 2. LeakCanary

**集成：**
```gradle
dependencies {
    debugImplementation 'com.squareup.leakcanary:leakcanary-android:2.12'
}
```

**原理：**
- 监控 Activity/Fragment 的销毁
- 使用 WeakReference + ReferenceQueue
- 检测到泄漏时，dump 堆内存并分析
- 显示泄漏链路

### 3. MAT (Memory Analyzer Tool)

**使用步骤：**
1. 导出 hprof 文件
2. 转换格式：`hprof-conv input.hprof output.hprof`
3. 用 MAT 打开
4. 分析 Dominator Tree
5. 查找泄漏对象的 GC Root

### 4. adb 命令

```bash
# 查看内存信息
adb shell dumpsys meminfo <package_name>

# 强制 GC
adb shell kill -10 <pid>

# 导出 hprof
adb shell am dumpheap <pid> /data/local/tmp/heap.hprof
adb pull /data/local/tmp/heap.hprof
```

---

## 四、OOM 分析

### 1. OOM 原因
- 内存泄漏导致内存持续增加
- 大图片加载
- 大文件读取
- 数据库查询返回大量数据

### 2. OOM 场景

#### 图片加载 OOM
```java
// 错误示例：加载大图片
Bitmap bitmap = BitmapFactory.decodeFile("/sdcard/large_image.jpg");

// 解决方案：采样压缩
BitmapFactory.Options options = new BitmapFactory.Options();
options.inSampleSize = 2; // 缩小 2 倍
Bitmap bitmap = BitmapFactory.decodeFile("/sdcard/large_image.jpg", options);
```

#### 数据库查询 OOM
```java
// 错误示例：查询所有数据
Cursor cursor = db.query("table", null, null, null, null, null, null);

// 解决方案：分页查询
Cursor cursor = db.query("table", null, null, null, null, null, null, "0,100");
```

### 3. OOM 监控

```java
// 监控 Java 堆内存
Runtime runtime = Runtime.getRuntime();
long maxMemory = runtime.maxMemory(); // 最大内存
long totalMemory = runtime.totalMemory(); // 已分配内存
long freeMemory = runtime.freeMemory(); // 空闲内存
long usedMemory = totalMemory - freeMemory; // 已使用内存

// 内存使用率
double usageRate = (double) usedMemory / maxMemory;

if (usageRate > 0.8) {
    // 内存紧张，触发 GC 或清理缓存
    System.gc();
}
```

---

## 五、内存优化策略

### 1. 图片优化

```java
// 1. 采样压缩
BitmapFactory.Options options = new BitmapFactory.Options();
options.inSampleSize = calculateInSampleSize(options, reqWidth, reqHeight);
Bitmap bitmap = BitmapFactory.decodeResource(getResources(), resId, options);

// 2. 使用 inBitmap 复用
options.inMutable = true;
options.inBitmap = reusableBitmap;

// 3. 使用 RGB_565 格式
options.inPreferredConfig = Bitmap.Config.RGB_565;

// 4. 及时回收
bitmap.recycle();
```

### 2. 集合优化

```java
// 1. 使用合适的初始容量
HashMap<String, String> map = new HashMap<>(16);

// 2. 使用 WeakHashMap
WeakHashMap<Object, String> weakMap = new WeakHashMap<>();

// 3. 使用 ArrayMap/SparseArray 替代 HashMap
ArrayMap<String, Object> arrayMap = new ArrayMap<>();
SparseArray<Object> sparseArray = new SparseArray<>();
```

### 3. 对象复用

```java
// 1. 使用对象池
private static final int MAX_POOL_SIZE = 10;
private Queue<MyObject> mPool = new LinkedList<>();

public MyObject obtain() {
    MyObject obj = mPool.poll();
    if (obj == null) {
        obj = new MyObject();
    }
    return obj;
}

public void recycle(MyObject obj) {
    if (mPool.size() < MAX_POOL_SIZE) {
        obj.reset();
        mPool.offer(obj);
    }
}

// 2. 使用 Message.obtain()
Message msg = Message.obtain(); // 从对象池获取
```

### 4. 避免内存抖动

```java
// 错误示例
@Override
protected void onDraw(Canvas canvas) {
    Paint paint = new Paint(); // 每次绘制都创建
    canvas.drawRect(0, 0, getWidth(), getHeight(), paint);
}

// 正确示例
private Paint mPaint = new Paint(); // 构造函数中创建

@Override
protected void onDraw(Canvas canvas) {
    canvas.drawRect(0, 0, getWidth(), getHeight(), mPaint);
}
```

### 5. 使用 LruCache

```java
int maxMemory = (int) (Runtime.getRuntime().maxMemory() / 1024);
int cacheSize = maxMemory / 8;

LruCache<String, Bitmap> mMemoryCache = new LruCache<String, Bitmap>(cacheSize) {
    @Override
    protected int sizeOf(String key, Bitmap bitmap) {
        return bitmap.getByteCount() / 1024;
    }
};

// 添加缓存
mMemoryCache.put(key, bitmap);

// 获取缓存
Bitmap bitmap = mMemoryCache.get(key);
```

---

## 六、面试高频问题

### Q1: 内存泄漏和内存溢出的区别？
- **内存泄漏**：对象无法被 GC 回收
- **内存溢出**：内存不足，无法分配新对象
- 内存泄漏可能导致内存溢出

### Q2: 如何检测内存泄漏？
- LeakCanary：自动检测
- Android Profiler：手动分析
- MAT：深度分析堆转储

### Q3: 强引用、软引用、弱引用、虚引用的区别？
| 引用类型 | 回收时机 | 使用场景 |
|----------|----------|----------|
| 强引用 | 永不回收 | 普通对象 |
| 软引用 | 内存不足时 | 缓存 |
| 弱引用 | GC 时 | 防止内存泄漏 |
| 虚引用 | 随时 | 跟踪 GC |

### Q4: Bitmap 内存计算？
```
内存 = 宽 × 高 × 每像素字节数

ARGB_8888: 4 字节/像素
RGB_565: 2 字节/像素

例：1080 × 1920 ARGB_8888
内存 = 1080 × 1920 × 4 = 8,294,400 字节 ≈ 7.9 MB
```

### Q5: 如何优化大图加载？
1. 采样压缩（inSampleSize）
2. 区域解码（BitmapRegionDecoder）
3. 使用 Glide/Coil 等图片库
4. 使用 RGB_565 格式

---

## 七、最佳实践

1. **使用 LeakCanary**：开发阶段自动检测泄漏
2. **避免静态持有 Activity**：使用 Application Context
3. **及时注销监听**：在 onDestroy 中注销
4. **使用弱引用**：避免长生命周期对象持有短生命周期对象
5. **图片优化**：采样压缩、格式选择、及时回收
6. **对象复用**：使用对象池、LruCache
7. **监控内存**：使用 Profiler 监控内存使用
8. **定期 dump 堆内存**：分析内存使用情况
