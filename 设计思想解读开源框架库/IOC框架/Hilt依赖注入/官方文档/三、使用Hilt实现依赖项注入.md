[TOC]

## 使用 Hilt 实现依赖项注入

* Hilt 是 Android 的依赖项注入库，可减少在项目中执行手动依赖项注入的样板代码。执行[手动依赖项注入](https://developer.android.google.cn/training/dependency-injection/manual?hl=zh_cn)要求您手动构造每个类及其依赖项，并借助容器重复使用和管理依赖项。

* Hilt 通过为项目中的每个 Android 类提供容器并自动管理其生命周期，提供了一种在应用中使用 DI（依赖项注入）的标准方法。Hilt 在热门 DI 库 [Dagger](https://developer.android.google.cn/training/dependency-injection/dagger-basics?hl=zh_cn) 的基础上构建而成，因而能够受益于 Dagger 的编译时正确性、运行时性能、可伸缩性和 [Android Studio 支持](https://medium.com/androiddevelopers/dagger-navigation-support-in-android-studio-49aa5d149ec9)。如需了解详情，请参阅 [Hilt 和 Dagger](https://developer.android.google.cn/training/dependency-injection/hilt-android?hl=zh_cn#hilt-and-dagger)。

* 本指南介绍了 Hilt 的基本概念及其生成的容器，，还演示了如何开始在现有应用中使用 Hilt。

### 一、添加依赖项

* 首先，将 `hilt-android-gradle-plugin` 插件添加到项目的根级 `build.gradle` 文件中：

```groovy
buildscript {
    ...
    dependencies {
        ...
        classpath 'com.google.dagger:hilt-android-gradle-plugin:2.28-alpha'
    }
}
```

* 然后，应用 Gradle 插件并在 `app/build.gradle` 文件中添加以下依赖项：

```groovy
...
apply plugin: 'kotlin-kapt'
apply plugin: 'dagger.hilt.android.plugin'

android {
    ...
}

dependencies {
    implementation "com.google.dagger:hilt-android:2.28-alpha"
    kapt "com.google.dagger:hilt-android-compiler:2.28-alpha"
}
```

* **注意**：同时使用 Hilt 和[数据绑定](https://developer.android.google.cn/topic/libraries/data-binding?hl=zh_cn)的项目需要 Android Studio 4.0 或更高版本。

* Hilt 使用 [Java 8 功能](https://developer.android.google.cn/studio/write/java8-support?hl=zh_cn)。如需在项目中启用 Java 8，请将以下代码添加到 `app/build.gradle` 文件中：

```groovy
android {
  ...
  compileOptions {
    sourceCompatibility JavaVersion.VERSION_1_8
    targetCompatibility JavaVersion.VERSION_1_8
  }
}
```

### 二、Hilt 应用类

* 所有使用 Hilt 的应用都必须包含一个带有 `@HiltAndroidApp` 注释的 `Application` 类。

* `@HiltAndroidApp` 会触发 Hilt 的代码生成操作，生成的代码包括应用的一个基类，该基类充当应用级依赖项容器。

```kotlin
@HiltAndroidApp
class MyApplication : Application() {
}
```

* 生成的这一 Hilt 组件会附加到 `Application` 对象的生命周期，并为其提供依赖项。此外，它也是应用的父组件，这意味着，其他组件可以访问它提供的依赖项。

### 三、将依赖项注入 Android 类

* 在 `Application` 类中设置了 Hilt 且有了应用级组件后，Hilt 可以为带有 `@AndroidEntryPoint` 注释的其他 Android 类提供依赖项：

```kotlin
@AndroidEntryPoint
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
    }
}
```

* Hilt 目前支持以下 Android 类：
  * `Application`（通过使用 `@HiltAndroidApp`）
  * `Activity`
  * `Fragment`
  * `View`
  * `Service`
  * `BroadcastReceiver`

* 如果您使用 `@AndroidEntryPoint` 为某个 Android 类添加注释，则还必须为依赖于该类的 Android 类添加注释。例如，如果您为某个 Fragment 添加注释，则还必须为使用该 Fragment 的所有 Activity 添加注释。

* **注意**：在 Hilt 对 Android 类的支持方面还要注意以下几点：**Hilt 仅支持扩展 [`ComponentActivity`](https://developer.android.google.cn/reference/kotlin/androidx/activity/ComponentActivity?hl=zh_cn) 的 Activity，如 [`AppCompatActivity`](https://developer.android.google.cn/reference/kotlin/androidx/appcompat/app/AppCompatActivity?hl=zh_cn)。Hilt 仅支持扩展 `androidx.Fragment` 的 Fragment。Hilt 不支持保留的 Fragment**。

* `@AndroidEntryPoint` 会为项目中的每个 Android 类生成一个单独的 Hilt 组件。这些组件可以从它们各自的父类接收依赖项，如[组件层次结构](https://developer.android.google.cn/training/dependency-injection/hilt-android?hl=zh_cn#component-hierarchy)中所述。

* 如需从组件获取依赖项，请使用 `@Inject` 注释执行字段注入：

```kotlin
@AndroidEntryPoint
class MainActivity : AppCompatActivity() {
	@Inject lateinit var analytics: AnalyticsAdapter
	...
}
```

* **注意**：**由 Hilt 注入的字段不能为私有字段。尝试使用 Hilt 注入私有字段会导致编译错误。**

* Hilt 注入的类可以有同样使用注入的其他基类。如果这些类是抽象类，则它们不需要 `@AndroidEntryPoint` 注释。

* 如需详细了解 Android 类被注入的是哪个生命周期回调，请参阅[组件生命周期](https://developer.android.google.cn/training/dependency-injection/hilt-android?hl=zh_cn#component-lifetimes)。

### 四、定义 Hilt 绑定

* 为了执行字段注入，Hilt 需要知道如何从相应组件提供必要依赖项的实例。“绑定”包含将某个类型的实例作为依赖项提供所需的信息。

* 向 Hilt 提供绑定信息的一种方法是构造函数注入。在某个类的构造函数中使用 `@Inject` 注释，以告知 Hilt 如何提供该类的实例：

```kotlin
class AnalyticsAdapter @Inject constructor(
  private val service: AnalyticsService
) { ... }
```

* 在一个类的代码中，带有注释的构造函数的参数即是该类的依赖项。在本例中，`AnalyticsService` 是 `AnalyticsAdapter` 的一个依赖项。因此，Hilt 还必须知道如何提供 `AnalyticsService` 的实例。

* **注意**：在构建时，Hilt 会为 Android 类生成 [Dagger](https://developer.android.google.cn/training/dependency-injection/dagger-basics?hl=zh_cn) 组件。然后，Dagger 会走查您的代码，并执行以下步骤：构建并验证依赖关系图，确保没有未满足的依赖关系且没有依赖循环。生成它在运行时用来创建实际对象及其依赖项的类。

### 五、Hilt 模块

* 有时，类型不能通过构造函数注入。发生这种情况可能有多种原因。例如，您不能通过构造函数注入接口。此外，您也不能通过构造函数注入不归您所有的类型，如来自外部库的类。在这些情况下，您可以使用 Hilt 模块向 Hilt 提供绑定信息。

* Hilt 模块是一个带有 `@Module` 注释的类。与 [Dagger 模块](https://developer.android.google.cn/training/dependency-injection/dagger-android?hl=zh_cn#dagger-modules)一样，它会告知 Hilt 如何提供某些类型的实例。与 Dagger 模块不同的是，您必须使用 `@InstallIn` 为 Hilt 模块添加注释，以告知 Hilt 每个模块将用在或安装在哪个 Android 类中。

* **注意**：Hilt 模块与 [Gradle 模块](https://developer.android.google.cn/studio/projects?hl=zh_cn#ApplicationModules)不同。

* 您在 Hilt 模块中提供的依赖项可以在生成的所有与 Hilt 模块安装到的 Android 类关联的组件中使用。

* **注意**：由于 Hilt 的代码生成操作需要访问使用 Hilt 的所有 Gradle 模块，因此编译 `Application` 类的 Gradle 模块还需要在其传递依赖项中包含您的所有 Hilt 模块和通过构造函数注入的类。

#### 5.1 使用 @Binds 注入接口实例

* 以 `AnalyticsService` 为例。如果 `AnalyticsService` 是一个接口，则您无法通过构造函数注入它，而应向 Hilt 提供绑定信息，方法是在 Hilt 模块内创建一个带有 `@Binds` 注释的抽象函数。

* `@Binds` 注释会告知 Hilt 在需要提供接口的实例时要使用哪种实现。

* 带有注释的函数会向 Hilt 提供以下信息：
  * 函数返回类型会告知 Hilt 函数提供哪个接口的实例。
  * 函数参数会告知 Hilt 要提供哪种实现。

```kotlin
interface AnalyticsService {
  fun analyticsMethods()
}

// Constructor-injected, because Hilt needs to know how to
// provide instances of AnalyticsServiceImpl, too.
class AnalyticsServiceImpl @Inject constructor(
  ...
) : AnalyticsService { ... }

@Module
@InstallIn(ActivityComponent::class)
abstract class AnalyticsModule {

  @Binds
  abstract fun bindAnalyticsService(
    analyticsServiceImpl: AnalyticsServiceImpl
  ): AnalyticsService
}
```

* Hilt 模块 `AnalyticsModule` 带有 `@InstallIn(ActivityComponent::class)` 注释，因为您希望 Hilt 将该依赖项注入 `MainActivity`。此注释意味着，`AnalyticsModule` 中的所有依赖项都可以在应用的所有 Activity 中使用。

#### 5.2 使用 @Provides 注入实例

* 接口不是无法通过构造函数注入类型的唯一一种情况。如果某个类不归您所有（因为它来自外部库，如 [Retrofit](https://square.github.io/retrofit/)、[`OkHttpClient`](https://square.github.io/okhttp/) 或 [Room 数据库](https://developer.android.google.cn/topic/libraries/architecture/room?hl=zh_cn)等类），或者必须使用[构建器模式](https://en.wikipedia.org/wiki/Builder_pattern)创建实例，也无法通过构造函数注入。

* 接着前面的例子来讲。如果 `AnalyticsService` 类不直接归您所有，您可以告知 Hilt 如何提供此类型的实例，方法是在 Hilt 模块内创建一个函数，并使用 `@Provides` 为该函数添加注释。

* 带有注释的函数会向 Hilt 提供以下信息：
  * 函数返回类型会告知 Hilt 函数提供哪个类型的实例。
  * 函数参数会告知 Hilt 相应类型的依赖项。
  * 函数主体会告知 Hilt 如何提供相应类型的实例。每当需要提供该类型的实例时，Hilt 都会执行函数主体。

```kotlin
@Module
@InstallIn(ActivityComponent::class)
object AnalyticsModule {

  @Provides
  fun provideAnalyticsService(
    // Potential dependencies of this type
  ): AnalyticsService {
      return Retrofit.Builder()
               .baseUrl("https://example.com")
               .build()
               .create(AnalyticsService::class.java)
  }
}
```

#### 5.3 为同一类型提供多个绑定

* 如果您需要让 Hilt 以依赖项的形式提供同一类型的不同实现，必须向 Hilt 提供多个绑定。您可以使用限定符为同一类型定义多个绑定。

* 限定符是一种注释，当为某个类型定义了多个绑定时，您可以使用它来标识该类型的特定绑定。

* 仍然接着前面的例子来讲。如果需要拦截对 `AnalyticsService` 的调用，您可以使用带有[拦截器](https://square.github.io/okhttp/interceptors/)的 `OkHttpClient` 对象。对于其他服务，您可能需要以不同的方式拦截调用。在这种情况下，您需要告知 Hilt 如何提供两种不同的 `OkHttpClient` 实现。

* 首先，定义要用于为 `@Binds` 或 `@Provides` 方法添加注释的限定符：
* **Qualifier的意思是合格者，通过这个标示，表明了哪个实现类才是我们所需要的，添加@Qualifier注解，可以作为筛选的限定符，我们在做自定义注解时可以在其定义上增加@Qualifier，用来筛选需要的对象**
* **`AnnotationRetention.BINARY`：存储在编译后的 Class 文件，但是反射不可见。**

```kotlin
@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class AuthInterceptorOkHttpClient

@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class OtherInterceptorOkHttpClient
```

* 然后，Hilt 需要知道如何提供与每个限定符对应的类型的实例。在这种情况下，您可以使用带有 `@Provides` 的 Hilt 模块。这两种方法具有相同的返回类型，但限定符将它们标记为两个不同的绑定：

```kotlin
@Module
@InstallIn(ApplicationComponent::class)
object NetworkModule {

  @AuthInterceptorOkHttpClient
  @Provides
  fun provideAuthInterceptorOkHttpClient(
    authInterceptor: AuthInterceptor
  ): OkHttpClient {
      return OkHttpClient.Builder()
               .addInterceptor(authInterceptor)
               .build()
  }

  @OtherInterceptorOkHttpClient
  @Provides
  fun provideOtherInterceptorOkHttpClient(
    otherInterceptor: OtherInterceptor
  ): OkHttpClient {
      return OkHttpClient.Builder()
               .addInterceptor(otherInterceptor)
               .build()
  }
}
```

* 您可以通过使用相应的限定符为字段或参数添加注释来注入所需的特定类型：

```kotlin
// As a dependency of another class.
@Module
@InstallIn(ActivityComponent::class)
object AnalyticsModule {

  @Provides
  fun provideAnalyticsService(
    @AuthInterceptorOkHttpClient okHttpClient: OkHttpClient
  ): AnalyticsService {
      return Retrofit.Builder()
               .baseUrl("https://example.com")
               .client(okHttpClient)
               .build()
               .create(AnalyticsService::class.java)
  }
}

// As a dependency of a constructor-injected class.
class ExampleServiceImpl @Inject constructor(
  @AuthInterceptorOkHttpClient private val okHttpClient: OkHttpClient
) : ...

// At field injection.
@AndroidEntryPoint
class ExampleActivity: AppCompatActivity() {

  @AuthInterceptorOkHttpClient
  @Inject lateinit var okHttpClient: OkHttpClient
}
```

* 最佳做法是，如果您向某个类型添加限定符，应向提供该依赖项的所有可能的方式添加限定符。让基本实现或通用实现不带限定符容易出错，并且可能会导致 Hilt 注入错误的依赖项。

#### 5.4 Hilt 中的预定义限定符

* Hilt 提供了一些预定义的限定符。例如，由于您可能需要来自应用或 Activity 的 `Context` 类，因此 Hilt 提供了 `@ApplicationContext` 和 `@ActivityContext` 限定符。

* 假设本例中的 `AnalyticsAdapter` 类需要 Activity 的上下文。以下代码演示了如何向 `AnalyticsAdapter` 提供 Activity 上下文：

```kotlin
class AnalyticsAdapter @Inject constructor(
    @ActivityContext private val context: Context,
    private val service: AnalyticsService
) { ... }
```

* 如需了解 Hilt 中提供的其他预定义绑定，请参阅[组件默认绑定](https://developer.android.google.cn/training/dependency-injection/hilt-android?hl=zh_cn#component-default)。

### 六、为 Android 类生成的组件

* 对于您可以从中执行字段注入的每个 Android 类，都有一个关联的 Hilt 组件，您可以在 `@InstallIn` 注释中引用该组件。每个 Hilt 组件负责将其绑定注入相应的 Android 类。

* 前面的示例演示了如何在 Hilt 模块中使用 `ActivityComponent`。

* Hilt 提供了以下组件：

| Hilt 组件                   | 注入器面向的对象                              |
| :-------------------------- | :-------------------------------------------- |
| `SingletonComponent`        | `Application`                                 |
| `ActivityRetainedComponent` | N/A                                           |
| `ViewModelComponent`        | `ViewModel`                                   |
| `ActivityComponent`         | `Activity`                                    |
| `FragmentComponent`         | `Fragment`                                    |
| `ViewComponent`             | `View`                                        |
| `ViewWithFragmentComponent` | `View` annotated with `@WithFragmentBindings` |

* **注意**：Hilt 不会为广播接收器生成组件，因为 Hilt 直接从 `SingletonComponent` 注入广播接收器。

#### 6.1 组件生命周期

* Hilt 会按照相应 Android 类的生命周期自动创建和销毁生成的组件类的实例。

| 生成的组件                  | 创建时机                 | 销毁时机                  |
| :-------------------------- | :----------------------- | :------------------------ |
| `SingletonComponent`        | `Application#onCreate()` | `Application#onDestroy()` |
| `ActivityRetainedComponent` | `Activity#onCreate()`    | `Activity#onDestroy()`    |
| `ViewModelComponent`        | `ViewModel` created      | `ViewModel` destroyed     |
| `ActivityComponent`         | `Activity#onCreate()`    | `Activity#onDestroy()`    |
| `FragmentComponent`         | `Fragment#onAttach()`    | `Fragment#onDestroy()`    |
| `ViewComponent`             | `View#super()`           | View destroyed            |
| `ViewWithFragmentComponent` | `View#super()`           | View destroyed            |
| `ServiceComponent`          | `Service#onCreate()`     | `Service#onDestroy()`     |

* **注意**：`ActivityRetainedComponent` 在配置更改后仍然存在，因此它在第一次调用 `Activity#onCreate()` 时创建，在最后一次调用 `Activity#onDestroy()` 时销毁。

#### 6.2 组件作用域

* 默认情况下，Hilt 中的所有绑定都未限定作用域。这意味着，每当应用请求绑定时，Hilt 都会创建所需类型的一个新实例。

* 在本例中，每当 Hilt 提供 `AnalyticsAdapter` 作为其他类型的依赖项或通过字段注入提供它（如在 `ExampleActivity` 中）时，Hilt 都会提供 `AnalyticsAdapter` 的一个新实例。

* 不过，Hilt 也允许将绑定的作用域限定为特定组件。Hilt 只为绑定作用域限定到的组件的每个实例创建一次限定作用域的绑定，对该绑定的所有请求共享同一实例。

* 下表列出了生成的每个组件的作用域注释：

| Android 类                                    | 生成的组件                  | 作用域                    |
| :-------------------------------------------- | :-------------------------- | :------------------------ |
| `Application`                                 | `SingletonComponent`        | `@Singleton`              |
| `Activity`                                    | `ActivityRetainedComponent` | `@ActivityRetainedScoped` |
| `ViewModel`                                   | `ViewModelComponent`        | `@ViewModelScoped`        |
| `Activity`                                    | `ActivityComponent`         | `@ActivityScoped`         |
| `Fragment`                                    | `FragmentComponent`         | `@FragmentScoped`         |
| `View`                                        | `ViewComponent`             | `@ViewScoped`             |
| `View` annotated with `@WithFragmentBindings` | `ViewWithFragmentComponent` | `@ViewScoped`             |
| `Service`                                     | `ServiceComponent`          | `@ServiceScoped`          |

* 在本例中，如果您使用 `@ActivityScoped` 将 `AnalyticsAdapter` 的作用域限定为 `ActivityComponent`，Hilt 会在相应 Activity 的整个生命周期内提供 `AnalyticsAdapter` 的同一实例：

```kotlin
@ActivityScoped
class AnalyticsAdapter @Inject constructor(
  private val service: AnalyticsService
) { ... }
```

* **注意**：将绑定的作用域限定为某个组件的成本可能很高，因为提供的对象在该组件被销毁之前一直保留在内存中。请在应用中尽量少用限定作用域的绑定。如果绑定的内部状态要求在某一作用域内使用同一实例，或者绑定的创建成本很高，那么将绑定的作用域限定为某个组件是一种恰当的做法。

* 假设 `AnalyticsService` 的内部状态要求每次都使用同一实例 - 不只是在 `ExampleActivity` 中，而是在应用中的任何位置。在这种情况下，将 `AnalyticsService` 的作用域限定为 `ApplicationComponent` 是一种恰当的做法。结果是，每当组件需要提供 `AnalyticsService` 的实例时，都会提供同一实例。

* 以下示例演示了如何将绑定的作用域限定为 Hilt 模块中的某个组件。绑定的作用域必须与其安装到的组件的作用域一致，因此在本例中，您必须将 `AnalyticsService` 安装在 `ApplicationComponent` 中，而不是安装在 `ActivityComponent` 中：

```kotlin
// If AnalyticsService is an interface.
@Module
@InstallIn(ApplicationComponent::class)
abstract class AnalyticsModule {

  @Singleton
  @Binds
  abstract fun bindAnalyticsService(
    analyticsServiceImpl: AnalyticsServiceImpl
  ): AnalyticsService
}

// If you don't own AnalyticsService.
@Module
@InstallIn(ApplicationComponent::class)
object AnalyticsModule {

  @Singleton
  @Provides
  fun provideAnalyticsService(): AnalyticsService {
      return Retrofit.Builder()
               .baseUrl("https://example.com")
               .build()
               .create(AnalyticsService::class.java)
  }
}
```

#### 6.3 组件层次结构

* 将模块安装到组件后，其绑定就可以用作该组件中其他绑定的依赖项，也可以用作组件层次结构中该组件下的任何子组件中其他绑定的依赖项：

![ViewWithFragmentComponent 位于 FragmentComponent 下。FragmentComponent 和 ViewComponent 位于 ActivityComponent 下。ActivityComponent 位于 ActivityRetainedComponent 下。ActivityRetainedComponent 和 ServiceComponent 位于 ApplicationComponent 下。](https://developer.android.google.cn/images/training/dependency-injection/hilt-hierarchy.svg?hl=zh_cn) 

*  图 1. Hilt 生成的组件的层次结构。

* **注意**：默认情况下，如果您在视图中执行字段注入，`ViewComponent` 可以使用 `ActivityComponent` 中定义的绑定。如果您还需要使用 `FragmentComponent` 中定义的绑定并且视图是 Fragment 的一部分，应将 `@WithFragmentBindings` 注释和 `@AndroidEntryPoint` 一起使用。

#### 6.4 组件默认绑定

* 每个 Hilt 组件都附带一组默认绑定，Hilt 可以将其作为依赖项注入您自己的自定义绑定。请注意，这些绑定对应于常规 Activity 和 Fragment 类型，而不对应于任何特定子类。这是因为，Hilt 会使用单个 Activity 组件定义来注入所有 Activity。每个 Activity 都有此组件的不同实例。

| Android 组件                | 默认绑定                                      |
| :-------------------------- | :-------------------------------------------- |
| `SingletonComponent`        | `Application`                                 |
| `ActivityRetainedComponent` | `Application`                                 |
| `ViewModelComponent`        | `SavedStateHandle`                            |
| `ActivityComponent`         | `Application`, `Activity`                     |
| `FragmentComponent`         | `Application`, `Activity`, `Fragment`         |
| `ViewComponent`             | `Application`, `Activity`, `View`             |
| `ViewWithFragmentComponent` | `Application`, `Activity`, `Fragment`, `View` |
| `ServiceComponent`          | `Application`, `Service`                      |

* 还可以使用 `@ApplicationContext` 获得应用上下文绑定。例如：

```kotlin
class AnalyticsServiceImpl @Inject constructor(
  @ApplicationContext context: Context
) : AnalyticsService { ... }

// The Application binding is available without qualifiers.
class AnalyticsServiceImpl @Inject constructor(
  application: Application
) : AnalyticsService { ... }
```

* 此外，还可以使用 `@ActivityContext` 获得 Activity 上下文绑定。例如：

```kotlin
class AnalyticsAdapter @Inject constructor(
  @ActivityContext context: Context
) { ... }

// The Activity binding is available without qualifiers.
class AnalyticsAdapter @Inject constructor(
  activity: FragmentActivity
) { ... }
```

### 七、在 Hilt 不支持的类中注入依赖项

* Hilt 支持最常见的 Android 类。不过，您可能需要在 Hilt 不支持的类中执行字段注入。

* 在这些情况下，您可以使用 `@EntryPoint` 注释创建入口点。入口点是由 Hilt 管理的代码与并非由 Hilt 管理的代码之间的边界。它是代码首次进入 Hilt 所管理对象的图的位置。入口点允许 Hilt 使用它并不管理的代码提供依赖关系图中的依赖项。

* 例如，Hilt 并不直接支持[内容提供程序](https://developer.android.google.cn/guide/topics/providers/content-providers?hl=zh_cn)。如果您希望内容提供程序使用 Hilt 来获取某些依赖项，需要为所需的每个绑定类型定义一个带有 `@EntryPoint` 注释的接口并添加限定符。然后，添加 `@InstallIn` 以指定要在其中安装入口点的组件，如下所示：

```kotlin
class ExampleContentProvider : ContentProvider() {

  @EntryPoint
  @InstallIn(ApplicationComponent::class)
  interface ExampleContentProviderEntryPoint {
    fun analyticsService(): AnalyticsService
  }

  ...
}
```

* 如需访问入口点，请使用来自 `EntryPointAccessors` 的适当静态方法。参数应该是组件实例或充当组件持有者的 `@AndroidEntryPoint` 对象。确保您以参数形式传递的组件和 `EntryPointAccessors` 静态方法都与 `@EntryPoint` 接口上的 `@InstallIn` 注释中的 Android 类匹配：

```kotlin
class ExampleContentProvider: ContentProvider() {
    ...

  override fun query(...): Cursor {
    val appContext = context?.applicationContext ?: throw IllegalStateException()
    val hiltEntryPoint =
      EntryPointAccessors.fromApplication(appContext, ExampleContentProviderEntryPoint::class.java)

    val analyticsService = hiltEntryPoint.analyticsService()
    ...
  }
}
```

* 在本例中，您必须使用 `ApplicationContext` 检索入口点，因为入口点安装在 `ApplicationComponent` 中。如果您要检索的绑定位于 `ActivityComponent` 中，应改用 `ActivityContext`。

### 八、Hilt 和 Dagger

* Hilt 在依赖项注入库 [Dagger](https://dagger.dev/) 的基础上构建而成，提供了一种将 Dagger 纳入 Android 应用的标准方法。

* 关于 Dagger，Hilt 的目标如下：
  * 简化 Android 应用的 Dagger 相关基础架构。
  * 创建一组标准的组件和作用域，以简化设置、提高可读性以及在应用之间共享代码。
  * 提供一种简单的方法来为各种构建类型（如测试、调试或发布）配置不同的绑定。

* 由于 Android 操作系统会实例化它自己的许多框架类，因此在 Android 应用中使用 Dagger 要求您编写大量的样板。Hilt 可减少在 Android 应用中使用 Dagger 所涉及的样板代码。Hilt 会自动生成并提供以下各项：
  * **用于将 Android 框架类与 Dagger 集成的组件** - 您不必手动创建。
  * **作用域注释** - 与 Hilt 自动生成的组件一起使用。
  * **预定义的绑定** - 表示 Android 类，如 `Application` 或 `Activity`。
  * **预定义的限定符** - 表示 `@ApplicationContext` 和 `@ActivityContext`。

* Dagger 和 Hilt 代码可以共存于同一代码库中。不过，在大多数情况下，最好使用 Hilt 管理您在 Android 上对 Dagger 的所有使用。如需将使用 Dagger 的项目迁移到 Hilt，请参阅[迁移指南](https://dagger.dev/hilt/migration-guide)和[“将 Dagger 应用迁移到 Hilt”Codelab](https://codelabs.developers.google.com/codelabs/android-dagger-to-hilt?hl=zh_cn)。

### 九、其他资源

* 如需详细了解 Hilt，请参阅下面列出的其他资源。

### 十、示例

- [Android 架构蓝图 - Hilt](https://github.com/android/architecture-samples/tree/dev-hilt)

### 十一、Codelab

- [在 Android 应用中使用 Hilt](https://codelabs.developers.google.com/codelabs/android-hilt/?hl=zh_cn)
- [将 Dagger 应用迁移到 Hilt](https://codelabs.developers.google.com/codelabs/android-dagger-to-hilt/?hl=zh_cn)

### 十二、博客

- [使用 Hilt 在 Android 上实现依赖项注入](https://medium.com/androiddevelopers/dependency-injection-on-android-with-hilt-67b6031e62d)