[TOC]

# 定义工作请求

* [入门指南](https://developer.android.google.cn/topic/libraries/architecture/workmanager/basics)介绍了如何创建简单的 `WorkRequest` 并将其加入队列。

* 在本指南中，您将了解如何定义和自定义 `WorkRequest` 对象来处理常见用例，例如：
  * 调度一次性工作和重复性工作
  * 设置工作约束条件，例如要求连接到 Wi-Fi 网络或正在充电
  * 确保至少延迟一定时间再执行工作
  * 设置重试和退避策略
  * 将输入数据传递给工作
  * 使用标记将相关工作分组在一起

## 一、概览

* 工作通过 `WorkRequest` 在 WorkManager 中进行定义。为了使用 WorkManager 调度任何工作，您必须先创建一个 `WorkRequest` 对象，然后将其加入队列。

```kotlin
val myWorkRequest = ...
WorkManager.getInstance(myContext).enqueue(myWorkRequest)
```

* WorkRequest 对象包含 WorkManager 调度和运行工作所需的所有信息。其中包括运行工作必须满足的约束、调度信息（例如延迟或重复间隔）、重试配置，并且可能包含输入数据（如果工作需要）。

* `WorkRequest` 本身是抽象基类。该类有两个派生实现，可用于创建 `OneTimeWorkRequest` 和 `PeriodicWorkRequest` 请求。顾名思义，`OneTimeWorkRequest` 适用于调度非重复性工作，而 `PeriodicWorkRequest` 则更适合调度以一定间隔重复执行的工作。

## 二、调度一次性工作

* 对于无需额外配置的简单工作，请使用静态方法 `from`：

```kotlin
val myWorkRequest = OneTimeWorkRequest.from(MyWork::class.java)
```

* 对于更复杂的工作，可以使用构建器。

```kotlin
val uploadWorkRequest: WorkRequest =
   OneTimeWorkRequestBuilder<MyWork>()
       // Additional configuration
       .build()
```

## 三、调度定期工作

* 您的应用有时可能需要定期运行某些工作。例如，您可能要定期备份数据、定期下载应用中的新鲜内容或者定期上传日志到服务器。

* 使用 `PeriodicWorkRequest` 创建定期执行的 `WorkRequest` 对象的方法如下：

```kotlin
val saveRequest =
       PeriodicWorkRequestBuilder<SaveImageToFileWorker>(1, TimeUnit.HOURS)
    // Additional configuration
           .build()
```

* 在此示例中，工作的运行时间间隔定为一小时。

* 时间间隔定义为两次重复执行之间的最短时间。工作器的确切执行时间取决于您在 WorkRequest 对象中设置的约束以及系统执行的优化。

* **注意**：可以定义的最短重复间隔是 15 分钟（与 [JobScheduler API](https://developer.android.google.cn/reference/android/app/job/JobScheduler) 相同）。

### 3.1 灵活的运行间隔

* 如果您的工作的性质致使其对运行时间敏感，您可以将 `PeriodicWorkRequest` 配置为在每个时间间隔的**灵活时间段**内运行，如图 1 所示。

![您可以为定期作业设置一个灵活间隔。您要定义一个重复间隔，然后再定义一个灵活间隔（指定一个在重复间隔末尾开始的具体时间段）。WorkManager 会尝试在每个周期的灵活间隔内运行作业。](https://developer.android.google.cn/images/topic/libraries/architecture/workmanager/how-to/definework-flex-period.png)

* **图 1.** 此图显示了可在灵活时间段内运行工作的的重复间隔。

* 如需定义具有灵活时间段的定期工作，请在创建 `PeriodicWorkRequest` 时传递 `flexInterval` 以及 `repeatInterval`。灵活时间段从 `repeatInterval - flexInterval` 开始，一直到间隔结束。

* 以下是可在每小时的最后 15 分钟内运行的定期工作的示例。

```kotlin
val myUploadWork = PeriodicWorkRequestBuilder<SaveImageToFileWorker>(
       1, TimeUnit.HOURS, // repeatInterval (the period cycle)
       15, TimeUnit.MINUTES) // flexInterval
    .build()
```

* 重复间隔必须大于或等于 `PeriodicWorkRequest.MIN_PERIODIC_INTERVAL_MILLIS`（十五分钟），而灵活间隔必须大于或等于 `PeriodicWorkRequest.MIN_PERIODIC_FLEX_MILLIS`（五分钟）。

### 3.2 约束对定期工作的影响

* 您可以对定期工作设置[约束](https://developer.android.google.cn/topic/libraries/architecture/workmanager/how-to/define-work#work-constraints)。例如，您可以为工作请求添加约束，以便工作仅在用户设备充电时运行。在这种情况下，除非满足约束条件，否则即使过了定义的重复间隔，`PeriodicWorkRequest` 也不会运行。这可能会导致工作在某次运行时出现延迟，甚至会因在相应间隔内未满足条件而被跳过。

## 四、工作约束

* [约束](https://developer.android.google.cn/reference/androidx/work/Constraints)可确保将工作延迟到满足最佳条件时运行。以下约束适用于 WorkManager。

| **NetworkType**      | 约束运行工作所需的[网络类型](https://developer.android.google.cn/reference/androidx/work/NetworkType)。例如 Wi-Fi (`UNMETERED`)。 |
| -------------------- | ------------------------------------------------------------ |
| **BatteryNotLow**    | 如果设置为 true，那么当设备处于“电量不足模式”时，工作不会运行。 |
| **RequiresCharging** | 如果设置为 true，那么工作只能在设备充电时运行。              |
| **DeviceIdle**       | 如果设置为 true，则要求用户的设备必须处于空闲状态，才能运行工作。如果您要运行批量操作，否则可能会降低用户设备上正在积极运行的其他应用的性能，建议您使用此约束。 |
| **StorageNotLow**    | 如果设置为 true，那么当用户设备上的存储空间不足时，工作不会运行。 |

* 如需创建一组约束并将其与某项工作相关联，请使用一个 `Contraints.Builder()` 创建 `Constraints` 实例，并将该实例分配给 `WorkRequest.Builder()`。

* 例如，以下代码会构建了一个工作请求，该工作请求仅在用户设备正在充电且连接到 Wi-Fi 网络时才会运行：

```kotlin
val constraints = Constraints.Builder()
   .setRequiredNetworkType(NetworkType.UNMETERED)
   .setRequiresCharging(true)
   .build()

val myWorkRequest: WorkRequest =
   OneTimeWorkRequestBuilder<MyWork>()
       .setConstraints(constraints)
       .build()
```

* 如果指定了多个约束，工作将仅在满足所有约束时才会运行。

* 如果在工作运行时不再满足某个约束，WorkManager 将停止工作器。系统将在满足所有约束后重试工作。

## 五、延迟工作

* 如果工作没有约束，或者当工作加入队列时所有约束都得到了满足，那么系统可能会选择立即运行该工作。如果您不希望工作立即运行，可以将工作指定为在经过一段最短初始延迟时间后再启动。

* 下面举例说明了如何将工作设置为在加入队列后至少经过 10 分钟后再运行。

```kotlin
val myWorkRequest = OneTimeWorkRequestBuilder<MyWork>()
   .setInitialDelay(10, TimeUnit.MINUTES)
   .build()
```

* 该示例说明了如何为 `OneTimeWorkRequest` 设置初始延迟时间，您也可以为 `PeriodicWorkRequest` 设置初始延迟时间。在这种情况下，定期工作只有首次运行时会延迟。

* **注意**：执行工作器的确切时间还取决于 WorkRequest 中使用的约束和系统优化方式。WorkManager 经过设计，能够在满足这些约束的情况下提供可能的最佳行为。

## 六、重试和退避政策

* 如果您需要让 WorkManager 重试工作，可以从工作器返回 `Result.retry()`。然后，系统将根据[退避延迟时间](https://developer.android.google.cn/reference/androidx/work/WorkRequest#DEFAULT_BACKOFF_DELAY_MILLIS)和[退避政策](https://developer.android.google.cn/reference/androidx/work/BackoffPolicy)重新调度工作。
  * 退避延迟时间指定了首次尝试后重试工作前的最短等待时间。此值不能超过 10 秒（或 [MIN_BACKOFF_MILLIS](https://developer.android.google.cn/reference/androidx/work/WorkRequest#MIN_BACKOFF_MILLIS)）。
  * 退避政策定义了在后续重试过程中，退避延迟时间随时间以怎样的方式增长。WorkManager 支持 2 个退避政策，即 `LINEAR` 和 `EXPONENTIAL`。

* 每个工作请求都有退避政策和退避延迟时间。默认政策是 `EXPONENTIAL`，延迟时间为 10 秒，但您可以在工作请求配置中替换此设置。

* 以下是自定义退避延迟时间和政策的示例。

```kotlin
val myWorkRequest = OneTimeWorkRequestBuilder<MyWork>()
   .setBackoffCriteria(
       BackoffPolicy.LINEAR,
       OneTimeWorkRequest.MIN_BACKOFF_MILLIS,
       TimeUnit.MILLISECONDS)
   .build()
```

* 在本示例中，最短退避延迟时间设置为允许的最小值，即 10 秒。由于政策为 `LINEAR`，每次尝试重试时，重试间隔都会增加约 10 秒。例如，第一次运行以 `Result.retry()` 结束并在 10 秒后重试；然后，如果工作在后续尝试后继续返回 `Result.retry()`，那么接下来会在 20 秒、30 秒、40 秒后重试，以此类推。如果退避政策设置为 `EXPONENTIAL`，那么重试时长序列将接近 20、40、80 秒，以此类推。

* **注意**：退避延迟时间不精确，在两次重试之间可能会有几秒钟的差异，但绝不会低于配置中指定的初始退避延迟时间。

## 七、标记工作

* 每个工作请求都有一个[唯一标识符](https://developer.android.google.cn/reference/androidx/work/WorkRequest#getId())，该标识符可用于在以后标识该工作，以便[取消](https://developer.android.google.cn/topic/libraries/architecture/workmanager/how-to/managing-work#cancelling)工作或[观察其进度](https://developer.android.google.cn/topic/libraries/architecture/workmanager/how-to/states-and-observation#observing)。

* 如果有一组在逻辑上相关的工作，对这些工作项进行标记可能也会很有帮助。通过标记，您一起处理一组工作请求。

* 例如，`WorkManager.cancelAllWorkByTag(String)` 会取消带有特定标记的所有工作请求，`WorkManager.getWorkInfosByTag(String)` 会返回一个 WorkInfo 对象列表，该列表可用于确定当前工作状态。

* 以下代码展示了如何向工作添加“cleanup”标记：

```kotlin
val myWorkRequest = OneTimeWorkRequestBuilder<MyWork>()
   .addTag("cleanup")
   .build()
```

* 最后，可以向单个工作请求添加多个标记。这些标记在内部以一组字符串的形式进行存储。对于工作请求，您可以通过 [WorkRequest.getTags()](https://developer.android.google.cn/reference/androidx/work/ListenableWorker#getTags()) 检索其标记集。

## 八、分配输入数据

* 您的工作可能需要输入数据才能正常运行。例如，处理图片上传的工作可能需要使用待上传图片的 URI 作为输入数据。

* 输入值以键值对的形式存储在 `Data` 对象中，并且可以在工作请求中设置。WorkManager 会在执行工作时将输入 `Data` 传递给工作。`Worker` 类可通过调用 `Worker.getInputData()` 访问输入参数。以下代码展示了如何创建需要输入数据的 `Worker` 实例，以及如何在工作请求中发送该实例。

```kotlin
// Define the Worker requiring input
class UploadWork(appContext: Context, workerParams: WorkerParameters)
   : Worker(appContext, workerParams) {

   override fun doWork(): Result {
       val imageUriInput =
           inputData.getString("IMAGE_URI") ?: return Result.failure()

       uploadFile(imageUriInput)
       return Result.success()
   }
   ...
}

// Create a WorkRequest for your Worker and sending it input
val myUploadWork = OneTimeWorkRequestBuilder<UploadWork>()
   .setInputData(workDataOf(
       "IMAGE_URI" to "http://..."
   ))
   .build()
```

* 同样，可使用 `Data` 类输出返回值。如需详细了解输入和输出数据，请参阅[输入参数和返回值](https://developer.android.google.cn/topic/libraries/architecture/workmanager/advanced#params)部分。

## 九、后续步骤

* 在[状态和观察](https://developer.android.google.cn/topic/libraries/architecture/workmanager/how-to/states-and-observation)页面中，您将详细了解工作状态以及如何监控工作的进度。