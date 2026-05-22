# Jetpack 组件原理

> Jetpack 是 Android 开发的核心，面试必问！

---

## 一、Lifecycle

### 1. 作用
- 感知组件（Activity/Fragment）的生命周期
- 避免内存泄漏
- 代码解耦

### 2. 核心类

```java
// 生命周期持有者
public interface LifecycleOwner {
    Lifecycle getLifecycle();
}

// 生命周期观察者
public interface LifecycleObserver {
}

// 生命周期事件
public enum Event {
    ON_CREATE, ON_START, ON_RESUME, ON_PAUSE, ON_STOP, ON_DESTROY
}

// 生命周期状态
public enum State {
    CREATED, STARTED, RESUMED, DESTROYED
}
```

### 3. 使用方式

```java
// 实现 LifecycleObserver
public class MyObserver implements LifecycleObserver {
    
    @OnLifecycleEvent(Lifecycle.Event.ON_CREATE)
    public void onCreate() {
        // Activity onCreate
    }
    
    @OnLifecycleEvent(Lifecycle.Event.ON_DESTROY)
    public void onDestroy() {
        // Activity onDestroy
    }
}

// 注册观察者
public class MainActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        getLifecycle().addObserver(new MyObserver());
    }
}
```

### 4. 原理分析

```java
// ReportFragment 注入
public class ReportFragment extends Fragment {
    
    public static void injectIfNeededIn(Activity activity) {
        if (Build.VERSION.SDK_INT >= 29) {
            // API 29+ 使用 Activity.registerActivityLifecycleCallbacks
            activity.registerActivityLifecycleCallbacks(
                new LifecycleCallbacks()
            );
        }
        
        // 添加 ReportFragment
        android.app.FragmentManager manager = activity.getFragmentManager();
        if (manager.findFragmentByTag(TAG) == null) {
            manager.beginTransaction()
                .add(new ReportFragment(), TAG)
                .commit();
            manager.executePendingTransactions();
        }
    }
    
    // 在生命周期回调中分发事件
    @Override
    public void onActivityCreated(Bundle savedInstanceState) {
        dispatchCreate(activity);
    }
    
    private void dispatchCreate(Activity activity) {
        if (activity instanceof LifecycleRegistryOwner) {
            ((LifecycleRegistryOwner) activity).getLifecycle()
                .handleLifecycleEvent(Lifecycle.Event.ON_CREATE);
        }
    }
}
```

**原理总结：**
- 通过 ReportFragment 注入生命周期回调
- 在各个生命周期方法中分发事件
- LifecycleRegistry 管理状态和观察者

---

## 二、ViewModel

### 1. 作用
- 存储和管理 UI 相关数据
- 配置更改（如旋转屏幕）时保留数据
- 避免在 Activity 中存储数据

### 2. 使用方式

```java
public class MyViewModel extends ViewModel {
    private MutableLiveData<List<User>> users;
    
    public LiveData<List<User>> getUsers() {
        if (users == null) {
            users = new MutableLiveData<>();
            loadUsers();
        }
        return users;
    }
    
    private void loadUsers() {
        // 加载数据
    }
}

// Activity 中使用
public class MainActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        
        MyViewModel model = new ViewModelProvider(this).get(MyViewModel.class);
        model.getUsers().observe(this, users -> {
            // 更新 UI
        });
    }
}
```

### 3. 原理分析

```java
// ViewModelStore
public class ViewModelStore {
    private final HashMap<String, ViewModel> mMap = new HashMap<>();
    
    final void put(String key, ViewModel viewModel) {
        ViewModel oldViewModel = mMap.put(key, viewModel);
        if (oldViewModel != null) {
            oldViewModel.onCleared();
        }
    }
    
    final ViewModel get(String key) {
        return mMap.get(key);
    }
    
    public final void clear() {
        for (ViewModel vm : mMap.values()) {
            vm.onCleared();
        }
        mMap.clear();
    }
}

// ViewModelStoreOwner
public interface ViewModelStoreOwner {
    ViewModelStore getViewModelStore();
}

// Activity 实现 ViewModelStoreOwner
public class ComponentActivity extends Activity implements ViewModelStoreOwner {
    private ViewModelStore mViewModelStore;
    
    @Override
    public ViewModelStore getViewModelStore() {
        if (mViewModelStore == null) {
            // 从 NonConfigurationInstances 中恢复
            NonConfigurationInstances nc = (NonConfigurationInstances) getLastNonConfigurationInstance();
            if (nc != null) {
                mViewModelStore = nc.viewModelStore;
            }
            if (mViewModelStore == null) {
                mViewModelStore = new ViewModelStore();
            }
        }
        return mViewModelStore;
    }
    
    // 配置更改时保存
    @Override
    public final Object onRetainNonConfigurationInstance() {
        return new NonConfigurationInstances(mViewModelStore);
    }
}
```

**原理总结：**
- ViewModelStore 存储 ViewModel 实例
- 配置更改时，通过 onRetainNonConfigurationInstance 保存 ViewModelStore
- Activity 重建后，从 getLastNonConfigurationInstance 恢复
- Activity 真正销毁时，调用 ViewModel.onCleared()

---

## 三、LiveData

### 1. 作用
- 可观察的数据持有者
- 具有生命周期感知能力
- 只在活跃状态下通知观察者

### 2. 使用方式

```java
// 创建 LiveData
public class UserViewModel extends ViewModel {
    private MutableLiveData<User> userLiveData = new MutableLiveData<>();
    
    public LiveData<User> getUser() {
        return userLiveData;
    }
    
    public void setUser(User user) {
        userLiveData.setValue(user);
    }
}

// 观察 LiveData
userViewModel.getUser().observe(this, user -> {
    // 更新 UI
});
```

### 3. 原理分析

```java
public abstract class LiveData<T> {
    
    // 观察者包装
    private SafeIterableMap<Observer<? super T>, ObserverWrapper> mObservers = new SafeIterableMap<>();
    
    // 版本号
    private int mVersion = START_VERSION;
    
    // 添加观察者
    public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
        // 创建生命周期感知的观察者
        LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
        
        ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
        
        // 注册生命周期观察者
        owner.getLifecycle().addObserver(wrapper);
    }
    
    // 生命周期感知的观察者
    class LifecycleBoundObserver extends ObserverWrapper implements LifecycleEventObserver {
        final LifecycleOwner mOwner;
        
        @Override
        public void onStateChanged(LifecycleOwner source, Lifecycle.Event event) {
            if (mOwner.getLifecycle().getCurrentState() == DESTROYED) {
                removeObserver(mObserver); // 自动移除
                return;
            }
            activeStateChanged(shouldBeActive());
        }
        
        @Override
        boolean shouldBeActive() {
            return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED);
        }
    }
    
    // 设置值
    protected void setValue(T value) {
        assertMainThread("setValue");
        mVersion++;
        mData = value;
        dispatchingValue(null);
    }
    
    // 分发值
    void dispatchingValue(@Nullable ObserverWrapper initiator) {
        for (Iterator<Map.Entry<Observer<? super T>, ObserverWrapper>> iterator = 
             mObservers.iteratorWithAdditions(); iterator.hasNext(); ) {
            
            ObserverWrapper wrapper = iterator.next().getValue();
            if (wrapper.mActive) {
                considerNotify(wrapper); // 通知观察者
            }
        }
    }
    
    // 通知观察者
    private void considerNotify(ObserverWrapper observer) {
        if (!observer.mActive) {
            return;
        }
        
        if (!observer.shouldBeActive()) {
            observer.activeStateChanged(false);
            return;
        }
        
        if (observer.mLastVersion >= mVersion) {
            return; // 已经通知过
        }
        
        observer.mLastVersion = mVersion;
        observer.mObserver.onChanged(mData); // 回调
    }
}
```

**原理总结：**
- LifecycleBoundObserver 包装观察者，感知生命周期
- 只有在 STARTED 或 RESUMED 状态才通知
- 生命周期 DESTROYED 时自动移除观察者
- 通过版本号避免重复通知

---

## 四、Room

### 1. 作用
- SQLite 的抽象层
- 编译时验证 SQL 语句
- 支持 LiveData 和 RxJava

### 2. 使用方式

```java
// 定义 Entity
@Entity(tableName = "users")
public class User {
    @PrimaryKey
    public int id;
    
    @ColumnInfo(name = "name")
    public String name;
    
    @ColumnInfo(name = "age")
    public int age;
}

// 定义 DAO
@Dao
public interface UserDao {
    @Query("SELECT * FROM users")
    LiveData<List<User>> getAllUsers();
    
    @Insert
    void insert(User user);
    
    @Update
    void update(User user);
    
    @Delete
    void delete(User user);
}

// 定义 Database
@Database(entities = {User.class}, version = 1)
public abstract class AppDatabase extends RoomDatabase {
    public abstract UserDao userDao();
}

// 使用
AppDatabase db = Room.databaseBuilder(
    getApplicationContext(),
    AppDatabase.class,
    "database-name"
).build();

UserDao userDao = db.userDao();
userDao.insert(new User(1, "John", 25));
```

### 3. 原理分析

- **编译时注解处理**：生成 DAO 实现类
- **SQLiteOpenHelper**：管理数据库创建和升级
- **线程安全**：数据库操作必须在后台线程

---

## 五、Navigation

### 1. 作用
- 管理 Fragment 导航
- 处理 Fragment 事务
- 支持深层链接

### 2. 使用方式

```xml
<!-- nav_graph.xml -->
<navigation xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/nav_graph"
    app:startDestination="@id/homeFragment">
    
    <fragment
        android:id="@+id/homeFragment"
        android:name="com.example.HomeFragment"
        android:label="Home">
        <action
            android:id="@+id/action_home_to_detail"
            app:destination="@id/detailFragment" />
    </fragment>
    
    <fragment
        android:id="@+id/detailFragment"
        android:name="com.example.DetailFragment"
        android:label="Detail" />
</navigation>
```

```java
// 导航
NavHostFragment.findNavController(this)
    .navigate(R.id.action_home_to_detail);
```

### 3. 原理分析

- **NavHostFragment**：承载导航的 Fragment
- **NavController**：管理导航栈
- **NavGraph**：定义导航图

---

## 六、WorkManager

### 1. 作用
- 管理后台任务
- 保证任务执行（即使应用退出）
- 支持约束条件

### 2. 使用方式

```java
// 定义 Worker
public class UploadWorker extends Worker {
    public UploadWorker(Context context, WorkerParameters params) {
        super(context, params);
    }
    
    @Override
    public Result doWork() {
        // 执行后台任务
        uploadData();
        return Result.success();
    }
}

// 创建任务
WorkRequest uploadWork = new OneTimeWorkRequestBuilder<UploadWorker>()
    .setConstraints(new Constraints.Builder()
        .setRequiredNetworkType(NetworkType.CONNECTED)
        .build())
    .build();

// 提交任务
WorkManager.getInstance(context).enqueue(uploadWork);
```

### 3. 原理分析

- **JobScheduler**：API 23+ 使用
- **AlarmManager + BroadcastReceiver**：API 14-22 使用
- **持久化**：任务信息存储在数据库中

---

## 七、面试高频问题

### Q1: ViewModel 为什么能在配置更改时保留数据？
- 通过 onRetainNonConfigurationInstance 保存 ViewModelStore
- Activity 重建后从 getLastNonConfigurationInstance 恢复
- 只有 Activity 真正销毁时才调用 onCleared()

### Q2: LiveData 如何感知生命周期？
- 通过 LifecycleBoundObserver 包装观察者
- 在 onStateChanged 中检查生命周期状态
- 只有 STARTED 或 RESUMED 状态才通知

### Q3: 如何避免 LiveData 内存泄漏？
- LiveData 自动在 DESTROYED 时移除观察者
- 使用 observe(owner, observer) 而不是 observeForever()

### Q4: Room 和 SQLiteOpenHelper 的区别？
- Room 提供编译时 SQL 验证
- Room 支持 LiveData/RxJava
- Room 简化了数据库操作

### Q5: WorkManager 和 AlarmManager 的区别？
- WorkManager 保证任务执行
- WorkManager 支持约束条件
- WorkManager 适配不同 API 版本
