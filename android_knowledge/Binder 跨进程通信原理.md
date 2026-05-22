# Binder 跨进程通信原理

> Binder 是 Android 最核心的 IPC 机制，面试必问！

---

## 一、为什么使用 Binder？

### 1. 传统 IPC 方式对比

| 方式 | 数据拷贝次数 | 特点 | 安全性 |
|------|-------------|------|--------|
| 共享内存 | 0 | 控制复杂 | 低 |
| 管道/消息队列 | 2 | 效率低 | 低 |
| Socket | 2 | 开销大 | 低 |
| Binder | 1 | Android 特有 | 高 |

### 2. Binder 的优势
- **性能好**：只需一次数据拷贝（mmap）
- **安全性高**：通过 UID/PID 验证身份
- **使用简单**：AIDL 接口定义

### 3. 为什么不用共享内存？
- 共享内存控制复杂，需要同步机制
- Binder 提供了封装好的接口，使用简单
- Binder 有安全校验机制

---

## 二、Binder 通信原理

### 1. 跨进程通信模型

```
┌─────────────────────────────────────────────────────────────┐
│                      Binder 通信模型                         │
├─────────────────────────────────────────────────────────────┤
│  ┌──────────┐         ┌──────────┐         ┌──────────┐    │
│  │  Client  │         │  Server  │         │ Service  │    │
│  │  进程    │         │  进程    │         │ Manager  │    │
│  └────┬─────┘         └────┬─────┘         └────┬─────┘    │
│       │                    │                    │           │
│       │         ┌─────────────────┐            │           │
│       │         │   Binder 驱动   │            │           │
│       │         │   (内核空间)    │            │           │
│       │         └────────┬────────┘            │           │
│       │                  │                     │           │
│       └──────────────────┼─────────────────────┘           │
│                          │                                  │
└──────────────────────────┼──────────────────────────────────┘
                           │
              ┌────────────┴────────────┐
              │      物理内存 (mmap)      │
              └─────────────────────────┘
```

### 2. 一次拷贝原理

```
传统 IPC（2次拷贝）：
┌──────────┐    ┌──────────────┐    ┌──────────┐
│ 用户空间A │ →  │   内核空间   │ →  │ 用户空间B │
│  发送方   │    │   (复制1)   │    │  接收方   │
└──────────┘    └──────────────┘    └──────────┘
   复制1              复制2

Binder（1次拷贝）：
┌──────────┐    ┌──────────────┐
│ 用户空间A │ →  │   内核空间   │ ←─┐
│  发送方   │    │   (复制1)   │   │ mmap 映射
└──────────┘    └──────────────┘   │
                    ↑              │
                    └──────────────┘
                    ┌──────────┐
                    │ 用户空间B │
                    │  接收方   │
                    └──────────┘
```

**mmap 原理：**
- 将内核空间和接收方用户空间映射到同一块物理内存
- 发送方数据拷贝到内核空间后，接收方直接访问
- 只需要一次数据拷贝

### 3. Binder 通信流程

```
1. Server 启动，注册服务到 ServiceManager
2. Client 从 ServiceManager 获取服务的 Binder 代理
3. Client 通过 Binder 代理发送请求
4. Binder 驱动将请求转发给 Server
5. Server 处理请求，返回结果
6. Binder 驱动将结果返回给 Client
```

---

## 三、AIDL 使用

### 1. 定义 AIDL 接口

```aidl
// IMyService.aidl
package com.example;

interface IMyService {
    // 基本类型
    String getData(int id);
    void setData(int id, String data);
    
    // 自定义对象（需要 Parcelable）
    User getUser(int id);
    void setUser(in User user);
    
    // List
    List<User> getUsers();
    
    // 回调接口
    void registerCallback(IMyCallback callback);
    void unregisterCallback(IMyCallback callback);
}
```

```aidl
// IMyCallback.aidl
package com.example;

interface IMyCallback {
    void onDataChanged(String data);
}
```

```java
// User.java - Parcelable 对象
public class User implements Parcelable {
    private int id;
    private String name;
    
    protected User(Parcel in) {
        id = in.readInt();
        name = in.readString();
    }
    
    public static final Creator<User> CREATOR = new Creator<User>() {
        @Override
        public User createFromParcel(Parcel in) {
            return new User(in);
        }
        
        @Override
        public User[] newArray(int size) {
            return new User[size];
        }
    };
    
    @Override
    public int describeContents() {
        return 0;
    }
    
    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeInt(id);
        dest.writeString(name);
    }
}
```

### 2. 服务端实现

```java
public class MyService extends Service {
    private final IMyService.Stub mBinder = new IMyService.Stub() {
        @Override
        public String getData(int id) throws RemoteException {
            return "Data " + id;
        }
        
        @Override
        public void setData(int id, String data) throws RemoteException {
            // 处理数据
        }
        
        @Override
        public User getUser(int id) throws RemoteException {
            return new User(id, "User " + id);
        }
        
        @Override
        public void setUser(User user) throws RemoteException {
            // 处理用户
        }
        
        @Override
        public List<User> getUsers() throws RemoteException {
            List<User> users = new ArrayList<>();
            users.add(new User(1, "张三"));
            users.add(new User(2, "李四"));
            return users;
        }
        
        private List<IMyCallback> mCallbacks = new ArrayList<>();
        
        @Override
        public void registerCallback(IMyCallback callback) throws RemoteException {
            if (callback != null) {
                mCallbacks.add(callback);
            }
        }
        
        @Override
        public void unregisterCallback(IMyCallback callback) throws RemoteException {
            if (callback != null) {
                mCallbacks.remove(callback);
            }
        }
        
        private void notifyDataChanged(String data) {
            for (IMyCallback callback : mCallbacks) {
                callback.onDataChanged(data);
            }
        }
    };
    
    @Override
    public IBinder onBind(Intent intent) {
        return mBinder;
    }
}
```

### 3. 客户端调用

```java
public class MainActivity extends Activity {
    private IMyService mService;
    private boolean mBound = false;
    
    private ServiceConnection mConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            // 将 IBinder 转换为 IMyService
            mService = IMyService.Stub.asInterface(service);
            mBound = true;
            
            // 注册死亡代理
            try {
                service.linkToDeath(mDeathRecipient, 0);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }
        
        @Override
        public void onServiceDisconnected(ComponentName name) {
            mService = null;
            mBound = false;
        }
    };
    
    // 死亡代理
    private IBinder.DeathRecipient mDeathRecipient = new IBinder.DeathRecipient() {
        @Override
        public void binderDied() {
            // 服务进程死亡，重新绑定
            if (mService != null) {
                mService.asBinder().unlinkToDeath(mDeathRecipient, 0);
                mService = null;
            }
            bindService();
        }
    };
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        
        // 绑定服务
        bindService();
    }
    
    private void bindService() {
        Intent intent = new Intent(this, MyService.class);
        bindService(intent, mConnection, Context.BIND_AUTO_CREATE);
    }
    
    @Override
    protected void onDestroy() {
        super.onDestroy();
        if (mBound) {
            unbindService(mConnection);
            mBound = false;
        }
    }
    
    private void callService() {
        if (mBound) {
            try {
                // 调用服务方法
                String data = mService.getData(1);
                
                User user = mService.getUser(1);
                
                List<User> users = mService.getUsers();
                
                // 注册回调
                IMyCallback.Stub callback = new IMyCallback.Stub() {
                    @Override
                    public void onDataChanged(String data) throws RemoteException {
                        runOnUiThread(() -> {
                            // 更新 UI
                        });
                    }
                };
                mService.registerCallback(callback);
                
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }
    }
}
```

### 4. 权限验证

```java
// AndroidManifest.xml
<permission
    android:name="com.example.MY_SERVICE_PERMISSION"
    android:protectionLevel="normal" />

<service
    android:name=".MyService"
    android:permission="com.example.MY_SERVICE_PERMISSION">
    <intent-filter>
        <action android:name="com.example.IMyService" />
    </intent-filter>
</service>
```

```java
// 服务端验证权限
@Override
public IBinder onBind(Intent intent) {
    // 检查调用者权限
    int check = checkCallingOrSelfPermission("com.example.MY_SERVICE_PERMISSION");
    if (check == PackageManager.PERMISSION_DENIED) {
        return null; // 权限不足
    }
    return mBinder;
}
```

---

## 四、Binder 核心类

### 1. IBinder
```java
// Binder 通信的基本接口
public interface IBinder {
    // 获取本地 Binder 对象
    public IInterface queryLocalInterface(String descriptor);
    
    // 发送事务
    public boolean transact(int code, Parcel data, Parcel reply, int flags)
            throws RemoteException;
    
    // 注册死亡代理
    public void linkToDeath(DeathRecipient recipient, int flags)
            throws RemoteException;
    
    // 注销死亡代理
    public boolean unlinkToDeath(DeathRecipient recipient, int flags);
}
```

### 2. Binder（服务端）
```java
// 服务端基类
public class Binder implements IBinder {
    // 处理客户端请求
    protected boolean onTransact(int code, Parcel data, Parcel reply, int flags)
            throws RemoteException {
        // 子类重写此方法处理请求
        return false;
    }
    
    // 获取本地接口
    public IInterface queryLocalInterface(String descriptor) {
        // 如果在同一进程，返回本地对象
        if (mDescriptor.equals(descriptor)) {
            return mOwner;
        }
        return null;
    }
}
```

### 3. BinderProxy（客户端）
```java
// 客户端代理
final class BinderProxy implements IBinder {
    // 发送事务到服务端
    public boolean transact(int code, Parcel data, Parcel reply, int flags)
            throws RemoteException {
        // 通过 Binder 驱动发送
        return transactNative(code, data, reply, flags);
    }
    
    // native 方法
    private native boolean transactNative(int code, Parcel data, Parcel reply, int flags)
            throws RemoteException;
}
```

### 4. Parcel
```java
// 数据容器，序列化和反序列化数据
public final class Parcel {
    // 写入基本类型
    public void writeInt(int val);
    public void writeLong(long val);
    public void writeFloat(float val);
    public void writeDouble(double val);
    public void writeString(String val);
    
    // 写入对象
    public void writeParcelable(Parcelable p, int parcelableFlags);
    public void writeStrongBinder(IBinder val);
    
    // 读取基本类型
    public int readInt();
    public long readLong();
    public float readFloat();
    public double readDouble();
    public String readString();
    
    // 读取对象
    public <T extends Parcelable> T readParcelable(ClassLoader loader);
    public IBinder readStrongBinder();
}
```

---

## 五、AIDL 生成代码分析

```java
public interface IMyService extends IInterface {
    
    // Binder 本地对象
    public static abstract class Stub extends Binder implements IMyService {
        
        private static final String DESCRIPTOR = "com.example.IMyService";
        
        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }
        
        // 将 IBinder 转换为 IMyService
        public static IMyService asInterface(IBinder obj) {
            if (obj == null) {
                return null;
            }
            
            // 如果在同一进程，直接返回本地对象
            IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (iin != null && iin instanceof IMyService) {
                return (IMyService) iin;
            }
            
            // 否则返回代理
            return new Proxy(obj);
        }
        
        @Override
        public IBinder asBinder() {
            return this;
        }
        
        // 处理客户端请求
        @Override
        public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
                throws RemoteException {
            switch (code) {
                case TRANSACTION_getData: {
                    data.enforceInterface(DESCRIPTOR);
                    int id = data.readInt();
                    String result = this.getData(id);
                    reply.writeNoException();
                    reply.writeString(result);
                    return true;
                }
                
                case TRANSACTION_getUser: {
                    data.enforceInterface(DESCRIPTOR);
                    int id = data.readInt();
                    User result = this.getUser(id);
                    reply.writeNoException();
                    reply.writeParcelable(result, 0);
                    return true;
                }
            }
            return super.onTransact(code, data, reply, flags);
        }
        
        // 客户端代理
        private static class Proxy implements IMyService {
            private IBinder mRemote;
            
            Proxy(IBinder remote) {
                mRemote = remote;
            }
            
            @Override
            public IBinder asBinder() {
                return mRemote;
            }
            
            @Override
            public String getData(int id) throws RemoteException {
                Parcel data = Parcel.obtain();
                Parcel reply = Parcel.obtain();
                
                try {
                    data.writeInterfaceToken(DESCRIPTOR);
                    data.writeInt(id);
                    
                    // 发送请求到服务端
                    mRemote.transact(TRANSACTION_getData, data, reply, 0);
                    reply.readException();
                    
                    return reply.readString();
                } finally {
                    data.recycle();
                    reply.recycle();
                }
            }
            
            @Override
            public User getUser(int id) throws RemoteException {
                Parcel data = Parcel.obtain();
                Parcel reply = Parcel.obtain();
                
                try {
                    data.writeInterfaceToken(DESCRIPTOR);
                    data.writeInt(id);
                    
                    mRemote.transact(TRANSACTION_getUser, data, reply, 0);
                    reply.readException();
                    
                    return reply.readParcelable(getClass().getClassLoader());
                } finally {
                    data.recycle();
                    reply.recycle();
                }
            }
        }
    }
    
    // 接口方法
    public String getData(int id) throws RemoteException;
    public User getUser(int id) throws RemoteException;
}
```

---

## 六、Binder 线程池

### 1. 线程池配置

```java
// 每个进程默认最大 16 个 Binder 线程
// 可以通过 ProcessState 修改
ProcessState::self()->setThreadPoolMaxThreadCount(15);
```

### 2. 线程池工作流程

```
1. 主线程调用 startThreadPool() 启动线程池
2. 线程池创建新线程等待请求
3. 收到请求后，分配空闲线程处理
4. 处理完成后，线程返回线程池
```

### 3. 同步与异步调用

```java
// 同步调用（默认）
mRemote.transact(code, data, reply, 0);
// 客户端阻塞等待服务端返回

// 异步调用（oneway）
mRemote.transact(code, data, null, IBinder.FLAG_ONEWAY);
// 客户端不等待返回
```

---

## 七、Binder 在 Android 中的应用

### 1. 系统服务

```java
// ActivityManagerService
IBinder b = ServiceManager.getService("activity");
IActivityManager am = IActivityManager.Stub.asInterface(b);

// WindowManagerService
IBinder b = ServiceManager.getService("window");
IWindowManager wm = IWindowManager.Stub.asInterface(b);

// PackageManagerService
IBinder b = ServiceManager.getService("package");
IPackageManager pm = IPackageManager.Stub.asInterface(b);
```

### 2. 四大组件通信

```java
// Activity 启动流程（跨进程）
// 1. Client 调用 startActivity
// 2. 通过 Binder 调用 AMS.startActivity
// 3. AMS 通过 Binder 回调 Client 的 ApplicationThread
// 4. Client 创建 Activity

// Service 绑定
// 1. Client 调用 bindService
// 2. 通过 Binder 调用 AMS.bindService
// 3. AMS 创建 Service，调用 onBind
// 4. 通过 Binder 返回 IBinder 给 Client

// ContentProvider
// 1. Client 调用 getContentResolver().query
// 2. 通过 Binder 调用 ContentProvider.query
// 3. ContentProvider 返回 Cursor

// BroadcastReceiver
// 1. Client 调用 sendBroadcast
// 2. 通过 Binder 调用 AMS.broadcastIntent
// 3. AMS 查找匹配的 Receiver
// 4. 通过 Binder 回调 Receiver.onReceive
```

### 3. 应用层

```java
// 自定义 Service
public class MyService extends Service {
    private final IMyService.Stub mBinder = new IMyService.Stub() {
        // 实现接口方法
    };
    
    @Override
    public IBinder onBind(Intent intent) {
        return mBinder;
    }
}

// ContentProvider
public class MyProvider extends ContentProvider {
    @Override
    public Cursor query(Uri uri, String[] projection, String selection,
            String[] selectionArgs, String sortOrder) {
        // 查询数据
    }
}
```

---

## 八、面试高频问题

### Q1: 为什么 Binder 只需一次拷贝？
- 使用 mmap 将内核空间和接收方用户空间映射到同一物理内存
- 发送方数据拷贝到内核空间后，接收方直接访问

### Q2: Binder 如何保证安全性？
- 通过 Binder 驱动获取调用方的 UID/PID
- 服务端可以验证调用方身份
- AndroidManifest.xml 中声明权限

### Q3: ServiceManager 是什么？
- 系统服务的注册中心
- 管理所有系统服务的 Binder 引用
- 通过 context_manager 进程实现

### Q4: Binder 线程池？
- 每个进程默认最大 16 个 Binder 线程
- 通过 Binder 线程池处理并发请求
- 主线程也可以处理 Binder 请求

### Q5: oneway 关键字？
```aidl
oneway void sendData(int data);
```
- 异步调用，不等待返回
- 适用于不需要返回值的场景
- 提高性能

### Q6: Binder 传输数据大小限制？
- 事务缓冲区约 1MB
- 传输大数据会导致 TransactionTooLargeException
- 解决方案：使用 ContentProvider 或共享内存

### Q7: 为什么不用 Socket？
- Socket 开销大，需要 2 次数据拷贝
- Socket 需要序列化和反序列化
- Binder 更高效，安全性更高

### Q8: Binder 死亡代理是什么？
```java
// 监听服务进程死亡
IBinder.DeathRecipient recipient = new IBinder.DeathRecipient() {
    @Override
    public void binderDied() {
        // 服务进程死亡，重新绑定
    }
};
binder.linkToDeath(recipient, 0);
```

---

## 九、最佳实践

### 1. 使用 AIDL
```java
// 定义清晰的接口
// 使用 Parcelable 传输对象
// 处理 RemoteException
```

### 2. 处理 RemoteException
```java
try {
    mService.getData(1);
} catch (RemoteException e) {
    // 跨进程调用可能失败
    e.printStackTrace();
}
```

### 3. 避免传输大数据
```java
// Binder 事务缓冲区约 1MB
// 传输大数据使用 ContentProvider 或文件
```

### 4. 使用 oneway
```aidl
// 不需要返回值时使用
oneway void sendData(int data);
```

### 5. 考虑线程安全
```java
// 服务端可能被多线程调用
// 使用 synchronized 或 ConcurrentHashMap
```

### 6. 使用 Messenger（简单场景）
```java
// 比 AIDL 更简单
public class MyService extends Service {
    private final Messenger mMessenger = new Messenger(new Handler() {
        @Override
        public void handleMessage(Message msg) {
            // 处理消息
        }
    });
    
    @Override
    public IBinder onBind(Intent intent) {
        return mMessenger.getBinder();
    }
}
```

---

## 十、Binder 与 ContentProvider

### 1. ContentProvider 的 Binder 实现

```java
public abstract class ContentProvider {
    // ContentProvider 底层使用 Binder
    @Override
    public IBinder onBind(Intent intent) {
        return new ContentProviderNative() {
            // 实现 IContentProvider 接口
        };
    }
}
```

### 2. ContentProvider 的使用

```java
// 查询
Cursor cursor = getContentResolver().query(
    Uri.parse("content://com.example.provider/users"),
    new String[]{"id", "name"},
    "id > ?",
    new String[]{"0"},
    "name ASC"
);

// 插入
ContentValues values = new ContentValues();
values.put("name", "张三");
Uri uri = getContentResolver().insert(
    Uri.parse("content://com.example.provider/users"),
    values
);

// 更新
ContentValues values = new ContentValues();
values.put("name", "李四");
int count = getContentResolver().update(
    Uri.parse("content://com.example.provider/users"),
    values,
    "id = ?",
    new String[]{"1"}
);

// 删除
int count = getContentResolver().delete(
    Uri.parse("content://com.example.provider/users"),
    "id = ?",
    new String[]{"1"}
);
```
