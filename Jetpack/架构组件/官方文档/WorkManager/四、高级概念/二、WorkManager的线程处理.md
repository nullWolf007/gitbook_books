[TOC]

# WorkManager 中的线程处理

## 一、概览

* 在 [WorkManager 使用入门](https://developer.android.google.cn/topic/libraries/architecture/workmanager/basics)中，我们提到 WorkManager 可以代表您异步执行后台工作。该基本实现可满足大多数应用的需求。关于更高级的用例（例如正确处理正在停止的工作），您应了解 WorkManager 中的线程处理和并发机制。

* WorkManager 提供了四种不同类型的工作基元：
  * [`Worker`](https://developer.android.google.cn/reference/androidx/work/Worker) 是最简单的实现，我们已在前面几节进行了介绍。WorkManager 会在后台线程中自动运行该基元（您可以将它替换掉）。请参阅[工作器中的线程处理](https://developer.android.google.cn/topic/libraries/architecture/workmanager/advanced/worker)，详细了解 `Worker` 实例中的线程处理。
  * [`CoroutineWorker`](https://developer.android.google.cn/reference/kotlin/androidx/work/CoroutineWorker) 是为 Kotlin 用户建议的实现。`CoroutineWorker` 实例公开了后台工作的一个挂起函数。默认情况下，这些实例运行默认的 `Dispatcher`，但您可以进行自定义。请参阅 [CoroutineWorker 中的线程处理](https://developer.android.google.cn/topic/libraries/architecture/workmanager/advanced/coroutineworker)，详细了解 `CoroutineWorker` 实例中的线程处理。
  * [`RxWorker`](https://developer.android.google.cn/reference/androidx/work/RxWorker) 是为 RxJava 用户建议的实现。如果您有很多现有异步代码是用 RxJava 建模的，则应使用 RxWorker。与所有 RxJava 概念一样，您可以自由选择所需的线程处理策略。请参阅 [RxWorker 中的线程处理](https://developer.android.google.cn/topic/libraries/architecture/workmanager/advanced/rxworker)，详细了解 `RxWorker` 实例中的线程处理。
  * [`ListenableWorker`](https://developer.android.google.cn/reference/androidx/work/ListenableWorker) 是 `Worker`、`CoroutineWorker` 和 `RxWorker` 的基类。这个类专为需要与基于回调的异步 API（例如 `FusedLocationProviderClient`）进行交互并且不使用 RxJava 的 Java 开发者而设计。请参阅 [ListenableWorker 中的线程处理](https://developer.android.google.cn/topic/libraries/architecture/workmanager/advanced/listenableworker)，详细了解 `ListenableWorker` 实例中的线程处理。

## 二、用Worker处理线程

* 当您使用 [`Worker`](https://developer.android.google.cn/reference/androidx/work/Worker) 时，WorkManager 会自动在后台线程中调用 [`Worker.doWork()`](https://developer.android.google.cn/reference/androidx/work/Worker#doWork())。该后台线程来自于 WorkManager 的 [`Configuration`](https://developer.android.google.cn/reference/androidx/work/Configuration) 中指定的 `Executor`。默认情况下，WorkManager 会为您设置 `Executor`，但您也可以自己进行自定义。例如，您可以在应用中共享现有的后台 Executor，也可以创建单线程 `Executor` 以确保所有后台工作都按顺序执行，甚至可以指定一个自定义 `Executor`。如需自定义 `Executor`，请确保手动初始化 WorkManager。

* 在手动配置 WorkManager 时，您可以按以下方式指定 `Executor`：

```kotlin
WorkManager.initialize(
    context,
    Configuration.Builder()
         // Uses a fixed thread pool of size 8 threads.
        .setExecutor(Executors.newFixedThreadPool(8))
        .build())
```

* 下面是一个简单的 `Worker` 示例，其会下载网页内容 100 次：

```kotlin
class DownloadWorker(context: Context, params: WorkerParameters) : Worker(context, params) {

    override fun doWork(): ListenableWorker.Result {
        repeat(100) {
            try {
                downloadSynchronously("https://www.google.com")
            } catch (e: IOException) {
                return ListenableWorker.Result.failure()
            }
        }

        return ListenableWorker.Result.success()
    }
}
```

* 请注意，[`Worker.doWork()`](https://developer.android.google.cn/reference/androidx/work/Worker#doWork()) 是同步调用 - 您应以阻塞方式完成整个后台工作，并在方法退出时完成工作。如果您在 `doWork()` 中调用异步 API 并返回 [`Result`](https://developer.android.google.cn/reference/androidx/work/ListenableWorker.Result)，那么回调可能无法正常运行。如果您遇到这种情况，请考虑使用 [`ListenableWorker`](https://developer.android.google.cn/reference/androidx/work/ListenableWorker)（请参阅[在 ListenableWorker 中进行线程处理](https://developer.android.google.cn/topic/libraries/architecture/workmanager/advanced/listenableworker)）。

* 如果当前正在运行的 `Worker` [因任何原因而停止](https://developer.android.google.cn/topic/libraries/architecture/workmanager/how-to/managing-work#cancelling)，它就会收到对 [`Worker.onStopped()`](https://developer.android.google.cn/reference/androidx/work/ListenableWorker#onStopped()) 的调用。在必要的情况下，只需替换此方法或调用 [`Worker.isStopped()`](https://developer.android.google.cn/reference/androidx/work/ListenableWorker#isStopped())，即可对代码进行检查点处理并释放资源。当上述示例中的 `Worker` 被停止时，内容的下载可能才完成了一半；但即使该工作器被停止，下载也会继续。如需优化此行为，您可以执行以下操作：

```kotlin
class DownloadWorker(context: Context, params: WorkerParameters) : Worker(context, params) {

    override fun doWork(): ListenableWorker.Result {
        repeat(100) {
            if (isStopped) {
                break
            }

            try {
                downloadSynchronously("https://www.google.com")
            } catch (e: IOException) {
                return ListenableWorker.Result.failure()
            }

        }

        return ListenableWorker.Result.success()
    }
}
```

* `Worker` 停止后，从 `Worker.doWork()` 返回什么已不重要；`Result` 将被忽略。

## 三、用CoroutineWorker处理线程

* 对于 Kotlin 用户，WorkManager 为[协程](https://kotlinlang.org/docs/reference/coroutines-overview.html)提供了一流的支持。如要开始使用，请将 [`work-runtime-ktx` 包含到您的 gradle 文件中](https://developer.android.google.cn/jetpack/androidx/releases/work#declaring_dependencies)。不要扩展 [`Worker`](https://developer.android.google.cn/reference/androidx/work/Worker)，而应扩展 [`CoroutineWorker`](https://developer.android.google.cn/reference/kotlin/androidx/work/CoroutineWorker)，后者包含 `doWork()` 的挂起版本。例如，如果要构建一个简单的 `CoroutineWorker` 来执行某些网络操作，您需要执行以下操作：

```kotlin
class CoroutineDownloadWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        val data = downloadSynchronously("https://www.google.com")
        saveData(data)
        return Result.success()
    }
}
```

* 请注意，[`CoroutineWorker.doWork()`](https://developer.android.google.cn/reference/kotlin/androidx/work/CoroutineWorker#doWork()) 是一个“挂起”函数。此代码不同于 `Worker`，不会在 [`Configuration`](https://developer.android.google.cn/reference/androidx/work/Configuration) 中指定的 `Executor` 中运行，而是默认为 `Dispatchers.Default`。您可以提供自己的 `CoroutineContext` 来自定义这个行为。在上面的示例中，您可能希望在 `Dispatchers.IO` 上完成此操作，如下所示：

```kotlin
class CoroutineDownloadWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        withContext(Dispatchers.IO) {
            val data = downloadSynchronously("https://www.google.com")
            saveData(data)
            return Result.success()
        }
    }
}
```

* `CoroutineWorker` 通过取消协程并传播取消信号来自动处理停工情况。您无需执行任何特殊操作来处理[停工情况](https://developer.android.google.cn/topic/libraries/architecture/workmanager/how-to/managing-work#cancelling)。

### 3.1 在其他进程中运行 CoroutineWorker

* 您还可以使用 [`RemoteCoroutineWorker`](https://developer.android.google.cn/reference/kotlin/androidx/work/multiprocess/RemoteCoroutineWorker)（`ListenableWorker` 的实现）将工作器绑定到特定进程。

* `RemoteCoroutineWorker` 会使用您在构建工作请求时于输入数据中提供的两个额外参数绑定到特定进程：`ARGUMENT_CLASS_NAME` 和 `ARGUMENT_PACKAGE_NAME`。

* 以下示例演示了如何构建绑定到特定进程的工作请求：

```kotlin
val PACKAGE_NAME = "com.example.background.multiprocess"

val serviceName = RemoteWorkerService::class.java.name
val componentName = ComponentName(PACKAGE_NAME, serviceName)

val data: Data = Data.Builder()
   .putString(ARGUMENT_PACKAGE_NAME, componentName.packageName)
   .putString(ARGUMENT_CLASS_NAME, componentName.className)
   .build()

return OneTimeWorkRequest.Builder(ExampleRemoteCoroutineWorker::class.java)
   .setInputData(data)
   .build()
```

* 对于每个 `RemoteWorkerService`，您还需要在 `AndroidManifest.xml` 文件中添加服务定义：

```xml
<manifest ... >
    <service
            android:name="androidx.work.multiprocess.RemoteWorkerService"
            android:exported="false"
            android:process=":worker1" />

        <service
            android:name=".RemoteWorkerService2"
            android:exported="false"
            android:process=":worker2" />
    ...
</manifest>
```

## 四、用RxWorker处理线程

* 我们在 WorkManager 与 RxJava 之间提供互操作性。如需开始使用这种互操作性，除了在您的 gradle 文件中包含 [`work-runtime` 之外，还应包含 `work-rxjava3` 依赖项](https://developer.android.google.cn/jetpack/androidx/releases/work#declaring_dependencies)。而且还有一个支持 rxjava2 的 `work-rxjava2` 依赖项，您可以根据情况使用。

* 然后，您应该扩展 `RxWorker`，而不是扩展 `Worker`。最后替换 [`RxWorker.createWork()`](https://developer.android.google.cn/reference/androidx/work/RxWorker#createWork()) 方法以返回 `Single<Result>`，用于表示代码执行的 [`Result`](https://developer.android.google.cn/reference/androidx/work/ListenableWorker.Result)，如下所示：

```kotlin
class RxDownloadWorker(
        context: Context,
        params: WorkerParameters
) : RxWorker(context, params) {
    override fun createWork(): Single<Result> {
        return Observable.range(0, 100)
                .flatMap { download("https://www.example.com") }
                .toList()
                .map { Result.success() }
    }
}
```

* 请注意，`RxWorker.createWork()` 在主线程上调用，但默认情况下会在后台线程上订阅返回值。您可以替换 [`RxWorker.getBackgroundScheduler()`](https://developer.android.google.cn/reference/androidx/work/RxWorker#getBackgroundScheduler()) 来更改订阅线程。

* 当 `RxWorker` 为 `onStopped()` 时，系统会处理订阅，因此您无需以任何特殊方式处理[停工情况](https://developer.android.google.cn/topic/libraries/architecture/workmanager/how-to/managing-work#cancelling)。

## 五、用ListenableWorker处理线程

* 在某些情况下，您可能需要提供自定义线程处理策略。例如，您可能需要处理基于回调的异步操作。在这种情况下，不能只依靠 `Worker` 来完成操作，因为它无法以阻塞方式完成这项工作。WorkManager 通过 [`ListenableWorker`](https://developer.android.google.cn/reference/androidx/work/ListenableWorker) 支持该用例。`ListenableWorker` 是最基本的工作器 API；[`Worker`](https://developer.android.google.cn/reference/androidx/work/Worker)、[`CoroutineWorker`](https://developer.android.google.cn/reference/kotlin/androidx/work/CoroutineWorker) 和 [`RxWorker`](https://developer.android.google.cn/reference/androidx/work/RxWorker) 都是从这个类衍生而来的。`ListenableWorker` 只会发出信号以表明应该开始和停止工作，而线程处理则完全交您决定。开始工作信号在主线程上调用，因此请务必手动转到您选择的后台线程。

* 抽象方法 [`ListenableWorker.startWork()`](https://developer.android.google.cn/reference/androidx/work/ListenableWorker#startWork()) 会返回一个将使用操作的 [`Result`](https://developer.android.google.cn/reference/androidx/work/ListenableWorker.Result) 设置的 `ListenableFuture`。`ListenableFuture` 是一个轻量级接口：它是一个 `Future`，用于提供附加监听器和传播异常的功能。在 `startWork` 方法中，应该返回 `ListenableFuture`，完成操作后，您需要使用操作的 `Result` 设置这个返回结果。您可以通过以下两种方式之一创建 `ListenableFuture` 实例：

1. 如果您使用的是 Guava，请使用 `ListeningExecutorService`。
2. 否则，请将 [`councurrent-futures`](https://developer.android.google.cn/jetpack/androidx/releases/concurrent#declaring_dependencies) 包含到您的 gradle 文件中并使用 [`CallbackToFutureAdapter`](https://developer.android.google.cn/reference/androidx/concurrent/futures/CallbackToFutureAdapter)。

* 如果您希望基于异步回调执行某些工作，则应以类似如下的方式执行：

```kotlin
class CallbackWorker(
        context: Context,
        params: WorkerParameters
) : ListenableWorker(context, params) {
    override fun startWork(): ListenableFuture<Result> {
        return CallbackToFutureAdapter.getFuture { completer ->
            val callback = object : Callback {
                var successes = 0

                override fun onFailure(call: Call, e: IOException) {
                    completer.setException(e)
                }

                override fun onResponse(call: Call, response: Response) {
                    successes++
                    if (successes == 100) {
                        completer.set(Result.success())
                    }
                }
            }

            repeat(100) {
                downloadAsynchronously("https://example.com", callback)
            }

            callback
        }
    }
}
```

* 如果您的工作[停止](https://developer.android.google.cn/topic/libraries/architecture/workmanager/how-to/managing-work#cancelling)会发生什么？如果预计工作会停止，则始终会取消 `ListenableWorker` 的 `ListenableFuture`。通过使用 `CallbackToFutureAdapter`，您只需添加一个取消监听器即可，如下所示：

```kotlin
class CallbackWorker(
        context: Context,
        params: WorkerParameters
) : ListenableWorker(context, params) {
    override fun startWork(): ListenableFuture<Result> {
        return CallbackToFutureAdapter.getFuture { completer ->
            val callback = object : Callback {
                var successes = 0

                override fun onFailure(call: Call, e: IOException) {
                    completer.setException(e)
                }

                override fun onResponse(call: Call, response: Response) {
                    ++successes
                    if (successes == 100) {
                        completer.set(Result.success())
                    }
                }
            }

 completer.addCancellationListener(cancelDownloadsRunnable, executor)

            repeat(100) {
                downloadAsynchronously("https://example.com", callback)
            }

            callback
        }
    }
}
```

## 5.1 在其他进程中运行 ListenableWorker

* 您还可以使用 [`RemoteListenableWorker`](https://developer.android.google.cn/reference/kotlin/androidx/work/multiprocess/RemoteListenableWorker)（`ListenableWorker` 的实现）将工作器绑定到特定进程。

* `RemoteListenableWorker` 会使用您在构建工作请求时于输入数据中提供的两个额外参数绑定到特定进程：`ARGUMENT_CLASS_NAME` 和 `ARGUMENT_PACKAGE_NAME`。

* 以下示例演示了如何构建绑定到特定进程的工作请求：

```kotlin
val PACKAGE_NAME = "com.example.background.multiprocess"

val serviceName = RemoteWorkerService::class.java.name
val componentName = ComponentName(PACKAGE_NAME, serviceName)

val data: Data = Data.Builder()
   .putString(ARGUMENT_PACKAGE_NAME, componentName.packageName)
   .putString(ARGUMENT_CLASS_NAME, componentName.className)
   .build()

return OneTimeWorkRequest.Builder(ExampleRemoteListenableWorker::class.java)
   .setInputData(data)
   .build()
```

* 对于每个 `RemoteWorkerService`，您还需要在 `AndroidManifest.xml` 文件中添加服务定义：

```xml
<manifest ... >
    <service
            android:name="androidx.work.multiprocess.RemoteWorkerService"
            android:exported="false"
            android:process=":worker1" />

        <service
            android:name=".RemoteWorkerService2"
            android:exported="false"
            android:process=":worker2" />
    ...
</manifest>
```