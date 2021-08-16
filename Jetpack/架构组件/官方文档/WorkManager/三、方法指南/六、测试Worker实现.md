[TOC]

# 测试 Worker 实现

* WorkManager 提供了用于测试 [`Worker`](https://developer.android.google.cn/reference/kotlin/androidx/work/Worker)、[`ListenableWorker`](https://developer.android.google.cn/reference/androidx/work/ListenableWorker) 和 `ListenableWorker` 变体（[`CoroutineWorker`](https://developer.android.google.cn/reference/kotlin/androidx/work/CoroutineWorker) 和 [`RxWorker`](https://developer.android.google.cn/reference/androidx/work/RxWorker)）的 API。

* **注意**：在版本 2.1.0 以前，如需测试工作器，您需要使用 [`WorkManagerTestInitHelper`](https://developer.android.google.cn/reference/androidx/work/testing/WorkManagerTestInitHelper) 来初始化 WorkManager。从 2.1.0 开始，如果要测试 `Worker` 的实现，将无需使用 `WorkManagerTestInitHelper`。

## 一、测试工作器

* 假设我们有一个类似以下示例的 `Worker`：

```kotlin
class SleepWorker(context: Context, parameters: WorkerParameters) :
    Worker(context, parameters) {

    override fun doWork(): Result {
        // Sleep on a background thread.
        Thread.sleep(1000)
        return Result.success()
    }
}
```

* 如需测试这个 `Worker`，您可以使用 [`TestWorkerBuilder`](https://developer.android.google.cn/reference/androidx/work/testing/TestWorkerBuilder)。此构建器有助于构建可用于测试业务逻辑的 `Worker` 实例。

```kotlin
// Kotlin code uses the TestWorkerBuilder extension to build
// the Worker
@RunWith(AndroidJUnit4::class)
class SleepWorkerTest {
    private lateinit var context: Context
    private lateinit var executor: Executor

    @Before
    fun setUp() {
        context = ApplicationProvider.getApplicationContext()
        executor = Executors.newSingleThreadExecutor()
    }

    @Test
    fun testSleepWorker() {
        val worker = TestWorkerBuilder<SleepWorker>(
            context = context,
            executor = executor
        ).build()

        val result = worker.doWork()
        assertThat(result, `is`(Result.success()))
    }
}
```

* `TestWorkerBuilder` 也可用于设置标记（例如 `inputData` 或 `runAttemptCount`），以便您单独验证工作器状态。例如，`SleepWorker` 将休眠时长当作输入数据，而不是工作器中定义的常量数据：

```kotlin
class SleepWorker(context: Context, parameters: WorkerParameters) :
    Worker(context, parameters) {

    override fun doWork(): Result {
        // Sleep on a background thread.
        val sleepDuration = inputData.getLong(SLEEP_DURATION, 1000)
        Thread.sleep(sleepDuration)
        return Result.success()
    }

    companion object {
        const val SLEEP_DURATION = "SLEEP_DURATION"
    }
}
```

* 在 `SleepWorkerTest` 中，您可以将该输入数据提供给 `TestWorkerBuilder`，以满足 `SleepWorker` 的需求。

```kotlin
// Kotlin code uses the TestWorkerBuilder extension to build
// the Worker
@RunWith(AndroidJUnit4::class)
class SleepWorkerTest {
    private lateinit var context: Context
    private lateinit var executor: Executor

    @Before
    fun setUp() {
        context = ApplicationProvider.getApplicationContext()
        executor = Executors.newSingleThreadExecutor()
    }

    @Test
    fun testSleepWorker() {
        val worker = TestWorkerBuilder<SleepWorker>(
            context = context,
            executor = executor,
            inputData = workDataOf("SLEEP_DURATION" to 1000L)
        ).build()

        val result = worker.doWork()
        assertThat(result, `is`(Result.success()))
    }
}
```

* 如需详细了解 `TestWorkerBuilder` API，请参阅 [`TestListenableWorkerBuilder`](https://developer.android.google.cn/reference/androidx/work/testing/TestListenableWorkerBuilder)（`TestWorkerBuilder` 的父类）的参考页面。

## 二、测试 ListenableWorker 及其变体

* 如需测试 [`ListenableWorker`](https://developer.android.google.cn/reference/androidx/work/ListenableWorker) 或其变体（[`CoroutineWorker`](https://developer.android.google.cn/reference/kotlin/androidx/work/CoroutineWorker) 和 [`RxWorker`](https://developer.android.google.cn/reference/androidx/work/RxWorker)），请使用 [`TestListenableWorkerBuilder`](https://developer.android.google.cn/reference/androidx/work/testing/TestListenableWorkerBuilder)。`TestWorkerBuilder` 和 [`TestListenableWorkerBuilder`](https://developer.android.google.cn/reference/androidx/work/testing/TestListenableWorkerBuilder) 的主要区别在于，`TestWorkerBuilder` 允许指定用来运行 `Worker` 的后台 `Executor`，而 `TestListenableWorkerBuilder` 依赖 `ListenableWorker` 实现的线程逻辑。

* 例如，假设我们需要测试类似以下示例的 `CoroutineWorker`：

```kotlin
class SleepWorker(context: Context, parameters: WorkerParameters) :
    CoroutineWorker(context, parameters) {
    override suspend fun doWork(): Result {
        delay(1000L) // milliseconds
        return Result.success()
    }
}
```

* 为了测试 `SleepWorker`，我们首先使用 `TestListenableWorkerBuilder` 创建一个工作器实例，然后在协程中调用其 `doWork` 函数。

```kotlin
@RunWith(AndroidJUnit4::class)
class SleepWorkerTest {
    private lateinit var context: Context

    @Before
    fun setUp() {
        context = ApplicationProvider.getApplicationContext()
    }

    @Test
    fun testSleepWorker() {
        val worker = TestListenableWorkerBuilder<SleepWorker>(context).build()
        runBlocking {
            val result = worker.doWork()
            assertThat(result, `is`(Result.success()))
        }
    }
}
```

* `runBlocking` 可用作测试的协程构建器，使任何异步执行的代码都转为并行运行。

* 测试 `RxWorker` 实现类似于测试 `CoroutineWorker`，因为 `TestListenableWorkerBuilder` 可以处理 `ListenableWorker` 的任何子类。假设某个版本的 `SleepWorker` 使用 RxJava 而非协程。

```kotlin
class SleepWorker(
    context: Context,
    parameters: WorkerParameters
) : RxWorker(context, parameters) {
    override fun createWork(): Single<Result> {
        return Single.just(Result.success())
            .delay(1000L, TimeUnit.MILLISECONDS)
    }
}
```

* 测试 `RxWorker` 的 `SleepWorkerTest` 版本可能类似于测试 `CoroutineWorker` 的版本。您使用相同的 `TestListenableWorkerBuilder`，但现在会调用 `RxWorker` 的 `createWork` 函数。`createWork` 会返回一个 `Single`，可用于验证工作器的行为。`TestListenableWorkerBuilder` 可处理线程方面的任何复杂问题，还能并行执行工作器代码。

```kotlin
@RunWith(AndroidJUnit4::class)
class SleepWorkerTest {
    private lateinit var context: Context

    @Before
    fun setUp() {
        context = ApplicationProvider.getApplicationContext()
    }

    @Test
    fun testSleepWorker() {
        val worker = TestListenableWorkerBuilder<SleepWorker>(context).build()
        worker.createWork().subscribe { result ->
            assertThat(result, `is`(Result.success()))
        }
    }
}
```