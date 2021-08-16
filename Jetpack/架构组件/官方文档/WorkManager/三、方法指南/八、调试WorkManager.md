[TOC]

# 调试 WorkManager

* 如果您发现您的工作器运行过于频繁或根本没有运行，请按照以下几个调试步骤操作，以发现可能存在的问题。

## 一、启用日志记录

* 如需确定工作器未正确运行的原因，查看详细的 WorkManager 日志很有帮助。如需启用日志记录功能，您需要使用[自定义初始化](https://developer.android.google.cn/topic/libraries/architecture/workmanager/advanced/custom-configuration)。首先，通过创建应用了清单合并规则 **`remove`** 的新 WorkManager 提供程序来停用 `AndroidManifest.xml` 中的默认 `WorkManagerInitializer`：

```xml
<provider
    android:name="androidx.work.impl.WorkManagerInitializer"
    android:authorities="${applicationId}.workmanager-init"
    tools:node="remove"/>
```

* 现在，默认 WorkManager 初始化程序已停用，您可以使用[按需初始化](https://developer.android.google.cn/topic/libraries/architecture/workmanager/advanced/custom-configuration#implement-configuration-provider)。为此，`android.app.Application` 类必须提供 `androidx.work.Configuration.Provider` 的实现。

```kotlin
class MyApplication() : Application(), Configuration.Provider {
    override fun getWorkManagerConfiguration() =
        Configuration.Builder()
            .setMinimumLoggingLevel(android.util.Log.DEBUG)
            .build()
}
```

* 在您定义自定义 WorkManager 配置后，WorkManager 会在您调用 [`WorkManager.getInstance(Context)`](https://developer.android.google.cn/reference/androidx/work/WorkManager#getInstance(android.content.Context)) 时进行初始化，而不是在应用启动时自动初始化。如需了解详情，包括对 2.1.0 之前的 WorkManager 版本提供的支持，请参阅[自定义 WorkManager 配置和初始化](https://developer.android.google.cn/topic/libraries/architecture/workmanager/advanced/custom-configuration)。

* 启用调试日志记录后，系统会开始显示更多包含日志标记前缀 `WM-` 的日志。

## 二、使用 adb shell dumpsys jobscheduler

* 搭载 API 级别 23 或更高版本时，您可以运行命令 `adb shell dumpsys jobscheduler` 来查看已归因于您的软件包的作业列表。

* 您应该会看到与以下类似的内容：

```
JOB #u0a172/4: 6412553 com.google.android.youtube/androidx.work.impl.background.systemjob.SystemJobService
  u0a172 tag=*job*/com.google.android.youtube/androidx.work.impl.background.systemjob.SystemJobService
  Source: uid=u0a172 user=0 pkg=com.google.android.youtube
  JobInfo:
    Service: com.google.android.youtube/androidx.work.impl.background.systemjob.SystemJobService
    Requires: charging=false batteryNotLow=false deviceIdle=false
    Extras: mParcelledData.dataSize=180
    Network type: NetworkRequest [ NONE id=0, [ Capabilities: NOT_METERED&INTERNET&NOT_RESTRICTED&TRUSTED&VALIDATED Uid: 10172] ]
    Minimum latency: +1h29m59s687ms
    Backoff: policy=1 initial=+30s0ms
    Has early constraint
  Required constraints: TIMING_DELAY CONNECTIVITY [0x90000000]
  Satisfied constraints: DEVICE_NOT_DOZING BACKGROUND_NOT_RESTRICTED WITHIN_QUOTA [0x3400000]
  Unsatisfied constraints: TIMING_DELAY CONNECTIVITY [0x90000000]
  Tracking: CONNECTIVITY TIME QUOTA
  Implicit constraints:
    readyNotDozing: true
    readyNotRestrictedInBg: true
  Standby bucket: RARE
  Base heartbeat: 0
  Enqueue time: -51m29s853ms
  Run time: earliest=+38m29s834ms, latest=none, original latest=none
  Last run heartbeat: 0
  Ready: false (job=false user=true !pending=true !active=true !backingup=true comp=true)
```

* 使用 WorkManager 时，负责管理工作器执行的组件为 `SystemJobService`（搭载 API 级别 23 或更高版本）。您应该查找归因于您的软件包名称和 `androidx.work.impl.background.systemjob.SystemJobService` 的作业实例。

* 对于每个作业，命令的输出都会列出**必需**、**满足**和**不满足**约束条件。您应该检查是否完全满足工作器的约束条件。

* 您可以利用它检查最近是否调用了 `SystemJobService`。输出还包括最近执行的作业的作业历史记录：

```
Job history:
     -1h35m26s440ms   START: #u0a107/9008 com.google.android.youtube/androidx.work.impl.background.systemjob.SystemJobService
     -1h35m26s362ms  STOP-P: #u0a107/9008 com.google.android.youtube/androidx.work.impl.background.systemjob.SystemJobService app called jobFinished
```

## 三、从 WorkManager 2.4.0 及更高版本请求诊断信息

* 在应用的调试 build 中，您可以使用以下命令从 WorkManager 2.4.0 及更高版本请求诊断信息：

```
adb shell am broadcast -a "androidx.work.diagnostics.REQUEST_DIAGNOSTICS" -p "<your_app_package_name>"
```

* 这提供了以下方面的信息：
  * 在过去 24 小时内完成的工作请求。
  * 目前正在运行的工作请求。
  * 预定运行的工作请求。

* 诊断信息如下所示（输出通过 `logcat` 显示）：

```
adb shell am broadcast -a "androidx.work.diagnostics.REQUEST_DIAGNOSTICS" -p "androidx.work.integration.testapp"

adb logcat
...
2020-02-13 14:21:37.990 29528-29660/androidx.work.integration.testapp I/WM-DiagnosticsWrkr: Recently completed work:
2020-02-13 14:21:38.083 29528-29660/androidx.work.integration.testapp I/WM-DiagnosticsWrkr: Id  Class Name   State  Unique Name Tags
    08be261c-2def-4bd6-a716-1e4410968dc4     androidx.work.impl.workers.DiagnosticsWorker    SUCCEEDED  null    androidx.work.impl.workers.DiagnosticsWorker
    48ce04f1-8df9-450b-96ec-6eceabb9c690     androidx.work.impl.workers.DiagnosticsWorker    SUCCEEDED  null    androidx.work.impl.workers.DiagnosticsWorker
    c46f4699-c384-440c-a10e-26d56ce02963     androidx.work.impl.workers.DiagnosticsWorker    SUCCEEDED  null    androidx.work.impl.workers.DiagnosticsWorker
    ce125372-046e-484e-949f-9abb35ce62c3     androidx.work.impl.workers.DiagnosticsWorker    SUCCEEDED  null    androidx.work.impl.workers.DiagnosticsWorker
    72887ddd-8ed1-4018-b798-fac218e95e16     androidx.work.impl.workers.DiagnosticsWorker    SUCCEEDED  null    androidx.work.impl.workers.DiagnosticsWorker
    dcff3d61-320d-4996-8644-5d97944bd09c     androidx.work.impl.workers.DiagnosticsWorker    SUCCEEDED  null    androidx.work.impl.workers.DiagnosticsWorker
    acab0bf7-6087-43ad-bdb5-be0df9195acb     androidx.work.impl.workers.DiagnosticsWorker    SUCCEEDED  null    androidx.work.impl.workers.DiagnosticsWorker
    23136bcd-01dd-46eb-b910-0fe8a140c2a4     androidx.work.integration.testapp.ToastWorker   SUCCEEDED  null    androidx.work.integration.testapp.ToastWorker
    245f4879-c6d2-4997-8130-e4e90e1cab4c     androidx.work.integration.testapp.ToastWorker   SUCCEEDED  null    androidx.work.integration.testapp.ToastWorker
    17d05835-bb61-429a-ad11-fe43fc320a54     androidx.work.integration.testapp.ToastWorker   SUCCEEDED  null    androidx.work.integration.testapp.ToastWorker
    e95f12be-4b0c-4e64-88da-8ee07a31e42f     androidx.work.integration.testapp.ToastWorker   SUCCEEDED  null    androidx.work.integration.testapp.ToastWorker
    431c3ec2-4a55-469b-b50b-4072d35f1232     androidx.work.integration.testapp.ToastWorker   SUCCEEDED  null    androidx.work.integration.testapp.ToastWorker
    883a388f-f911-4098-9143-37bd8fbc098a     androidx.work.integration.testapp.ToastWorker   SUCCEEDED  null    androidx.work.integration.testapp.ToastWorker
    b904163c-6822-4299-8d5a-78df49b7e53d     androidx.work.integration.testapp.ToastWorker   SUCCEEDED  null    androidx.work.integration.testapp.ToastWorker
    453fd7b9-2b16-45b9-abc5-3d2ce7b6a4ba     androidx.work.integration.testapp.ToastWorker   SUCCEEDED  null    androidx.work.integration.testapp.ToastWorker
2020-02-13 14:21:38.083 29528-29660/androidx.work.integration.testapp I/WM-DiagnosticsWrkr: Running work:
2020-02-13 14:21:38.089 29528-29660/androidx.work.integration.testapp I/WM-DiagnosticsWrkr: Id  Class Name   State  Unique Name Tags
    b87c8a4f-4ac6-4e25-ba3e-4cea53ce468a     androidx.work.impl.workers.DiagnosticsWorker    RUNNING    null    androidx.work.impl.workers.DiagnosticsWorker
...
```