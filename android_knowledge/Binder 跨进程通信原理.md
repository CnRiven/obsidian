# Binder 跨进程通信原理

> Binder 是 Android 最核心的 IPC 机制，面试必问！

---

## 一、为什么使用 Binder？

### 1. 传统 IPC 方式对比

| 方式 | 数据拷贝次数 | 特点 |
|------|-------------|------|
| 共享内存 | 0 | 控制复杂 |
| 管道/消息队列 | 2 | 效率低 |
| Socket | 2 | 开销大 |
| Binder | 1 | Android 特有 |

### 2. Binder 的优势
- **性能好**：只需一次数据拷贝
- **安全性高**：通过 UID/PID 验证身份
- **使用简单**：AIDL 接口定义

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
│       │    Binder 驱动     │                    │           │
│       │         │          │                    │           │
│       └─────────┼──────────┘                    │           │
│                 │                               │           │
│                 │         ┌─────────┐           │           │
│                 └─────────│ Binder  │───────────┘           │
│                           │ Driver  │                       │
│                           └─────────┘                       │
│                              │                              │
│                     ┌────────┴────────┐                     │
│                     │   内核空间       │                     │
│                     └─────────────────┘                     │
└─────────────────────────────────────────────────────────────┘
```

### 2. 一次拷贝原理

```
传统 IPC：
用户空间 A → 内核空间 → 用户空间 B（2次拷贝）

Binder：
用户空间 A → 内核空间（mmap 映射）→ 用户空间 B（1次拷贝）
```

**mmap 原理：**
- 将内核空间和接收方用户空间映射到同一块物理内存
- 发送方数据拷贝到内核空间后，接收方直接访问

---

## 三、AIDL 使用

### 1. 定义 AIDL 接口

```aidl
// IMyService.aidl
package com.example;

interface IMyService {
    String getData(int id);
    void setData(int id, String data);
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
            mService = IMyService.Stub.asInterface(service);
            mBound = true;
        }
        
        @Override
        public void onServiceDisconnected(ComponentName name) {
            mService = null;
            mBound = false;
        }
    };
    
    @Override
    protected void onStart() {
        super.onStart();
        Intent intent = new Intent(this, MyService.class);
        bindService(intent, mConnection, Context.BIND_AUTO_CREATE);
    }
    
    @Override
    protected void onStop() {
        super.onStop();
        if (mBound) {
            unbindService(mConnection);
            mBound = false;
        }
    }
    
    private void callService() {
        if (mBound) {
            try {
                String data = mService.getData(1);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }
    }
}
```

---

## 四、Binder 核心类

### 1. IBinder
- Binder 通信的基本接口
- 定义了 transact() 方法

### 2. Binder
- 服务端基类
- 实现 IBinder 接口
- onTransact() 处理客户端请求

### 3. BinderProxy
- 客户端代理
- transact() 发送请求到服务端

### 4. Parcel
- 数据容器
- 序列化和反序列化数据

---

## 五、Binder 通信流程

### 1. 注册服务

```
Server 进程
    ↓
ServiceManager.addService("service_name", binder)
    ↓
Binder 驱动
    ↓
ServiceManager（存储服务引用）
```

### 2. 获取服务

```
Client 进程
    ↓
ServiceManager.getService("service_name")
    ↓
Binder 驱动
    ↓
返回 BinderProxy 对象
```

### 3. 调用服务

```
Client: proxy.transact(code, data, reply, flags)
    ↓
Binder 驱动
    ↓
Server: binder.onTransact(code, data, reply, flags)
    ↓
处理请求，写入 reply
    ↓
Binder 驱动
    ↓
Client: 读取 reply
```

---

## 六、AIDL 生成代码分析

```java
public interface IMyService extends IInterface {
    
    // Stub 是服务端基类
    public static abstract class Stub extends Binder implements IMyService {
        
        private static final String DESCRIPTOR = "com.example.IMyService";
        
        // 构造函数，注册接口
        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }
        
        // 将 IBinder 转换为 IMyService
        public static IMyService asInterface(IBinder obj) {
            if (obj == null) {
                return null;
            }
            
            // 如果在同一进程，直接返回
            IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (iin != null && iin instanceof IMyService) {
                return (IMyService) iin;
            }
            
            // 否则返回代理
            return new Proxy(obj);
        }
        
        // 处理客户端请求
        @Override
        public boolean onTransact(int code, Parcel data, Parcel reply, int flags) {
            switch (code) {
                case TRANSACTION_getData: {
                    data.enforceInterface(DESCRIPTOR);
                    int id = data.readInt();
                    String result = this.getData(id);
                    reply.writeNoException();
                    reply.writeString(result);
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
            public String getData(int id) throws RemoteException {
                Parcel data = Parcel.obtain();
                Parcel reply = Parcel.obtain();
                
                try {
                    data.writeInterfaceToken(DESCRIPTOR);
                    data.writeInt(id);
                    
                    mRemote.transact(TRANSACTION_getData, data, reply, 0);
                    reply.readException();
                    
                    return reply.readString();
                } finally {
                    data.recycle();
                    reply.recycle();
                }
            }
        }
    }
}
```

---

## 七、面试高频问题

### Q1: 为什么 Binder 只需一次拷贝？
- 使用 mmap 将内核空间和接收方用户空间映射到同一物理内存
- 发送方数据拷贝到内核空间后，接收方直接访问

### Q2: Binder 如何保证安全性？
- 通过 Binder 驱动获取调用方的 UID/PID
- 服务端可以验证调用方身份
- AndroidManifest.xml 中声明权限

### Q3: 为什么不用共享内存？
- 共享内存控制复杂，需要同步机制
- Binder 提供了封装好的接口，使用简单
- Binder 有安全校验机制

### Q4: ServiceManager 是什么？
- 系统服务的注册中心
- 管理所有系统服务的 Binder 引用
- 通过 context_manager 进程实现

### Q5: Binder 线程池？
- 每个进程默认最大 16 个 Binder 线程
- 通过 Binder 线程池处理并发请求
- 主线程也可以处理 Binder 请求

### Q6: oneway 关键字？
```aidl
oneway void sendData(int data);
```
- 异步调用，不等待返回
- 适用于不需要返回值的场景
- 提高性能

---

## 八、最佳实践

1. **使用 AIDL**：定义清晰的接口
2. **处理 RemoteException**：跨进程调用可能失败
3. **避免传输大数据**：Binder 事务缓冲区约 1MB
4. **使用 oneway**：不需要返回值时
5. **考虑线程安全**：服务端可能被多线程调用
6. **使用 Messenger**：简单场景下比 AIDL 更简单

---

## 九、Binder 在 Android 中的应用

### 1. 系统服务
- ActivityManagerService
- WindowManagerService
- PackageManagerService

### 2. 四大组件通信
- Activity 的启动流程
- Service 的绑定
- BroadcastReceiver 的注册
- ContentProvider 的访问

### 3. 应用层
- 自定义 Service
- ContentProvider
- AIDL 接口
