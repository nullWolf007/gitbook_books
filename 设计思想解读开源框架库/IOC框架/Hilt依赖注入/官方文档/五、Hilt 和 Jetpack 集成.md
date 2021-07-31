[TOC]

## Hilt 和 Jetpack 集成

* Hilt 包含一些扩展，用于提供其他 Jetpack 库中的类。Hilt 目前支持以下 Jetpack 组件：
  * `ViewModel`
  * Navigation
  * Compose
  * WorkManager
  
* 您必须添加 Hilt 依赖项，才能利用这些集成。如需详细了解如何添加依赖项，请参阅[使用 Hilt 实现依赖项注入](https://developer.android.google.cn/training/dependency-injection/hilt-android?hl=zh_cn#setup)。

* **注意**：这些集成目前依赖于 Alpha 版库。Alpha 版功能稳定，但功能可能不完整。在版本处于 Alpha 版状态时，可能会添加、移除或更改 API。

### 一、使用 Hilt 注入 ViewModel 对象

* 将下面这些额外的依赖项添加到 Gradle 文件中。请注意，除了库之外，您还需要添加一个额外的注释处理器，它在 Hilt 注释处理器的基础上运行：

* app/build.gradle

```groovy
...
dependencies {
  ...
  implementation 'androidx.hilt:hilt-lifecycle-viewmodel:1.0.0-alpha01'
  // When using Kotlin.
  kapt 'androidx.hilt:hilt-compiler:1.0.0-alpha01'
  // When using Java.
  annotationProcessor 'androidx.hilt:hilt-compiler:1.0.0-alpha01'
}
```

* 在 `ViewModel` 对象的构造函数中使用 `@HiltViewModel ` 和 `@Inject` 注释来提供一个 [`ViewModel`](https://developer.android.google.cn/topic/libraries/architecture/viewmodel?hl=zh_cn)。

```kotlin
@HiltViewModel
class ExampleViewModel @Inject constructor(
  private val savedStateHandle: SavedStateHandle,
  private val repository: ExampleRepository
) : ViewModel() {
  ...
}
```

* 然后，带有 `@AndroidEntryPoint` 注释的 Activity 或 Fragment 可以使用 `ViewModelProvider` 或 `by viewModels()` [KTX 扩展](https://developer.android.google.cn/kotlin/ktx?hl=zh_cn)照常获取 `ViewModel` 实例：

```kotlin
@AndroidEntryPoint
class ExampleActivity : AppCompatActivity() {
  private val exampleViewModel: ExampleViewModel by viewModels()
  ...
}
```

#### @ViewModelScoped

All Hilt ViewModels are provided by the `ViewModelComponent` which follows the same lifecycle as a `ViewModel`, and as such, can survive configuration changes. To scope a dependency to a `ViewModel` use the `@ViewModelScoped` annotation.

A `@ViewModelScoped` type will make it so that a single instance of the scoped type is provided across all dependencies injected into the `ViewModel`. Other instances of a ViewModel that request the scoped instance will receive a different instance.

If a single instance needs to be shared across various ViewModels, then it should be scoped using either `@ActivityRetainedScoped` or `@Singleton`.

* 大概意思就是，提供了@ViewModelScoped来限制组件的作用域，@ViewModelScoped提供给ViewModelComponent组件用，在ViewModel的create创建，ViewModel的destoryed时销毁
* 一个@ViewModelScoped将会保证所有注解的ViewModel将会是同一个实例，而其他的ViewModel去请求将会收到不一样的实例
* 如果一个实例需要去分享给所有的ViewModels，那么需要使用组件作用域类型的`@ActivityRetainedScoped` 或者 `@Singleton`.

### 二、和Jetpack的Navigation 结合

* 将下面这些额外的依赖项添加到 Gradle 文件中。

* app/build.gradle

```groovy
dependencies {
    ...
    implementation 'androidx.hilt:hilt-navigation-fragment:1.0.0'
}
```

* If your `ViewModel` is [scoped to the navigation graph](https://developer.android.com/guide/navigation/navigation-programmatic#share_ui-related_data_between_destinations_with_viewmodel), use the `hiltNavGraphViewModels` function that works with fragments that are annotated with `@AndroidEntryPoint`.
* 如果你的ViewModel是限定在Navigation，使用hiltNavGraphViewModels函数，对应的fragments也需要进行@AndroidEntryPoint注解

```kotlin
val viewModel: ExampleViewModel by hiltNavGraphViewModels(R.id.my_graph)
```

### 三、和Jetpack的Compose结合

* To see how Hilt integrates with Jetpack Compose, see the Hilt section of [Compose and other libraries](https://developer.android.com/jetpack/compose/libraries#hilt).
* 可以访问 [Compose and other libraries](https://developer.android.com/jetpack/compose/libraries#hilt).去查看

### 四、使用 Hilt 注入 WorkManager

* 将下面这些额外的依赖项添加到 Gradle 文件中。请注意，除了库之外，您还需要添加一个额外的注释处理器，它在 Hilt 注释处理器的基础上运行：

* app/build.gradle

```groovy
dependencies {
  ...
  implementation 'androidx.hilt:hilt-work:1.0.0'
  // When using Kotlin.
  kapt 'androidx.hilt:hilt-compiler:1.0.0'
  // When using Java.
  annotationProcessor 'androidx.hilt:hilt-compiler:1.0.0'
}
```

* 在 `Worker` 对象的构造函数中使用 `@HiltWorker` 注释和 `@AssistedInject` 来注入一个 [`Worker`](https://developer.android.google.cn/reference/kotlin/androidx/work/Worker?hl=zh_cn)。您只能在 `Worker` 对象中使用 `@Singleton` 或未限定作用域的绑定。您还必须使用 `@Assisted` 为 `Context` 和 `WorkerParameters` 依赖项添加注释：

```kotlin
@HiltWorker
class ExampleWorker @AssistedInject constructor(
  @Assisted appContext: Context,
  @Assisted workerParams: WorkerParameters,
  workerDependency: WorkerDependency
) : Worker(appContext, workerParams) { ... }
```

* 然后，让 `Application` 类实现 `Configuration.Provider` 接口，注入 `HiltWorkFactory` 的实例，并将其传入 `WorkManager` 配置，如下所示：

```kotlin
@HiltAndroidApp
class ExampleApplication : Application(), Configuration.Provider {

  @Inject lateinit var workerFactory: HiltWorkerFactory

  override fun getWorkManagerConfiguration() =
      Configuration.Builder()
            .setWorkerFactory(workerFactory)
            .build()
}
```