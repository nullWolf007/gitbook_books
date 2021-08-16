[TOC]

# 使用 WorkManager 进行集成测试

* [WorkManager](https://developer.android.google.cn/topic/libraries/architecture/workmanager) 提供了一个 `work-testing` 工件，可以协助进行工作器测试。

## 一、设置

* 如需使用 `work-testing` 工件，请将它作为 `androidTestImplementation` 依赖项添加到 `build.gradle` 中。

```groovy
dependencies {
    def work_version = "2.5.0"

    ...

    // optional - Test helpers
    androidTestImplementation "androidx.work:work-testing:$work_version"
}
```

* 如需详细了解如何添加依赖项，请参阅 [WorkManager 版本说明](https://developer.android.google.cn/jetpack/androidx/releases/work#declaring_dependencies)中的“声明依赖项”部分。

* **注意**：从 2.1.0 开始，WorkManager 提供了 [`TestWorkerBuilder`](https://developer.android.google.cn/reference/androidx/work/testing/TestWorkerBuilder) 和 [`TestListenableWorkerBuilder`](https://developer.android.google.cn/reference/androidx/work/testing/TestListenableWorkerBuilder) 类，让您无需通过 `WorkManagerTestInitHelper` 初始化 `WorkManager` 也能测试工作器中的业务逻辑。[测试工作器实现](https://developer.android.google.cn/topic/libraries/architecture/workmanager/how-to/testing-worker-impl)中介绍了这些类。当您需要执行集成测试时，本页中的资料仍然有用。

* **注意**：强烈建议使用 [`TestListenableWorkerBuilder`](https://developer.android.google.cn/reference/androidx/work/testing/TestListenableWorkerBuilder) 测试 [`CoroutineWorker`](https://developer.android.google.cn/reference/kotlin/androidx/work/CoroutineWorker) 实现，因为 `work-testing` 工件使用 [Dispatchers.Default](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-default.html)，而不是工作器实现的 [CoroutineDispatcher](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-dispatcher/)。如需详细了解此 API，请参阅[测试工作器实现](https://developer.android.google.cn/topic/libraries/architecture/workmanager/how-to/testing-worker-impl#testing_listenableworker_and_its_variants)。

## 二、概念

* `work-testing` 为测试模式提供了一种特殊的 WorkManager 实现，它通过使用 [`WorkManagerTestInitHelper`](https://developer.android.google.cn/reference/androidx/work/testing/WorkManagerTestInitHelper) 来初始化。

* `work-testing` 工件还提供了 [`SynchronousExecutor`](https://developer.android.google.cn/reference/androidx/work/testing/SynchronousExecutor)，让您可以更轻松地以同步方式编写测试，而无需处理多个线程、锁定或锁存。

* 以下示例展示了如何将所有这些类一起使用。

```kotlin
@RunWith(AndroidJUnit4::class)
class BasicInstrumentationTest {
    @Before
    fun setup() {
        val context = InstrumentationRegistry.getTargetContext()
        val config = Configuration.Builder()
            .setMinimumLoggingLevel(Log.DEBUG)
            .setExecutor(SynchronousExecutor())
            .build()

        // Initialize WorkManager for instrumentation tests.
        WorkManagerTestInitHelper.initializeTestWorkManager(context, config)
    }
}
```

## 三、构造测试

* 现在 WorkManager 已在测试模式中初始化，您可以测试您的工作器了。

* 假设您有一个 `EchoWorker`，它需要一些 `inputData` 并简单地将其输入复制（回显）到 `outputData`。

```kotlin
class EchoWorker(context: Context, parameters: WorkerParameters)
   : Worker(context, parameters) {
   override fun doWork(): Result {
       return when(inputData.size()) {
           0 -> Result.failure()
           else -> Result.success(inputData)
       }
   }
}
```

### 3.1 基本测试

* 以下是一个对 `EchoWorker` 进行测试的 Android 插桩测试。这里的要点是，在测试模式中测试 `EchoWorker` 与在真实应用中使用 `EchoWorker` 非常相似。

```kotlin
@Test
@Throws(Exception::class)
fun testSimpleEchoWorker() {
    // Define input data
    val input = workDataOf(KEY_1 to 1, KEY_2 to 2)

    // Create request
    val request = OneTimeWorkRequestBuilder<EchoWorker>()
        .setInputData(input)
        .build()

    val workManager = WorkManager.getInstance(applicationContext)
    // Enqueue and wait for result. This also runs the Worker synchronously
    // because we are using a SynchronousExecutor.
    workManager.enqueue(request).result.get()
    // Get WorkInfo and outputData
    val workInfo = workManager.getWorkInfoById(request.id).get()
    val outputData = workInfo.outputData

    // Assert
    assertThat(workInfo.state, `is`(WorkInfo.State.SUCCEEDED))
    assertThat(outputData, `is`(input))
}
```

* 我们来编写另一个测试，它将确保在 `EchoWorker` 没有获得任何输入数据时，其预期的 `Result` 为 `Result.failure()`。

```kotlin
@Test
@Throws(Exception::class)
fun testEchoWorkerNoInput() {
   // Create request
   val request = OneTimeWorkRequestBuilder<EchoWorker>()
       .build()

   val workManager = WorkManager.getInstance(applicationContext)
   // Enqueue and wait for result. This also runs the Worker synchronously
   // because we are using a SynchronousExecutor.
   workManager.enqueue(request).result.get()
   // Get WorkInfo
   val workInfo = workManager.getWorkInfoById(request.id).get()

   // Assert
   assertThat(workInfo.state, `is`(WorkInfo.State.FAILED))
}
```

## 四、模拟约束、延迟和定期工作

* `WorkManagerTestInitHelper` 为您提供了 [`TestDriver`](https://developer.android.google.cn/reference/androidx/work/testing/TestDriver) 的一个实例，可用于模拟初始延迟、满足 `ListenableWorker` 实例约束的条件，以及 `PeriodicWorkRequest` 实例的间隔。

### 4.1 测试初始延迟

* 工作器可以具有初始延迟。如需测试具有 `initialDelay` 的 `EchoWorker`，而不必在测试中等待 `initialDelay`，您可以使用 `TestDriver` 通过 `setInitialDelayMet` 将工作请求的初始延迟标记为已满足条件。

```kotlin
@Test
@Throws(Exception::class)
fun testWithInitialDelay() {
    // Define input data
    val input = workDataOf(KEY_1 to 1, KEY_2 to 2)

    // Create request
    val request = OneTimeWorkRequestBuilder<EchoWorker>()
        .setInputData(input)
        .setInitialDelay(10, TimeUnit.SECONDS)
        .build()

    val workManager = WorkManager.getInstance(getApplicationContext())
    // Get the TestDriver
    val testDriver = WorkManagerTestInitHelper.getTestDriver()
    // Enqueue
    workManager.enqueue(request).result.get()
    // Tells the WorkManager test framework that initial delays are now met.
    testDriver.setInitialDelayMet(request.id)
    // Get WorkInfo and outputData
    val workInfo = workManager.getWorkInfoById(request.id).get()
    val outputData = workInfo.outputData

    // Assert
    assertThat(workInfo.state, `is`(WorkInfo.State.SUCCEEDED))
    assertThat(outputData, `is`(input))
}
```

### 4.2 测试约束

* `TestDriver` 也可用于利用 `setAllConstraintsMet` 将约束标记为已满足条件。以下示例展示了如何测试具有约束的 `Worker`。

```kotlin
@Test
@Throws(Exception::class)
fun testWithConstraints() {
    // Define input data
    val input = workDataOf(KEY_1 to 1, KEY_2 to 2)

    val constraints = Constraints.Builder()
        .setRequiredNetworkType(NetworkType.CONNECTED)
        .build()

    // Create request
    val request = OneTimeWorkRequestBuilder<EchoWorker>()
        .setInputData(input)
        .setConstraints(constraints)
        .build()

    val workManager = WorkManager.getInstance(myContext)
    val testDriver = WorkManagerTestInitHelper.getTestDriver()
    // Enqueue
    workManager.enqueue(request).result.get()
    // Tells the testing framework that all constraints are met.
    testDriver.setAllConstraintsMet(request.id)
    // Get WorkInfo and outputData
    val workInfo = workManager.getWorkInfoById(request.id).get()
    val outputData = workInfo.outputData

    // Assert
    assertThat(workInfo.state, `is`(WorkInfo.State.SUCCEEDED))
    assertThat(outputData, `is`(input))
}
```

### 4.3 测试定期工作

* `TestDriver` 也公开 `setPeriodDelayMet`，可用于指明某一时间间隔已经完成。以下是使用的 `setPeriodDelayMet` 的示例。

```kotlin
@Test
@Throws(Exception::class)
fun testPeriodicWork() {
    // Define input data
    val input = workDataOf(KEY_1 to 1, KEY_2 to 2)

    // Create request
    val request = PeriodicWorkRequestBuilder<EchoWorker>(15, MINUTES)
        .setInputData(input)
        .build()

    val workManager = WorkManager.getInstance(myContext)
    val testDriver = WorkManagerTestInitHelper.getTestDriver()
    // Enqueue and wait for result.
    workManager.enqueue(request).result.get()
    // Tells the testing framework the period delay is met
    testDriver.setPeriodDelayMet(request.id)
    // Get WorkInfo and outputData
    val workInfo = workManager.getWorkInfoById(request.id).get()

    // Assert
    assertThat(workInfo.state, `is`(WorkInfo.State.ENQUEUED))
}
```