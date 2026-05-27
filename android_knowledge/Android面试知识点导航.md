# Android 面试知识点导航

> 系统化的 Android 面试必备知识点，涵盖从基础到高级的所有内容

---

## 📚 知识体系

### 1. Java/Kotlin 基础
- [[HashMap 原理与源码分析]]
- [[并发编程基础 - volatile synchronized 线程池]]
- [[JVM 垃圾回收机制]]
- [[动态代理原理]]
- [[TCP三次握手四次挥手]]

### 2. Android 四大组件
- [[Activity 生命周期与启动模式]]
- [[Service 详解]]
- [[BroadcastReceiver 详解]]
- [[ContentProvider 详解]]

### 3. Handler 机制
- [[Handler 机制原理详解]]
- [[同步屏障与异步消息]]
- [[Handler 内存泄漏分析]]

### 4. View 体系
- [[View 绘制流程 - measure layout draw]]
- [[事件分发机制详解]]
- [[自定义 View 实战]]
- [[RecyclerView 缓存机制]]

### 5. Framework 层
- [[Binder 跨进程通信原理]]
- [[AMS 与 ATMS]]
- [[Activity 启动流程]]
- [[APP Build 过程]]

### 6. 性能优化
- [[内存优化 - OOM 与内存泄漏]]
- [[启动优化]]
- [[页面渲染优化]]
- [[APK 瘦身]]
- [[线上 Bug 监控]]

### 7. 三方库源码
- [[Retrofit 原理分析]]
- [[OkHttp 原理分析]]
- [[Glide 图片加载原理]]
- [[Jetpack 组件原理]]

### 8. Kotlin 特性
- [[Kotlin 协程详解]]
- [[Kotlin 与 Java 的区别]]
- [[Kotlin 高级特性]]

### 9. 架构模式
- [[MVC MVP MVVM 架构对比]]
- [[组件化与模块化]]
- [[ARouter 原理分析]]

---

## 🎯 面试高频问题

### 基础题必问
1. HashMap 底层原理？扩容机制？
2. volatile 和 synchronized 的区别？
3. Handler 的工作原理？如何解决内存泄漏？
4. View 的绘制流程？
5. 事件分发机制？
6. Binder 跨进程通信原理？
7. Activity 的启动流程？
8. 内存泄漏有哪些场景？如何检测？
9. JVM 垃圾回收算法？
10. Kotlin 协程原理？

### 算法高频题
1. 两数之和
2. 第 K 个最大值
3. 遍历二叉树（前中后序）
4. 赛马问题
5. 链表反转

---

## 📖 学习路线

```text
Java基础 → Android基础 → Framework源码 → 性能优化 → 架构设计
    ↓           ↓              ↓            ↓          ↓
  集合框架    四大组件        Binder       内存优化    MVVM
  并发编程    UI体系         Handler      启动优化    组件化
  JVM原理    动画系统        WMS          卡顿优化    模块化
```

---

## 🔗 参考资源

- [Android 官方文档](https://developer.android.com)
- [AndroidX 源码](https://cs.android.com)
- [美团技术博客](https://tech.meituan.com)
- [掘金 Android 专栏](https://juejin.cn/tag/Android)
