# 内存优化 - OOM 与内存泄漏

> 性能优化是 Android 面试的重点，内存优化尤为重要！

---

## 一、内存基础

### 1. Android 内存区域

```
┌─────────────────────────────────────────────────────────────┐
│                    Android 内存布局                          │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────┐    │
│  │                    Java 堆 (Heap)                    │    │
│  │  - 对象分配                                          │    │
│  │  - GC 管理                                           │    │
│  │  - 容易 OOM                                          │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                   Native 堆                          │    │
│  │  - C/C++ 对象                                        │    │
│  │  - 手动管理                                          │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                    栈 (Stack)                        │    │
│  │  - 方法调用                                          │    │
│  │  - 自动管理                                          │    │
│  │  - 容易 StackOverflow                                │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                 方法区 (Method Area)                  │    │
│  │  - 类信息                                            │    │
│  │  - 常量池                                            │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

| 区域 | 说明 | 特点 | 管理方式 |
|------|------|------|----------|
| Java 堆 | 对象分配 | GC 管理，容易 OOM | 自动 |
| Native 堆 | C/C++ 对象 | 手动管理 | 手动 |
| 栈 | 方法调用 | 自动管理，容易 StackOverflow | 自动 |
| 方法区 | 类信息 | 存储类元数据 | 自动 |
| 常量池 | 常量 | 存储字符串常量等 | 自动 |

### 2. 内存抖动
- 短时间内大量创建和销毁对象
- 导致频繁 GC，造成卡顿
- 典型场景：在循环或 onDraw 中创建对象

```java
// 内存抖动示例
@Override
protected void onDraw(Canvas canvas) {
    // 错误：每次绘制都创建对象
    Paint paint = new Paint();
    Rect rect = new Rect(0, 0, getWidth(), getHeight());
    canvas.drawRect(rect, paint);
}

// 正确：复用对象
private Paint mPaint = new Paint();
private Rect mRect = new Rect();

@Override
protected void onDraw(Canvas canvas) {
    mRect.set(0, 0, getWidth(), getHeight());
    canvas.drawRect(mRect, mPaint);
}
```

### 3. 内存泄漏
- 对象不再使用，但无法被 GC 回收
- 长生命周期对象持有短生命周期对象的引用
- 导致内存逐渐增加，最终 OOM

### 4. OOM (Out Of Memory)
- 内存不足，无法分配新对象
- 常见原因：内存泄漏、大图片加载、大量数据

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
```
主线程 → Looper → MessageQueue → Message → Handler → Activity

- Message 持有 Handler 引用（target 字段）
- Handler 是非静态内部类，持有 Activity 引用
- Message 在队列中延迟 60 秒
- Activity 销毁后，Message 仍在队列中
```

**解决方案：**
```java
// 方案一：静态内部类 + 弱引用
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

// 方案二：onDestroy 中移除消息
@Override
protected void onDestroy() {
    super.onDestroy();
    mHandler.removeCallbacksAndMessages(null);
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

**泄漏原因：**
- 单例持有 Activity Context
- Activity 销毁后，单例仍然持有引用

**解决方案：**
```java
private AppManager(Context context) {
    this.context = context.getApplicationContext(); // 使用 Application Context
}

// 或者使用 Application
private AppManager(Context context) {
    this.context = context.getApplicationContext();
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
// 方案一：静态内部类
private static class MyTask implements Runnable {
    private WeakReference<MainActivity> mActivity;
    
    public MyTask(MainActivity activity) {
        mActivity = new WeakReference<>(activity);
    }
    
    @Override
    public void run() {
        MainActivity activity = mActivity.get();
        if (activity != null) {
            // 耗时操作
        }
    }
}

// 方案二：使用协程
lifecycleScope.launch {
    withContext(Dispatchers.IO) {
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
```java
// 使用 WeakHashMap
private WeakHashMap<Object, String> weakMap = new WeakHashMap<>();

// 及时清理集合
@Override
protected void onDestroy() {
    super.onDestroy();
    sList.clear();
}
```

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
        // 清理 WebView
        mWebView.loadDataWithBaseURL(null, "", "text/html", "utf-8", null);
        mWebView.clearHistory();
        
        ((ViewGroup) mWebView.getParent()).removeView(mWebView);
        mWebView.destroy();
        mWebView = null;
    }
    super.onDestroy();
}
```

### 7. 动画泄漏

```java
// 错误示例
public class MainActivity extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        
        ObjectAnimator animator = ObjectAnimator.ofFloat(view, "alpha", 0, 1);
        animator.setDuration(60000);
        animator.start();
        // 动画未取消
    }
}
```

**解决方案：**
```java
private ObjectAnimator animator;

@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    
    animator = ObjectAnimator.ofFloat(view, "alpha", 0, 1);
    animator.setDuration(60000);
    animator.start();
}

@Override
protected void onDestroy() {
    super.onDestroy();
    if (animator != null) {
        animator.cancel();
    }
}
```

### 8. 资源未关闭

```java
// 错误示例
public void readFile() {
    FileInputStream fis = new FileInputStream(file);
    // 读取文件
    // 忘记关闭
}
```

**解决方案：**
```java
// 使用 try-with-resources
public void readFile() {
    try (FileInputStream fis = new FileInputStream(file)) {
        // 读取文件
    } catch (IOException e) {
        e.printStackTrace();
    }
}

// 或者在 finally 中关闭
public void readFile() {
    FileInputStream fis = null;
    try {
        fis = new FileInputStream(file);
        // 读取文件
    } catch (IOException e) {
        e.printStackTrace();
    } finally {
        if (fis != null) {
            try {
                fis.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
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

**功能：**
- 实时查看内存使用
- 查看对象分配
- 分析内存泄漏
- 查看 GC 日志

### 2. LeakCanary

**集成：**
```gradle
dependencies {
    debugImplementation 'com.squareup.leakcanary:leakcanary-android:2.12'
}
```

**原理：**
```
1. 监控 Activity/Fragment 的销毁
2. 使用 WeakReference + ReferenceQueue
3. 检测到泄漏时，dump 堆内存并分析
4. 显示泄漏链路
```

**使用：**
- 自动检测 Activity/Fragment 泄漏
- 通知栏显示泄漏信息
- 查看泄漏链路

### 3. MAT (Memory Analyzer Tool)

**使用步骤：**
1. 导出 hprof 文件
```bash
adb shell am dumpheap <pid> /data/local/tmp/heap.hprof
adb pull /data/local/tmp/heap.hprof
```

2. 转换格式
```bash
hprof-conv input.hprof output.hprof
```

3. 用 MAT 打开
4. 分析 Dominator Tree
5. 查找泄漏对象的 GC Root

**功能：**
- Dominator Tree：查看对象占用内存
- Histogram：查看对象数量
- Leak Suspects：自动检测泄漏

### 4. adb 命令

```bash
# 查看内存信息
adb shell dumpsys meminfo <package_name>

# 强制 GC
adb shell kill -10 <pid>

# 导出 hprof
adb shell am dumpheap <pid> /data/local/tmp/heap.hprof
adb pull /data/local/tmp/heap.hprof

# 查看进程内存
adb shell cat /proc/<pid>/status

# 查看 Dalvik 堆
adb shell dumpsys meminfo <pid> | grep -A 20 "Dalvik Heap"
```

### 5. 代码监控

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

// 监控 Native 内存
Debug.getNativeHeapSize(); // Native 堆大小
Debug.getNativeHeapAllocatedSize(); // Native 已分配
Debug.getNativeHeapFreeSize(); // Native 空闲
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
// 图片可能 4000x3000，占用 48MB 内存

// 解决方案：采样压缩
BitmapFactory.Options options = new BitmapFactory.Options();
options.inJustDecodeBounds = true; // 只获取尺寸，不加载
BitmapFactory.decodeFile("/sdcard/large_image.jpg", options);

int imageWidth = options.outWidth;
int imageHeight = options.outHeight;

// 计算采样率
int inSampleSize = 1;
if (imageWidth > reqWidth || imageHeight > reqHeight) {
    int halfWidth = imageWidth / 2;
    int halfHeight = imageHeight / 2;
    while ((halfWidth / inSampleSize) >= reqWidth
            && (halfHeight / inSampleSize) >= reqHeight) {
        inSampleSize *= 2;
    }
}

options.inJustDecodeBounds = false;
options.inSampleSize = inSampleSize;
Bitmap bitmap = BitmapFactory.decodeFile("/sdcard/large_image.jpg", options);
```

#### 数据库查询 OOM
```java
// 错误示例：查询所有数据
Cursor cursor = db.query("table", null, null, null, null, null, null);
// 可能返回数万条数据

// 解决方案：分页查询
int pageSize = 100;
int offset = 0;
Cursor cursor = db.query("table", null, null, null, null, null, null,
    offset + "," + pageSize);
```

#### WebView OOM
```java
// WebView 会占用大量内存
// 解决方案：使用独立进程
<activity
    android:name=".WebViewActivity"
    android:process=":webview" />
```

### 3. OOM 监控

```java
// 监控 Java 堆内存
public class MemoryMonitor {
    private static final double OOM_THRESHOLD = 0.8;
    
    public static boolean isMemoryLow() {
        Runtime runtime = Runtime.getRuntime();
        long maxMemory = runtime.maxMemory();
        long totalMemory = runtime.totalMemory();
        long freeMemory = runtime.freeMemory();
        long usedMemory = totalMemory - freeMemory;
        
        double usageRate = (double) usedMemory / maxMemory;
        return usageRate > OOM_THRESHOLD;
    }
    
    public static void checkMemory() {
        if (isMemoryLow()) {
            // 触发 GC
            System.gc();
            
            // 清理缓存
            clearCache();
            
            // 记录日志
            Log.w("MemoryMonitor", "Memory is low");
        }
    }
}

// 监控 OOM
Thread.setDefaultUncaughtExceptionHandler((thread, throwable) -> {
    if (throwable instanceof OutOfMemoryError) {
        // 记录 OOM 信息
        Log.e("OOM", "Out of Memory", throwable);
        
        // 清理内存
        System.gc();
        
        // 重启应用
        restartApp();
    }
});
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
options.inBitmap = reusableBitmap; // 复用已有 Bitmap 的内存

// 3. 使用 RGB_565 格式
options.inPreferredConfig = Bitmap.Config.RGB_565; // 2 字节/像素，比 ARGB_8888 省一半

// 4. 及时回收
bitmap.recycle();
bitmap = null;

// 5. 使用图片库
// Glide、Coil 等自动处理内存优化
Glide.with(context)
    .load(url)
    .override(200, 200) // 指定尺寸
    .format(DecodeFormat.PREFER_RGB_565) // 使用 RGB_565
    .into(imageView);
```

### 2. 集合优化

```java
// 1. 使用合适的初始容量
HashMap<String, String> map = new HashMap<>(16);

// 2. 使用 WeakHashMap
WeakHashMap<Object, String> weakMap = new WeakHashMap<>();

// 3. 使用 ArrayMap/SparseArray 替代 HashMap
ArrayMap<String, Object> arrayMap = new ArrayMap<>(); // 小数据量更省内存
SparseArray<Object> sparseArray = new SparseArray<>(); // key 为 int 时
LongSparseArray<Object> longSparseArray = new LongSparseArray<>(); // key 为 long 时

// 4. 及时清理
map.clear();
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

// 3. 使用 StringBuilder
StringBuilder sb = new StringBuilder();
sb.append("Hello");
sb.append(" ");
sb.append("World");
String result = sb.toString();
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
int cacheSize = maxMemory / 8; // 使用最大内存的 1/8 作为缓存

LruCache<String, Bitmap> mMemoryCache = new LruCache<String, Bitmap>(cacheSize) {
    @Override
    protected int sizeOf(String key, Bitmap bitmap) {
        return bitmap.getByteCount() / 1024;
    }
    
    @Override
    protected void entryRemoved(boolean evicted, String key, Bitmap oldValue, Bitmap newValue) {
        // 缓存被移除时的回调
    }
};

// 添加缓存
mMemoryCache.put(key, bitmap);

// 获取缓存
Bitmap bitmap = mMemoryCache.get(key);
```

### 6. 使用 DiskLruCache

```java
// 磁盘缓存
DiskLruCache cache = DiskLruCache.open(cacheDir, 1, 1, MAX_CACHE_SIZE);

// 写入缓存
DiskLruCache.Editor editor = cache.edit(key);
OutputStream os = editor.newOutputStream(0);
// 写入数据
editor.commit();

// 读取缓存
DiskLruCache.Snapshot snapshot = cache.get(key);
InputStream is = snapshot.getInputStream(0);
// 读取数据
```

---

## 六、面试高频问题

### Q1: 内存泄漏和内存溢出的区别？
- **内存泄漏**：对象无法被 GC 回收，内存逐渐增加
- **内存溢出**：内存不足，无法分配新对象
- 内存泄漏可能导致内存溢出

### Q2: 如何检测内存泄漏？
- **LeakCanary**：自动检测，开发阶段使用
- **Android Profiler**：手动分析，查看内存曲线
- **MAT**：深度分析堆转储，查找 GC Root

### Q3: 强引用、软引用、弱引用、虚引用的区别？

| 引用类型 | 回收时机 | 使用场景 | 示例 |
|----------|----------|----------|------|
| 强引用 | 永不回收 | 普通对象 | Object obj = new Object() |
| 软引用 | 内存不足时 | 缓存 | SoftReference |
| 弱引用 | GC 时 | 防止内存泄漏 | WeakReference |
| 虚引用 | 随时 | 跟踪 GC | PhantomReference |

```java
// 强引用
Object obj = new Object();

// 软引用
SoftReference<Object> softRef = new SoftReference<>(new Object());
Object obj = softRef.get(); // 内存不足时返回 null

// 弱引用
WeakReference<Object> weakRef = new WeakReference<>(new Object());
Object obj = weakRef.get(); // GC 后返回 null

// 虚引用
PhantomReference<Object> phantomRef = new PhantomReference<>(new Object(), referenceQueue);
phantomRef.get(); // 永远返回 null
```

### Q4: Bitmap 内存计算？
```
内存 = 宽 × 高 × 每像素字节数

ARGB_8888: 4 字节/像素
RGB_565: 2 字节/像素
ALPHA_8: 1 字节/像素

例：1080 × 1920 ARGB_8888
内存 = 1080 × 1920 × 4 = 8,294,400 字节 ≈ 7.9 MB
```

### Q5: 如何优化大图加载？
1. 采样压缩（inSampleSize）
2. 区域解码（BitmapRegionDecoder）
3. 使用 Glide/Coil 等图片库
4. 使用 RGB_565 格式
5. 使用 inBitmap 复用

### Q6: LeakCanary 原理？
```
1. 监控 Activity/Fragment 的 onDestroy
2. 使用 WeakReference 包装 Activity
3. 等待 5 秒，检查 WeakReference 是否被回收
4. 如果未被回收，dump 堆内存
5. 分析 GC Root 到 Activity 的引用链
6. 显示泄漏信息
```

### Q7: 如何避免 OOM？
1. 及时释放资源（Bitmap、Cursor、Stream）
2. 使用弱引用持有 Activity/Fragment
3. 图片加载使用采样压缩
4. 数据库查询分页
5. 使用 LruCache 缓存
6. 监控内存使用

---

## 七、最佳实践

### 1. 使用 LeakCanary
```gradle
debugImplementation 'com.squareup.leakcanary:leakcanary-android:2.12'
```

### 2. 避免静态持有 Activity
```java
// 错误
private static Context context;

// 正确
private static Context context;

public static void init(Context ctx) {
    context = ctx.getApplicationContext();
}
```

### 3. 及时注销监听
```java
@Override
protected void onDestroy() {
    super.onDestroy();
    sensorManager.unregisterListener(this);
    locationManager.removeUpdates(this);
    EventBus.getDefault().unregister(this);
}
```

### 4. 使用弱引用
```java
private WeakReference<Activity> activityRef;

public MyAdapter(Activity activity) {
    this.activityRef = new WeakReference<>(activity);
}

public void doSomething() {
    Activity activity = activityRef.get();
    if (activity != null && !activity.isFinishing()) {
        // 使用 activity
    }
}
```

### 5. 图片优化
```java
// 使用 Glide
Glide.with(context)
    .load(url)
    .override(200, 200)
    .format(DecodeFormat.PREFER_RGB_565)
    .into(imageView);
```

### 6. 对象复用
```java
// 使用对象池
Message msg = Message.obtain();
// 使用 LruCache
LruCache<String, Bitmap> cache = new LruCache<>(maxSize);
```

### 7. 监控内存
```java
// 代码监控
Runtime runtime = Runtime.getRuntime();
long usedMemory = runtime.totalMemory() - runtime.freeMemory();
long maxMemory = runtime.maxMemory();
double usage = (double) usedMemory / maxMemory;

if (usage > 0.8) {
    Log.w("Memory", "Memory usage is high: " + usage);
    System.gc();
}
```
