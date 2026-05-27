# Kotlin 协程详解

> Kotlin 协程是现代 Android 开发的核心，面试必问！

---

## 一、协程基础

### 1. 什么是协程？
- 轻量级线程
- 可以挂起和恢复
- 简化异步编程

### 2. 协程 vs 线程

| 特性 | 协程 | 线程 |
|------|------|------|
| 开销 | 极低（KB级别） | 较高（MB级别） |
| 切换 | 用户态，无需系统调用 | 内核态，需要系统调用 |
| 并发 | 可以轻松创建数万个 | 数量有限 |
| 取消 | 支持 | 需要手动处理 |
| 写法 | 顺序写法 | 回调或 RxJava |

### 3. 协程的核心概念

```kotlin
// 协程作用域
GlobalScope.launch {
    // 全局作用域，生命周期与应用一致（不推荐）
}

CoroutineScope(Dispatchers.Main).launch {
    // 自定义作用域
}

viewModelScope.launch {
    // ViewModel 作用域（推荐）
}

lifecycleScope.launch {
    // Lifecycle 作用域（推荐）
}

// 协程构建器
launch {
    // 启动协程，不返回结果
}

async {
    // 启动协程，返回结果
    return@async "result"
}

runBlocking {
    // 阻塞当前线程（不推荐在 Android 中使用）
}
```

---

## 二、挂起函数

### 1. 什么是挂起函数？
- 可以暂停执行，稍后恢复
- 不会阻塞线程
- 使用 `suspend` 关键字

### 2. 基本用法

```kotlin
// 定义挂起函数
suspend fun fetchData(): String {
    delay(1000) // 挂起 1 秒
    return "Data"
}

// 调用挂起函数
GlobalScope.launch {
    val data = fetchData() // 挂起等待
    println(data)
}
```kotlin

### 3. 原理分析

```kotlin
// 编译器转换
suspend fun fetchData(): String {
    delay(1000)
    return "Data"
}

// 转换后（简化）
fun fetchData(continuation: Continuation<String>): Any? {
    // Continuation 保存了协程的状态

    val label = (continuation as? FetchDataContinuationImpl)?.label ?: 0

    when (label) {
        0 -> {
            // 第一次调用
            (continuation as FetchDataContinuationImpl).label = 1
            val result = delay(1000, continuation)
            if (result == COROUTINE_SUSPENDED) {
                return COROUTINE_SUSPENDED // 挂起
            }
        }
        1 -> {
            // 恢复执行
            return "Data"
        }
    }

    return "Data"
}
```

**原理总结：**
- 编译器将挂起函数转换为状态机
- Continuation 保存协程状态
- 挂起时返回 COROUTINE_SUSPENDED
- 恢复时继续执行

---

## 三、协程上下文与调度器

### 1. 协程上下文 (CoroutineContext)

```kotlin
// 上下文元素
Job: 协程的生命周期
Dispatcher: 调度器，决定在哪个线程执行
CoroutineName: 协程名称
CoroutineExceptionHandler: 异常处理器

// 组合上下文
val context = Job() + Dispatchers.Main + CoroutineName("MyCoroutine")
```

### 2. 调度器 (Dispatcher)

```kotlin
Dispatchers.Main {
    // 主线程，用于 UI 操作
}

Dispatchers.IO {
    // IO 线程池，用于网络/磁盘操作
}

Dispatchers.Default {
    // 默认线程池，用于 CPU 密集型操作
}

Dispatchers.Unconfined {
    // 不限制线程，在当前线程启动
}
```kotlin

### 3. 切换线程

```kotlin
// 方法一：使用 withContext（推荐）
viewModelScope.launch {
    val data = withContext(Dispatchers.IO) {
        // 切换到 IO 线程
        repository.getData()
    }
    // 自动切换回主线程
    updateUI(data)
}

// 方法二：使用 launch
viewModelScope.launch(Dispatchers.Main) {
    val data = async(Dispatchers.IO) {
        repository.getData()
    }.await()
    updateUI(data)
}
```

---

## 四、协程构建器

### 1. launch

```kotlin
// 返回 Job，不返回结果
val job = GlobalScope.launch {
    delay(1000)
    println("Hello")
}

// 取消协程
job.cancel()

// 等待协程完成
job.join()

// 取消并等待
job.cancelAndJoin()
```kotlin

### 2. async

```kotlin
// 返回 Deferred，可以获取结果
val deferred = GlobalScope.async {
    delay(1000)
    "Result"
}

// 获取结果（挂起）
val result = deferred.await()

// 并发执行
val deferred1 = async { fetchUser() }
val deferred2 = async { fetchPosts() }
val user = deferred1.await()
val posts = deferred2.await()
```

### 3. runBlocking

```kotlin
// 阻塞当前线程
runBlocking {
    delay(1000)
    println("Hello")
}

// 主要用于测试和 main 函数
fun main() = runBlocking {
    launch {
        delay(1000)
        println("World")
    }
    println("Hello")
}
```kotlin

### 4. withContext

```kotlin
// 切换上下文执行
suspend fun fetchData(): String {
    return withContext(Dispatchers.IO) {
        // 在 IO 线程执行
        repository.getData()
    }
    // 返回主线程
}
```kotlin

---

## 五、协程异常处理

### 1. try-catch

```kotlin
viewModelScope.launch {
    try {
        val data = withContext(Dispatchers.IO) {
            repository.getData()
        }
        updateUI(data)
    } catch (e: Exception) {
        showError(e.message)
    }
}
```kotlin

### 2. CoroutineExceptionHandler

```kotlin
val handler = CoroutineExceptionHandler { _, exception ->
    println("Caught: $exception")
}

GlobalScope.launch(handler) {
    throw RuntimeException("Error")
}
```kotlin

### 3. SupervisorJob

```kotlin
// 子协程失败不影响其他子协程
val scope = CoroutineScope(SupervisorJob() + Dispatchers.Main)

scope.launch {
    // 子协程 1
    throw RuntimeException("Error")
}

scope.launch {
    // 子协程 2，不受影响
    println("Still running")
}
```kotlin

### 4. supervisorScope

```kotlin
viewModelScope.launch {
    supervisorScope {
        val job1 = launch {
            throw RuntimeException("Error")
        }

        val job2 = launch {
            // 不受影响
            println("Still running")
        }
    }
}
```kotlin

---

## 六、协程取消

### 1. 取消协程

```kotlin
val job = GlobalScope.launch {
    repeat(1000) { i ->
        println("I'm sleeping $i ...")
        delay(500)
    }
}

delay(1300)
println("main: I'm tired of waiting!")
job.cancelAndJoin() // 取消并等待
println("main: Now I can quit.")
```kotlin

### 2. 协作取消

```kotlin
// 检查取消状态
val job = GlobalScope.launch {
    repeat(1000) { i ->
        if (!isActive) return@launch // 检查是否已取消
        println("I'm sleeping $i ...")
        delay(500)
    }
}

// 使用 ensureActive()
val job = GlobalScope.launch {
    repeat(1000) { i ->
        ensureActive() // 如果已取消，抛出 CancellationException
        println("I'm sleeping $i ...")
        delay(500)
    }
}

// 使用 yield()
val job = GlobalScope.launch {
    repeat(1000) { i ->
        yield() // 检查取消并让出执行权
        println("I'm sleeping $i ...")
        delay(500)
    }
}
```kotlin

### 3. 使用 try-finally 释放资源

```kotlin
val job = GlobalScope.launch {
    try {
        repeat(1000) { i ->
            println("I'm sleeping $i ...")
            delay(500)
        }
    } finally {
        println("I'm running finally")
        // 释放资源
    }
}
```kotlin

### 4. 不可取消的代码块

```kotlin
val job = GlobalScope.launch {
    try {
        repeat(1000) { i ->
            println("I'm sleeping $i ...")
            delay(500)
        }
    } finally {
        withContext(NonCancellable) {
            // 不可取消的代码块
            println("I'm running finally")
            delay(1000) // 不会被取消
            println("I'm still running")
        }
    }
}
```

---

## 七、Flow

### 1. 什么是 Flow？
- 冷流，只有被收集时才执行
- 支持背压
- 可以发射多个值

### 2. 基本用法

```kotlin
// 创建 Flow
fun fetchNumbers(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100)
        emit(i) // 发射值
    }
}

// 收集 Flow
viewModelScope.launch {
    fetchNumbers()
        .map { it * 2 }
        .filter { it > 2 }
        .collect { value ->
            println(value)
        }
}
```

### 3. Flow 操作符

```kotlin
// 转换操作符
flow.map { it * 2 }
flow.filter { it > 0 }
flow.take(3)
flow.drop(2)
flow.flatMapConcat { ... } // 顺序执行
flow.flatMapMerge { ... }  // 并发执行
flow.flatMapLatest { ... } // 只保留最新

// 组合操作符
flow1.zip(flow2) { a, b -> a + b } // 一一对应
flow1.combine(flow2) { a, b -> a + b } // 任意变化

// 终端操作符
flow.collect { ... }
flow.toList()
flow.first()
flow.single()
flow.reduce { acc, value -> acc + value }
flow.fold(initial) { acc, value -> acc + value }
```

### 4. Flow 异常处理

```kotlin
// catch 操作符
flow.catch { e ->
    println("Caught: $e")
}
.collect { value ->
    println(value)
}

// onCompletion 操作符
flow.onCompletion { cause ->
    if (cause == null) {
        println("Flow completed successfully")
    } else {
        println("Flow completed with error: $cause")
    }
}
.collect { value ->
    println(value)
}
```kotlin

### 5. StateFlow 和 SharedFlow

```kotlin
// StateFlow（有初始值，只保留最新值）
private val _state = MutableStateFlow(initialState)
val state: StateFlow<State> = _state.asStateFlow()

// 更新状态
_state.value = newState

// 收集状态
lifecycleScope.launch {
    state.collect { state ->
        updateUI(state)
    }
}

// SharedFlow（无初始值，可以发射多个值）
private val _events = MutableSharedFlow<Event>()
val events: SharedFlow<Event> = _events.asSharedFlow()

// 发射事件
_events.emit(event)

// 收集事件
lifecycleScope.launch {
    events.collect { event ->
        handleEvent(event)
    }
}
```kotlin

### 6. StateFlow vs LiveData

| 特性 | StateFlow | LiveData |
|------|-----------|----------|
| 初始值 | 必须有 | 可选 |
| 生命周期感知 | 需要配合 | 自动 |
| 操作符 | 丰富 | 有限 |
| 多平台支持 | 支持 | 仅 Android |
| 线程切换 | 自动 | 自动 |

---

## 八、协程最佳实践

### 1. 使用 viewModelScope

```kotlin
class MyViewModel : ViewModel() {
    fun loadData() {
        viewModelScope.launch {
            val data = withContext(Dispatchers.IO) {
                repository.getData()
            }
            _uiState.value = data
        }
    }
}
```kotlin

### 2. 使用 lifecycleScope

```kotlin
class MyActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.state.collect { state ->
                    updateUI(state)
                }
            }
        }
    }
}
```kotlin

### 3. 异常处理

```kotlin
viewModelScope.launch {
    try {
        val data = withContext(Dispatchers.IO) {
            repository.getData()
        }
        _uiState.value = Success(data)
    } catch (e: Exception) {
        _uiState.value = Error(e.message)
    }
}
```kotlin

### 4. 并发请求

```kotlin
viewModelScope.launch {
    val userDeferred = async { repository.getUser() }
    val postsDeferred = async { repository.getPosts() }

    val user = userDeferred.await()
    val posts = postsDeferred.await()

    _uiState.value = Success(user, posts)
}
```kotlin

### 5. 取消协程

```kotlin
class MyViewModel : ViewModel() {
    private var job: Job? = null

    fun loadData() {
        job?.cancel()
        job = viewModelScope.launch {
            val data = withContext(Dispatchers.IO) {
                repository.getData()
            }
            _uiState.value = data
        }
    }

    override fun onCleared() {
        super.onCleared()
        job?.cancel()
    }
}
```

---

## 九、面试高频问题

### Q1: 协程和线程的区别？
- 协程是用户态的，线程是内核态的
- 协程开销小，可以创建数万个
- 协程支持挂起和恢复，不阻塞线程
- 协程可以顺序写法，避免回调地狱

### Q2: suspend 关键字的作用？
- 标记函数为挂起函数
- 只能在协程或其他挂起函数中调用
- 编译器转换为状态机

### Q3: Dispatchers.Main、IO、Default 的区别？
- Main：主线程，UI 操作
- IO：IO 线程池，网络/磁盘操作
- Default：默认线程池，CPU 密集型操作

### Q4: StateFlow 和 LiveData 的区别？
| 特性 | StateFlow | LiveData |
|------|-----------|----------|
| 初始值 | 必须有 | 可选 |
| 生命周期感知 | 需要配合 | 自动 |
| 操作符 | 丰富 | 有限 |
| 多平台支持 | 支持 | 仅 Android |

### Q5: 如何处理协程异常？
- try-catch
- CoroutineExceptionHandler
- SupervisorJob / supervisorScope

### Q6: Flow 和 LiveData 的区别？
| 特性 | Flow | LiveData |
|------|------|----------|
| 冷流/热流 | 冷流 | 热流 |
| 操作符 | 丰富 | 有限 |
| 生命周期感知 | 需要配合 | 自动 |
| 背压支持 | 支持 | 不支持 |

### Q7: 协程的取消原理？
- 协程定期检查取消状态
- delay()、yield() 等挂起函数会检查取消
- 可以手动检查 isActive 或调用 ensureActive()

### Q8: 为什么避免 GlobalScope？
- GlobalScope 的生命周期与应用一致
- 无法自动取消
- 可能导致内存泄漏

---

## 十、最佳实践

### 1. 使用 viewModelScope/lifecycleScope
```kotlin
// 自动取消
viewModelScope.launch {
    // 任务
}
```kotlin

### 2. 使用 withContext 切换线程
```kotlin
// 避免嵌套 launch
suspend fun fetchData(): Data {
    return withContext(Dispatchers.IO) {
        repository.getData()
    }
}
```kotlin

### 3. 使用 SupervisorJob
```kotlin
// 子协程失败不影响其他
val scope = CoroutineScope(SupervisorJob() + Dispatchers.Main)
```kotlin

### 4. 使用 StateFlow 替代 LiveData
```kotlin
// 更强大的操作符
private val _state = MutableStateFlow(initialState)
val state: StateFlow<State> = _state.asStateFlow()
```kotlin

### 5. 处理异常
```kotlin
// try-catch
viewModelScope.launch {
    try {
        // 任务
    } catch (e: Exception) {
        // 处理异常
    }
}
```kotlin

### 6. 避免 GlobalScope
```kotlin
// 错误
GlobalScope.launch {
    // 任务
}

// 正确
viewModelScope.launch {
    // 任务
}
```
