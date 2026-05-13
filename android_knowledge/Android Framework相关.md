## 一、系统启动流程面试题解析

### 1. 你了解Android系统启动流程吗？

1. 启动电源以及系统启动

当电源按下时引导芯片从预定义的地方（固化在ROM）开始执行。加载引导程序BootLoader到RAM中，然后执行。

2. 引导程序BootLoader

引导程序BootLoader是在Android操作系统开始运行前的一个小程序，它的主要作用是将AndroidOS拉起来。

3. Linux内核(Kernel)启动

当内核启动时，设置缓存、被保护存储器、计划列表、加载驱动。当内核完成系统设置后，它首先会在系统文件中寻找init.rc文件，并启动init.rc进程。

4. init进程启动

init进程启动做了很多工作，但是总的来说主要就是做了一下三件事：

（1）创建和挂载启动所需文件目录。

（2）初始化和启动系统属性服务(其中系统服务进程包括 Zygote、service manager、media 等)。

（3）解析init.rc配置文件并启动Zygote进程。

5. Zygote进程启动

创建Java虚拟机并为Java虚拟机注册JNI方法；创建服务端Socket；预加载类和资源；启动SystemServer进程；等待AMS请求创建新的应用进程。

6. SystemServer进程启动

创建并启动Binder线程池，这样可以和其他进程进行通信。

创建SystemServiceManager，其用于对系统的服务进行创建、启动和生命周期的管理。

启动系统中的各种服务。包括我们熟悉的AMS、PMS、WMS。

7. Launcher启动

被SystemServer启动的AMS会启动Launcher，Launcher启动后会将已安装应用的图标显示在桌面上。

### 2. system_server为什么要在Zygote中启动，而不是由init直接启动呢？

Zygote 作为一个孵化器， 可以提前加载一些资源， 这样 fork() 时基于 Copy-On-Write 机制创建的其他进程就能 直接使用这些资源， 而不用重新加载。 比如 system_server 就可以直接使用 Zygote 中的 JNI 函数、共享库、常用 的类、 以及主题资源。

### 3.为什么要专门使用Zygote进程去孵化应用进程？而不是让system_server去孵化呢？

首先 system_server 相比 Zygote 多运行了 AMS、 WMS 等服务， 这些对一个应用程序来说是不需要的。 另外进 程 的 fork() 对多线程不友好， 仅会将发起调用的线程拷贝到子进程， 这可能会导致死锁， 而 system_server 中肯定 是有很多线程的。

### 4. 能说说具体是怎么导致死锁的吗？

在 POSIX 标准中， fork 的行为是这样的： 复制整个用户空间的数据 (通常使用 copy-on-write 的策略，所以可以实现的速度很快) 以4.及所有系统对象，然后仅复制当前线程到子进程。这里：所有父进程中别的线程，到了子进程 中都是突然蒸发掉的

对 于 锁来说， 从 OS 看， 每个锁有一个所有者， 即最后一次 lock 它的线程。假设这么一个环境， 在 fork 之前， 有 一个子线程 lock 了某个锁， 获得了对锁的所有权。fork  以后，在子进程中， 所有的额外线程都人间蒸发了。而锁却被正常复制了，在子进程看来，这个锁没有主人，所以没有任何人可以对它解锁。  当子进程想 lock 这个锁时，不再 有任何手段可以解开了。 程序发生死锁。

### 5.Zygote为什么不采用 Binder 机制进行 IPC 通信？

这个很重要的原因是如果zygote采用binder 会导致 fork出来的进程产生死锁。

在UNIX上有一个 程序设计的准则：多线程程序里不准使用fork。

这个规则的原因是：如果程序支持多线程，在程序进行fork的时候，就可能引起各种问题，最典型的问题就是，fork出来的子进程会复制父进程的所有内容，包括父进程的所有线程状态。那么父进程中如果有子线程正在处于等锁的状态的话，那么这个状态也会被复制到子进程中。父进程的中线程锁会有对应的线程来进行释放锁和解锁，但是子进程中的锁就等不到对应的线程来解锁了，所以为了避免这种子进程出现等锁的可能的风险，UNIX就有了不建议在多线程程序中使用fork的规则。

在Android系统中，Binder是支持多线程的，Binder线程池有可以有多个线程运行，那么binder 中就自然会有出现子线程处于等锁的状态。那么如果Zygote是使用的binder进程 IPC机制，那么Zygote中将有可能出现等锁的状态，此时，一旦通过zygote的fork去创建子进程，那么子进程将继承Zygote的等锁状态。这就会出现子进程一创建，天生的就在等待线程锁，而这个锁缺没有地方去帮它释放，子进程一直处于等待锁的状态。

## 二、Binder通信机制介绍

### 1. Binder通信模型

Binder的通信过程与网络请求类似，网络通信过程可以简化为4个角色：Client、Server、DNS服务器和路由器。一次完整的网络通信大体过程如下：

1、Client输入Server的域名

2、DNS解析域名：

通过域名是无法直接找到相应Server的，必须先通过DNS服务器将Server的域名转化为具体的IP地址。

3、通过路由器将请求发送至Server：

Client通过DNS服务器解析到Server的IP地址后，也还不能直接向Server发起请求，需要经过路由器的层层中转才还到达Server。

4、Server返回数据：

Server接收到请求并处理后，再通过路由器将数据返回给Client。

在Binder机制中，也定义了4个角色：Client、Server、Binder驱动和ServiceManager。

Binder驱动：类似网络通信中的路由器，负责将Client的请求转发到具体的Server中执行，并将Server返回的数据传回给Client。

ServiceManager：类似网络通信中的DNS服务器，负责将Client请求的Binder描述符转化为具体的Server地址，以便Binder驱动能够转发给具体的Server。Server如需提供Binder服务，需要向ServiceManager注册。
![](assets/Android%20Framework相关/file-20260513151527562.png)

具体的通信过程是这样的：

1、Server向ServiceManager注册：

Server通过Binder驱动向ServiceManager注册，声明可以对外提供服务。ServiceManager中会保留一份映射表：名字为zhangsan的Server对应的Binder引用是0x12345。/

2、Client向ServiceManager请求Server的Binder引用

Client想要请求Server的数据时，需要先通过Binder驱动向ServiceManager请求Server的Binder引用：我要向名字为zhangsan的Server通信，请告诉我Server的Binder引用。

3、向具体的Server发送请求

Client拿到这个Binder引用后，就可以通过Binder驱动和Server进行通信了。

4、Server返回结果

Server响应请求后，需要再次通过Binder驱动将结果返回给Client。

可以看到，Client、Server、ServiceManager之间的通信都是通过Binder驱动作为桥梁的，可见Binder驱动的重要性。也许你还有一点疑问，ServiceManager和Binder驱动属于两个不同的进程，它们是为Client和Server之间的进程间通信服务的，也就是说Client和Server之间的进程间通信依赖ServiceManager和Binder驱动之间的进程间通信，这就像是：“蛋生鸡，鸡生蛋，但第一个蛋得通过一只鸡孵出来”。Binder机制是如何创造第一只下蛋的鸡呢？

当Android系统启动后，会创建一个名称为servicemanager的进程，这个进程通过一个约定的命令BINDERSETCONTEXT_MGR向Binder驱动注册，申请成为为ServiceManager，Binder驱动会自动为ServiceManager创建一个Binder实体（第一只下蛋的鸡）；

并且这个Binder实体的引用在所有的Client中都为0，也就说各个Client通过这个0号引用就可以和ServiceManager进行通信。Server通过0号引用向ServiceManager进行注册，Client通过0号引用就可以获取到要通信的Server的Binder引用。
![](assets/Android%20Framework相关/file-20260513151600724.png)
### 2. 为什么使用Binder

- 性能 一次copy
- 稳定 c/s架构
- 安全 uid pid

Android Binder是用来做进程通信的，Android的各个应用以及系统服务都运行在独立的进程中，它们的通信都依赖于Binder。

为什么选用Binder，在讨论这个问题之前，我们知道Android也是基于Linux内核，Linux现有的进程通信手段有以下几种：

管道：在创建时分配一个page大小的内存，缓存区大小比较有限；

消息队列：信息复制两次，额外的CPU消耗；不合适频繁或信息量大的通信；

共享内存：无须复制，共享缓冲区直接付附加到进程虚拟地址空间，速度快；但进程间的同步问题操作系统无法实现，必须各进程利用同步工具解决；

套接字：作为更通用的接口，传输效率低，主要用于不通机器或跨网络的通信；

信号量：常作为一种锁机制，防止某进程正在访问共享资源时，其他进程也访问该资源。因此，主要作为进程间以及同一进程内不同线程之间的同步手段。

信号: 不适用于信息交换，更适用于进程中断控制，比如非法内存访问，杀死某个进程等；

既然有现有的IPC方式，为什么重新设计一套Binder机制呢。主要是出于以上三个方面的考量：

高性能：从数据拷贝次数来看Binder只需要进行一次内存拷贝，而管道、消息队列、Socket都需要两次，共享内存不需要拷贝，Binder的性能仅次于共享内存。

稳定性：上面说到共享内存的性能优于Binder，那为什么不适用共享内存呢，因为共享内存需要处理并发同步问题，控制负责，容易出现死锁和资源竞争，稳定性较差。而Binder基于C/S架构，客户端与服务端彼此独立，稳定性较好。

安全性：我们知道Android为每个应用分配了UID，用来作为鉴别进程的重要标志，Android内部也依赖这个UID进行权限管理，包括6.0以前的固定权限和6.0以后的动态权限，传统IPC只能由用户在数据包里填入UID/PID，这个标记完全是在用户空间控制的，没有放在内核空间，因此有被恶意篡改的可能，因此Binder的安全性更高。

### 3. Binder是如何做到一次拷贝的？

主要是因为Linux是使用的虚拟内存寻址方式，它有如下特性：

- 用户空间的虚拟内存地址是映射到物理内存中的
- 对虚拟内存的读写实际上是对物理内存的读写，这个过程就是内存映射
- 这个内存映射过程是通过系统调用mmap()来实现的
- Binder借助了内存映射的方法，在内核空间和接收方空间的数据缓存区之间做了一层内存映射，就相当于直接拷贝到了接收方用户空间的数据缓存区，从而减少了一次数据拷贝

### 4. MMAP的内存映射原理了解吗？

MMAP内存映射的实现过程，总的来说可以分为三个阶段：

一、进程启动映射过程，并在虚拟地址空间中位映射创建虚拟映射区域

1. 进程在用户空间调用库函数mmap，原型：void *mmap(void *start, size_t length, int prot, int flags, int fd, off_t offset);

2. 在当前进程的虚拟地址空间中，寻找一段空闲的满足要求的连续的虚拟地址

3. 为此虚拟区分配一个vm_area_struct结构，接着对这个结构的各个域进行了初始化

4. 将新建的虚拟区结构 (vm_area_struct) 插入进程的虚拟地址区域链表或树中

二、调用内核空间的系统调用函数mmap(不同于用户空间函数)，实现文件物理地址和进程虚拟地址的---映射关系

1. 为映射分配了新的虚拟地址区域后，通过待映射的文件指针，在文件描述符表中找到对应的文件描 述符，通过文件描述符，链接到内核“已打开文件集”中该文件的文件结构体 (struct file) ， 每个文件结构体维护着和这个已打开文件相关各项信息。

2. 通过该文件的文件结构体，链接到file_operations模块，调用内核函数mmap，其原型为： int mmap(struct file *filp, struct vm_area_struct *vma)，不同于用户空间库 函数。

3. 内核mmap函数通过虚拟文件系统inode模块定位到文件磁盘物理地址。

4. 通过remap_pfn_range函数建立页表，即实现了文件地址和虚拟地址区域的映射关系。此时，这 片虚拟地址并没有任何数据关联到主存中。

三、进程发起对这片映射空间的访问，引发缺页异常，实现文件内容到物理内存(主存)的拷贝

注：前两个阶段仅在于创建虚拟区间并完成地址映射，但是并没有将任何文件数据的拷贝至主存。 真正的文件读取是当进程发起读或写操作时。

进程的读或写操作访问虚拟地址空间这一段映射地址，通过查询页表，发现这一段地址并不在物理页面上。因为目前只建立了地址映射，真正的硬盘数据还没有拷贝到内存中，因此引发缺页异 常。

1. 缺页异常进行一系列判断，确定无非法操作后，内核发起请求调页过程。

2. 调页过程先在交换缓存空间 (swap cache) 中寻找需要访问的内存页，如果没有则调用nopage 函数把所缺的页从磁盘装入到主存中。

3. 之后进程即可对这片主存进行读或者写的操作，如果写操作改变了其内容，一定时间后系统会自动 回写脏页面到对应磁盘地址，也即完成了写入到文件的过程。

注：修改过的脏页面并不会立即更新回文件中，而是有一段时间的延迟，可以调用msync()来强 制同步 , 这样所写的内容就能立即保存到文件里了。

### 5. Binder机制是如何跨进程的

1. Binder驱动

- 在内核空间创建一块接收缓存区，
- 实现地址映射：将内核缓存区、接收进程用户空间映射到同一接收缓存区

2. 发送进程通过系统调用 (copy_from_user) 将数据发送到内核缓存区。由于内核缓存区和接收进程用户空间存在映射关系，故相当于也发送了接收进程的用户空间，实现了跨进程通信。

### 6. Zygote为什么不采用 Binder 机制进行 IPC 通信？

这个很重要的原因是如果zygote采用binder 会导致 fork出来的进程产生死锁。

在UNIX上有一个 程序设计的准则：多线程程序里不准使用fork。

这个规则的原因是：如果程序支持多线程，在程序进行fork的时候，就可能引起各种问题，最典型的问题就是，fork出来的子进程会复制父进程的所有内容，包括父进程的所有线程状态。那么父进程中如果有子线程正在处于等锁的状态的话，那么这个状态也会被复制到子进程中。父进程中的线程锁会有对应的线程来进行释放锁和解锁，但是子进程中的锁就等不到对应的线程来解锁了，所以为了避免这种子进程出现等锁的可能的风险，UNIX就有了不建议在多线程程序中使用fork的规则。

在Android系统中，Binder是支持多线程的，Binder线程池有可以有多个线程运行，那么binder 中就自然会有出现子线程处于等锁的状态。那么如果Zygote是使用的binder进程 IPC机制，那么Zygote中将有可能出现等锁的状态，此时，一旦通过zygote的fork去创建子进程，那么子进程将继承Zygote的等锁状态。这就会出现子进程一创建，天生的就在等待线程锁，而这个锁缺没有地方去帮它释放，子进程一直处于等待锁的状态。

### 7. 为什么Intent不能传递大数据？

Intent携带信息的大小其实是受Binder限制。数据以Parcel对象的形式存放在Binder传递缓存中。如果数据或返回值比传递buffer大， 则此次传递调用失败并抛出

TransactionTooLargeException异 常 。

Binder传递缓存有一个限定大小，通常是1Mb。但同一个进程中所有的传输共享缓存空间。多个地方在进行传输时，即使它们各自传输的数据不超出大小限制， TransactionTooLargeException异常也可能会被抛出 。在使用Intent传递数据时 ，1 Mb并不是安全上限。因为Binder中可能正在处理其它的传输工作。不同的机型和系统版本，这个上限值也可能会不同。

## 三、AMS面试题解析

### 1. ActivityManagerServices 是什么？什么时候初始化？有什么作用？

ActivityManagerService 主要负责系统中四大组件的启动、切换、调度及应用进程的管理和调度等工作 ， 其 职责与操作系统中的进程管理和调度模块类似。ActivityManagerService进行初始化的时机很明确 ，就是在 SystemServer进程开启的时候 ， 就会初始化ActivityManagerService。( 系统启动流程)如果打开一个App的话 ， 需要AMS去通知zygote进程 ，所有的Activity的生命周期AMS来控制。

### 2. ActivityThread是什么？ApplicationThread是什么？他们的区别？

ActivityThread

在 Android中它 就代表了 Android 的主 线程 ,它是创建完新进程之后 ,main函数被加载 ，然后执行一个 loop的循环 ，使当前线程进入消息循环 ，并且作为主线程。

ApplicationThread

ApplicationThread是 ActivityThread的 内部类 ， 是 一 个 Binder对 象 。 在此处它是作为 IApplicationThread 对 象的server端等待client端的请求然后进行处理 ，最大的client就是AMS。

### 3. Instrumenttation是什么？和ActivityThread是什么关系？

AMS与 ActivityThread之间 诸如Activity的创 建、暂停 等的交 互工作 实际上 是由 Instrumentation具体操作 的。 每个 Activity都 持 有 一 个 Instrumentation对 象 的 一 个 引 用  ，整 个 进 程 中 是 只 有 一 个 Instrumentation。

mInstrumentation的 初 始 化 在 ActivityThread: : handleBindApplication函 数 。

可以用来独立地控制某个组件的生命周期。

Activity`的`startActivity`方法。 `startActivity`会调用`mInstrumentation .execStartActivity() ; mInstrumentation 掉用 AMS , AMS 通过 socket 通信告知 Zygote 进程 fork 子进程。

### 4.ActivityManagerService和Zygote进程通信是如何实现的？

应用启动时,Launcher进程请求AMS。 AMS发送创建应用进程请求 ，Zygote进程接受请求并fork应用进程 客户端发送请求

调用 Process.start() 方法新建进程

连接调用的是 ZygoteState. connect() 方法 ， ZygoteState 是 ZygoteProcess 的内部类。 ZygoteState里用的 LocalSocket

```java
public static ZygoteState connect(LocalSocketAddress address) throws IOException {  
    DataInputStream zygoteInputStream = null;  
    BufferedWriter zygoteWriter = null;  
    final LocalSocket zygoteSocket = new LocalSocket();  
    try {  
        zygoteSocket.connect(address);  
        zygoteInputStream = new DataInputStream(zygoteSocket.getInputStream());  
        zygoteWriter = new BufferedWriter(new OutputStreamWriter(  
                zygoteSocket.getOutputStream()), 256);  
    } catch (IOException ex) {  
        try {  
            zygoteSocket.close();  
        } catch (IOException ignore) {  
        }        throw ex;  
    }  
    return new ZygoteState(zygoteSocket, zygoteInputStream, zygoteWriter, Arrays.asList(abiListString.split(",")));  
}
```

Zygote 处理客户端请求

Zygote 服务端接收到参数之后调用 ZygoteConnection. processOneCommand() 处理参数 ，并 fork 进程 最 后通过 findStaticMain() 找到 ActivityThread 类的 main() 方法并执行 ， 子进程就启动了

### 5. ActivityRecord、TaskRecord、ActivityStack、ActivityStackSupervisor、ProcessRecord

ActivityRecord

Activity管理的最小单位 ， 它对应着一个用户界面.

ActivityRecord是应用层Activity组件在AMS中的代表 ，每一个在应用中启动的Activity ，在AMS中都有一个ActivityRecord实例来与之对应 ，这个ActivityRecord伴随着Activity的启动而创建 ，也伴随着Activity的终止而销毁。

TaskRecord

TaskRecord即任务栈 ，每 一个TaskRecord都可能存在一个或多个ActivityRecord ， 栈顶的ActivityRecord表示当 前可见的界面。

 一个App是可能有多个TaskRecord存在的，一般情况下,启动App的第一个activity时 ， AMS为其创建一个TaskRecord任务栈。特殊情况, 启动singleTask的Activity ， 而且为该Activity指定了和包名不同的 taskAffinity ， 也会为该activity创建一个新的TaskRecord。

ActivityStack

ActivityStack, ActivityStack是 系 统 中 用 于 管 理 TaskRecord的 , 内 部 维 护 了 一 个 ArrayList。 ActivityStackSupervisor内 部有两个不同的ActivityStack对象 ： mHomeStack、 mFocusedStack ，用来管理 不同的 任务。我们启动的App对应的TaskRecord由非Launcher ActivityStack管理 ，它是在系统启动第一个 app时创建的。

ActivityStackSupervisor

ActivityStackSupervisor管 理着多个ActivityStack ， 但当前只会有一个获取焦点( Focused) 的 ActivityStack; AMS对象只会存在一个 ，在初始化的时候 ，会创建一个唯一的ActivityStackSupervisor对象。

ProcessRecord

ProcessRecord记录着属于一个进程的所有ActivityRecord ， 运行在不同TaskRecord中的 ActivityRecord可能 是属于同一个 ProcessRecord。

### 6.ActivityManager、ActivityManagerService、ActivityManagerNative、ActivityManagerProxy的关系

　IActivityManager作为ActivityManagerProxy和ActivityManagerNative的公共接口，

所以两个类具有部分相同的接口，可以实现合理的代理模式；

　　ActivityManagerProxy代理类是ActivityManagerNative的内部类；

ActivityManagerNative是个抽象类，真正发挥作用的是它的子类ActivityManagerService（系统Service组件）。

![0](https://note.youdao.com/yws/res/3197/WEBRESOURCEad7b34360e6d6f730a9bd7a2fe75c5ea)