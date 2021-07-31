[TOC]

# 在 Android 应用中使用 Dagger

* [Dagger 基础知识](https://developer.android.google.cn/training/dependency-injection/dagger-basics?hl=zh_cn)页面介绍了 Dagger 可以如何帮助您在应用中自动注入依赖项。使用 Dagger，您就不必编写冗长乏味且容易出错的样板代码。

* **注意**：使用 [Hilt](https://developer.android.google.cn/training/dependency-injection/hilt-android?hl=zh_cn) 可在 Android 上实现依赖项注入。Hilt 在 Dagger 的基础上构建而成，提供了一种在 Android 应用中纳入 Dagger 依赖项注入框架的标准方法。

## 一、最佳做法摘要

* **注意**：如果您已熟悉 Dagger，请查看以下最佳做法。如果不熟悉，请先阅读上述页面的内容，然后再返回本页面。

- 如果有可能，请通过`@Inject`进行构造函数注入，以向 Dagger 图中添加类型。如果没有可能，请执行以下操作：
  - 使用 `@Binds` 告知 Dagger 接口应采用哪种实现。
  - 使用 `@Provides` 告知 Dagger 如何提供您的项目所不具备的类。
- 您只能在组件中声明一次模块。
- 根据注释的使用生命周期，为作用域注释命名。示例包括 `@ApplicationScope`、`@LoggedUserScope` 和 `@ActivityScope`。

## 二、添加依赖项

* 如需在项目中使用 Dagger，请在 `build.gradle` 文件中向应用添加以下依赖项。您可以[在此 GitHub 项目中](https://github.com/google/dagger/releases)找到最新版本的 Dagger。

```kotlin
apply plugin: 'kotlin-kapt'

dependencies {
    implementation 'com.google.dagger:dagger:2.x'
    kapt 'com.google.dagger:dagger-compiler:2.x'
}
```

## 三、Android 中的 Dagger

* 假设有一个示例 Android 应用的依赖关系图如图 1 所示。

<img src="https://developer.android.google.cn/images/training/dependency-injection/4-application-graph.png?hl=zh_cn" alt="LoginActivity 依赖于 LoginViewModel，LoginViewModel 依赖于 UserRepository，UserRepository 依赖于 UserLocalDataSource 和 UserRemoteDataSource，而 UserRemoteDataSource 又依赖于 Retrofit。" style="zoom:67%;" />

* **图 1.** 示例代码的依赖关系图

* 在 Android 中，您通常会创建一个位于应用类中的 Dagger 图，因为只要应用处于运行状态，您就希望该图的实例在内存中。这样，就将该图附加到应用生命周期。在某些情况下，您可能还希望在该图中提供应用上下文。为此，您还需要使该图位于应用类中。这种方法的一个优势在于，该图可供其他 Android 框架类使用。此外，它还允许您在测试中使用自定义应用类，从而简化了测试。

* 由于生成该图的接口带有 `@Component` 注释，因此您可以将其命名为 `ApplicationComponent` 或 `ApplicationGraph`。您通常会将该组件的实例保留在自定义应用类中，并在每次需要应用图时调用该实例，如以下代码段所示：

```kotlin
// Definition of the Application graph
@Component
interface ApplicationComponent { ... }

// appComponent lives in the Application class to share its lifecycle
class MyApplication: Application() {
    // Reference to the application graph that is used across the whole app
    val appComponent = DaggerApplicationComponent.create()
}
```

* 由于某些 Android 框架类（如 Activity 和 Fragment）由系统实例化，因此 Dagger 无法为您创建这些类。具体而言，对于 Activity，任何初始化代码都需要放入 `onCreate()` 方法中。这意味着，无法像在前面的示例中那样，在类的构造函数中使用 `@Inject` 注释（构造函数注入）。必须改为使用字段注入。

* **注意**：[Dagger 基础知识](https://developer.android.google.cn/training/dependency-injection/dagger-basics?hl=zh_cn)页面介绍了如何在构造函数中使用 Dagger `@Inject` 注释。此注释会告知 Dagger 如何创建类的实例。

* 您希望 Dagger 为您填充 Activity 所需的依赖项，而不是在 `onCreate()` 方法中创建这些依赖项。对于字段注入，应将 `@Inject` 注释应用于您要从 Dagger 图中获取的字段。

```kotlin
class LoginActivity: Activity() {
    // You want Dagger to provide an instance of LoginViewModel from the graph
    @Inject lateinit var loginViewModel: LoginViewModel
}
```

* 为简单起见，`LoginViewModel` 不是 [Android 架构组件 ViewModel](https://developer.android.google.cn/topic/libraries/architecture/viewmodel?hl=zh_cn)，而只是充当 ViewModel 的常规类。如需详细了解如何注入这些类，请参阅 [dev-dagger](https://github.com/android/architecture-samples/tree/dev-dagger) 分支中的官方 [Android Blueprints Dagger 实现](https://github.com/android/architecture-samples/tree/dev-dagger)中的代码。

* 有关 Dagger 的一个注意事项是，注入的字段不能为私有字段。这些字段的公开范围必须至少为软件包私有，如前面的代码所示。

* **注意**：字段注入只能在无法使用构造函数注入的 Android 框架类中使用。

### 3.1 注入 Activity

* Dagger 需要知道 `LoginActivity` 必须访问该图才能提供其所需的 `ViewModel`。在 [Dagger 基础知识](https://developer.android.google.cn/training/dependency-injection/dagger-basics?hl=zh_cn)页面中，您使用 `@Component` 接口从该图中获取了一些对象，方法是提供了一些函数，让其返回类型为您要从该图中获取的内容。在这种情况下，您需要告知 Dagger 要求注入依赖项的对象（在本例中为 `LoginActivity`）。为此，您应提供一个函数，让该函数将请求注入的对象作为参数。

```kotlin
@Component
interface ApplicationComponent {
    // This tells Dagger that LoginActivity requests injection so the graph needs to
    // satisfy all the dependencies of the fields that LoginActivity is requesting.
    fun inject(activity: LoginActivity)
}
```

* 此函数会告知 Dagger `LoginActivity` 希望访问该图并请求注入。Dagger 需要满足 `LoginActivity` 所需的所有依赖项（`LoginViewModel` 具有自己的依赖项）。如果您有多个请求注入的类，则必须通过这些类的确切类型在组件中明确声明它们。例如，如果您有请求注入的 `LoginActivity` 和 `RegistrationActivity`，则需要两种 `inject()` 方法，而不是涵盖这两种情况的一种通用方法。通用 `inject()` 方法不会告知 Dagger 需要提供的内容。接口中的函数可以具有任何名称，但在它们以参数形式接收要注入的对象时将其称为 `inject()` 是 Dagger 中的一种惯例。

* 如需在 Activity 中注入对象，您应使用应用类中定义的 `appComponent` 并调用 `inject()` 方法，传入请求注入的 Activity 实例。

* 使用 Activity 时，应在调用 `super.onCreate()` 之前在 Activity 的 `onCreate()` 方法中注入 Dagger，以避免出现 Fragment 恢复问题。在 `super.onCreate()` 中的恢复阶段，Activity 会附加可能需要访问 Activity 绑定的 Fragment。

* 使用 Fragment 时，应在 Fragment 的 `onAttach()` 方法中注入 Dagger。在这种情况下，此操作可以在调用 `super.onAttach()` 之前或之后完成。

```kotlin
class LoginActivity: Activity() {
    // You want Dagger to provide an instance of LoginViewModel from the graph
    @Inject lateinit var loginViewModel: LoginViewModel

    override fun onCreate(savedInstanceState: Bundle?) {
        // Make Dagger instantiate @Inject fields in LoginActivity
        (applicationContext as MyApplication).appComponent.inject(this)
        // Now loginViewModel is available

        super.onCreate(savedInstanceState)
    }
}

// @Inject tells Dagger how to create instances of LoginViewModel
class LoginViewModel @Inject constructor(
    private val userRepository: UserRepository
) { ... }
```

* 下面我们告知 Dagger 如何提供剩余依赖项以构建该图：

```kotlin
class UserRepository @Inject constructor(
    private val localDataSource: UserLocalDataSource,
    private val remoteDataSource: UserRemoteDataSource
) { ... }

class UserLocalDataSource @Inject constructor() { ... }
class UserRemoteDataSource @Inject constructor(
    private val loginService: LoginRetrofitService
) { ... }
```

### 3.2 Dagger 模块

* 在本例中，您使用的是 [Retrofit](https://square.github.io/retrofit/) 网络库。`UserRemoteDataSource` 依赖于 `LoginRetrofitService`。不过，创建 `LoginRetrofitService` 实例的方法与您到目前为止一直执行的操作有所不同。它不是类实例化，而是调用 `Retrofit.Builder()` 并传入不同参数以配置登录服务的结果。

* 除了 `@Inject` 注释之外，还有一种方法可告知 Dagger 如何提供类实例，即使用 Dagger 模块中的信息。Dagger 模块是一个带有 `@Module` 注释的类。您可以在其中使用 `@Provides` 注释定义依赖项。

```kotlin
// @Module informs Dagger that this class is a Dagger Module
@Module
class NetworkModule {

    // @Provides tell Dagger how to create instances of the type that this function
    // returns (i.e. LoginRetrofitService).
    // Function parameters are the dependencies of this type.
    @Provides
    fun provideLoginRetrofitService(): LoginRetrofitService {
        // Whenever Dagger needs to provide an instance of type LoginRetrofitService,
        // this code (the one inside the @Provides method) is run.
        return Retrofit.Builder()
                .baseUrl("https://example.com")
                .build()
                .create(LoginService::class.java)
    }
}
```

* **注意**：您可以在 Dagger 模块中使用 `@Provides` 注释告知 Dagger 如何提供您的项目所不具备的类（例如，`Retrofit` 的实例）。

* 对于接口的实现，最佳做法是使用 [`@Binds`](https://dagger.dev/faq.html#binds)，如[“在 Android 应用中使用 Dagger”Codelab](https://codelabs.developers.google.com/codelabs/android-dagger/?hl=zh_cn) 中所述。

* 模块是一种以语义方式封装有关如何提供对象信息的方法。可以看到，您调用了 `NetworkModule` 类，以对提供网络相关对象的逻辑进行分组。如果应用扩展了，您还可以在此处添加如何提供 [`OkHttpClient`](https://square.github.io/okhttp/) 或者如何配置 [Gson](https://github.com/google/gson) 或 [Moshi](https://github.com/square/moshi) 的逻辑。

* `@Provides` 方法的依赖项是该方法的参数。对于上述方法，可以提供没有依赖项的 `LoginRetrofitService`，因为该方法没有参数。如果您已将 `OkHttpClient` 声明为参数，Dagger 将需要提供图中的 `OkHttpClient` 实例以满足 `LoginRetrofitService` 的依赖项要求。例如：

```kotlin
@Module
class NetworkModule {
    // Hypothetical dependency on LoginRetrofitService
    @Provides
    fun provideLoginRetrofitService(
        okHttpClient: OkHttpClient
    ): LoginRetrofitService { ... }
}
```

* 为了使 Dagger 图了解此模块，您必须将其添加到 `@Component` 接口，如下所示：

```kotlin
// The "modules" attribute in the @Component annotation tells Dagger what Modules
// to include when building the graph
@Component(modules = [NetworkModule::class])
interface ApplicationComponent {
    ...
}
```

* 如需将类型添加到 Dagger 图，建议的方法是使用构造函数注入（即在类的构造函数上使用 `@Inject` 注释）。有时，此方法不可行，您必须使用 Dagger 模块。例如，您希望 Dagger 使用计算结果确定如何创建对象实例时。每当必须提供该类型的实例时，Dagger 就会运行 `@Provides` 方法中的代码。

* 示例中的 Dagger 图目前如下所示：

![LoginActivity 依赖图的示意图](https://developer.android.google.cn/images/training/dependency-injection/4-graph-login.png?hl=zh_cn)

* **图 2.** 由 Dagger 注入 `LoginActivity` 的图的表示法

* 图的入口点为 `LoginActivity`。由于 `LoginActivity` 注入了 `LoginViewModel`，因此 Dagger 构建的图知道如何提供 `LoginViewModel` 的实例，以及如何以递归方式提供其依赖项的实例。Dagger 知道如何执行此操作，因为类的构造函数上有 `@Inject` 注释。

* 在由 Dagger 生成的 `ApplicationComponent` 内，有一种 factory 类型方法，可用于获取它知道如何提供的所有类的实例。在本例中，Dagger 委托给 `ApplicationComponent` 中包含的 `NetworkModule` 来获取 `LoginRetrofitService` 的实例。

### 3.3 Dagger 作用域

* [Dagger 基础知识](https://developer.android.google.cn/training/dependency-injection/dagger-basics?hl=zh_cn)页面提到了作用域，作用域可以用来在组件中指定某个类型的唯一实例。这就是将类型的作用域限定为组件生命周期的意义所在。

* 由于您可能希望在应用的其他功能中使用 `UserRepository`，并且可能不希望在每次需要时创建新对象，因此您可以将其指定为整个应用中的唯一实例。这同样适用于 `LoginRetrofitService`：创建成本可能很高，并且您还希望重复使用该对象的唯一实例。`UserRemoteDataSource` 实例的创建成本并不那么高，因此没有必要将其作用域限定为组件的生命周期。

* [`@Singleton`](https://docs.oracle.com/javaee/7/api/javax/inject/Singleton.html) 是 `javax.inject` 软件包随附的唯一一个作用域注释。您可以使用它为 `ApplicationComponent` 以及要在整个应用中重复使用的对象添加注释。

* **注意**：您可以使用**任何**作用域注释在组件中指定某个类型的唯一实例，只要该组件和类型带有该注释即可。`@Singleton` 随附于 Dagger 库，通常用于为应用组件添加注释，但您可以创建具有不同名称的自定义注释（例如 `@ApplicationScope`）。

```kotlin
@Singleton
@Component(modules = [NetworkModule::class])
interface ApplicationComponent {
    fun inject(activity: LoginActivity)
}

@Singleton
class UserRepository @Inject constructor(
    private val localDataSource: UserLocalDataSource,
    private val remoteDataSource: UserRemoteDataSource
) { ... }

@Module
class NetworkModule {
    // Way to scope types inside a Dagger Module
    @Singleton
    @Provides
    fun provideLoginRetrofitService(): LoginRetrofitService { ... }
}
```

* **注意**：使用作用域注释的模块只能在带有相同作用域注释的组件中使用。

* 请注意，在将作用域应用于对象时，不要引发内存泄漏。只要限定作用域的组件在内存中，创建的对象就也在内存中。因为 `ApplicationComponent` 是在应用启动时（在应用类中）创建的，所以它会随着应用的销毁而被销毁。因此，`UserRepository` 的唯一实例将始终保留在内存中，直到应用被销毁。

* **注意**：使用构造函数注入（通过 `@Inject`）时，应在类中添加作用域注释；使用 Dagger 模块时，应在 `@Provides` 方法中添加作用域注释。

### 3.4 Dagger 子组件

* 如果登录流程（由单个 `LoginActivity` 管理）由多个 Fragment 组成，您应在所有 Fragment 中重复使用 `LoginViewModel` 的同一实例。`@Singleton` 无法为 `LoginViewModel` 添加注释以重复使用该实例，原因如下：

1. 流程结束后，`LoginViewModel` 的实例将继续保留在内存中。
2. 您希望为每个登录流程使用不同的 `LoginViewModel` 实例。例如，如果用户退出，您希望使用不同的 `LoginViewModel` 实例，而不是用户首次登录时的实例。

* 如需将 `LoginViewModel` 的作用域限定为 `LoginActivity` 的生命周期，您需要为登录流程创建新组件（新子图）和新作用域。

* 我们来创建一个特定于登录流程的图。

```kotlin
@Component
interface LoginComponent {}
```

* 现在，`LoginActivity` 应从 `LoginComponent` 获得注入，因为它具有特定于登录的配置。这可免于从 `ApplicationComponent` 类注入 `LoginActivity`。

```kotlin
@Component
interface LoginComponent {
    fun inject(activity: LoginActivity)
}
```

* `LoginComponent` 必须能够访问 `ApplicationComponent` 中的对象，因为 `LoginViewModel` 依赖于 `UserRepository`。如需告知 Dagger 您希望新组件使用其他组件的一部分，方法是使用 Dagger 子组件。新组件必须是包含共享资源的组件的子组件。

* 子组件是继承并扩展父组件的对象图的组件。因此，父组件中提供的所有对象也将在子组件中提供。这样，子组件中的对象就可以依赖于父组件提供的对象。

* 如需创建子组件的实例，您需要父组件的实例。因此，父组件向子组件提供的对象的作用域仍限定为父组件。

* 在本例中，您必须将 `LoginComponent` 定义为 `ApplicationComponent` 的子组件。为此，请使用 `@Subcomponent` 为 `LoginComponent` 添加注释：

```kotlin
// @Subcomponent annotation informs Dagger this interface is a Dagger Subcomponent
@Subcomponent
interface LoginComponent {

    // This tells Dagger that LoginActivity requests injection from LoginComponent
    // so that this subcomponent graph needs to satisfy all the dependencies of the
    // fields that LoginActivity is injecting
    fun inject(loginActivity: LoginActivity)
}
```

* 您还必须在 `LoginComponent` 内定义子组件 factory，以便 `ApplicationComponent` 知道如何创建 `LoginComponent` 的实例。

```kotlin
@Subcomponent
interface LoginComponent {

    // Factory that is used to create instances of this subcomponent
    @Subcomponent.Factory
    interface Factory {
        fun create(): LoginComponent
    }

    fun inject(loginActivity: LoginActivity)
}
```

* 如需告知 Dagger `LoginComponent` 是 `ApplicationComponent` 的子组件，您必须通过以下方式予以指明：

1. 创建新的 Dagger 模块（例如 `SubcomponentsModule`），并将子组件的类传递给注释的 `subcomponents` 属性。

   ```kotlin
   // The "subcomponents" attribute in the @Module annotation tells Dagger what
   // Subcomponents are children of the Component this module is included in.
   @Module(subcomponents = LoginComponent::class)
   class SubcomponentsModule {}
   ```

2. 将新模块（即 `SubcomponentsModule`）添加到 `ApplicationComponent`：

   ```kotlin
   // Including SubcomponentsModule, tell ApplicationComponent that
   // LoginComponent is its subcomponent.
   @Singleton
   @Component(modules = [NetworkModule::class, SubcomponentsModule::class])
   interface ApplicationComponent {
   }
   ```

   * 请注意，`ApplicationComponent` 不再需要注入 `LoginActivity`，因为现在由 `LoginComponent` 负责注入，因此您可以从 `ApplicationComponent` 中移除 `inject()` 方法。

   * `ApplicationComponent` 的使用者需要知道如何创建 `LoginComponent` 的实例。父组件必须在其接口中添加方法，确保使用者能够根据父组件的实例创建子组件的实例：

3. 提供在接口中创建 `LoginComponent` 实例的 factory：

   ```kotlin
   @Singleton
   @Component(modules = [NetworkModule::class, SubcomponentsModule::class])
   interface ApplicationComponent {
   // This function exposes the LoginComponent Factory out of the graph so consumers
   // can use it to obtain new instances of LoginComponent
   fun loginComponent(): LoginComponent.Factory
   }
   ```

### 3.5 为子组件分配作用域

* 如果您构建项目，可以创建 `ApplicationComponent` 和 `LoginComponent` 的实例。`ApplicationComponent` 会附加到应用的生命周期，因为只要应用在内存中，您就希望使用图的同一实例。

* `LoginComponent` 的生命周期是什么？需要 `LoginComponent` 的其中一项原因是，您需要在与登录相关的 Fragment 之间共享 `LoginViewModel` 的同一实例。此外，每当有新的登录流程时，您都希望使用不同的 `LoginViewModel` 实例。`LoginActivity` 表示 `LoginComponent` 的正确生命周期：对于每个新 Activity，您都需要 `LoginComponent` 的新实例以及可使用该 `LoginComponent` 实例的 Fragment。

* 由于 `LoginComponent` 会附加到 `LoginActivity` 生命周期，因此您必须在 Activity 中保留对该组件的引用，方式与在应用类中保留对 `applicationComponent` 的引用相同。这样，Fragment 就可以访问该组件了。

```kotlin
class LoginActivity: Activity() {
    // Reference to the Login graph
    lateinit var loginComponent: LoginComponent
    ...
}
```

* 请注意，由于您不希望变量 `loginComponent` 由 Dagger 提供，因此该变量不带 `@Inject` 注释。

* 您可以使用 `ApplicationComponent` 获取对 `LoginComponent` 的引用，然后注入 `LoginActivity`，如下所示：

```kotlin
class LoginActivity: Activity() {
    // Reference to the Login graph
    lateinit var loginComponent: LoginComponent

    // Fields that need to be injected by the login graph
    @Inject lateinit var loginViewModel: LoginViewModel

    override fun onCreate(savedInstanceState: Bundle?) {
        // Creation of the login graph using the application graph
        loginComponent = (applicationContext as MyDaggerApplication)
                            .appComponent.loginComponent().create()

        // Make Dagger instantiate @Inject fields in LoginActivity
        loginComponent.inject(this)

        // Now loginViewModel is available

        super.onCreate(savedInstanceState)
    }
}
```

* `LoginComponent` 是在 Activity 的 `onCreate()` 方法中创建的，将随着 Activity 的销毁而被隐式销毁。

* 每次请求时，`LoginComponent` 必须始终提供 `LoginViewModel` 的同一实例。您可以通过创建自定义注释作用域并使用该作用域为 `LoginComponent` 和 `LoginViewModel` 添加注释确保这一点。请注意，不可使用 `@Singleton` 注释，因为该注释已被父组件使用，并且使对象成为应用单例（整个应用中的唯一实例）。您需要创建不同的注释作用域。

* **注意**：作用域限定规则如下：如果某个类型标记有作用域注释，该类型就只能由带有相同作用域注释的组件使用。如果某个组件标记有作用域注释，该组件就只能提供带有该注释的类型或不带注释的类型。子组件不能使用其某一父组件使用的作用域注释。组件还涉及此上下文中的子组件。

* 在这种情况下，您可能已将此作用域命名为 `@LoginScope`，但这种做法并不理想。作用域注释的名称不应明确指明其实现目的。相反，作用域注释应根据其生命周期进行命名，因为注释可以由同级组件（如 `RegistrationComponent` 和 `SettingsComponent`）重复使用。因此，您应将其命名为 `@ActivityScope` 而不是 `@LoginScope`。

```kotlin
// Definition of a custom scope called ActivityScope
@Scope
@Retention(value = AnnotationRetention.RUNTIME)
annotation class ActivityScope

// Classes annotated with @ActivityScope are scoped to the graph and the same
// instance of that type is provided every time the type is requested.
@ActivityScope
@Subcomponent
interface LoginComponent { ... }

// A unique instance of LoginViewModel is provided in Components
// annotated with @ActivityScope
@ActivityScope
class LoginViewModel @Inject constructor(
    private val userRepository: UserRepository
) { ... }
```

* 现在，如果您有两个需要 `LoginViewModel` 的 Fragment，系统就会为它们提供同一实例。例如，如果您有 `LoginUsernameFragment` 和 `LoginPasswordFragment`，它们需要由 `LoginComponent` 注入：

```kotlin
@ActivityScope
@Subcomponent
interface LoginComponent {

    @Subcomponent.Factory
    interface Factory {
        fun create(): LoginComponent
    }

    // All LoginActivity, LoginUsernameFragment and LoginPasswordFragment
    // request injection from LoginComponent. The graph needs to satisfy
    // all the dependencies of the fields those classes are injecting
    fun inject(loginActivity: LoginActivity)
    fun inject(usernameFragment: LoginUsernameFragment)
    fun inject(passwordFragment: LoginPasswordFragment)
}
```

* 组件访问位于 `LoginActivity` 对象中的组件的实例。`LoginUserNameFragment` 的示例代码显示在以下代码段中：

```kotlin
class LoginUsernameFragment: Fragment() {

    // Fields that need to be injected by the login graph
    @Inject lateinit var loginViewModel: LoginViewModel

    override fun onAttach(context: Context) {
        super.onAttach(context)

        // Obtaining the login graph from LoginActivity and instantiate
        // the @Inject fields with objects from the graph
        (activity as LoginActivity).loginComponent.inject(this)

        // Now you can access loginViewModel here and onCreateView too
        // (shared instance with the Activity and the other Fragment)
    }
}
```

* 同样适用于 `LoginPasswordFragment`：

```kotlin
class LoginPasswordFragment: Fragment() {

    // Fields that need to be injected by the login graph
    @Inject lateinit var loginViewModel: LoginViewModel

    override fun onAttach(context: Context) {
        super.onAttach(context)

        (activity as LoginActivity).loginComponent.inject(this)

        // Now you can access loginViewModel here and onCreateView too
        // (shared instance with the Activity and the other Fragment)
    }
}
```

* 图 3 显示了新子组件的 Dagger 图。带有白色圆点的类（`UserRepository`、`LoginRetrofitService` 和 `LoginViewModel`）是唯一实例作用域限定为各自组件的类。

<img src="https://developer.android.google.cn/images/training/dependency-injection/4-graph-subcomponent.png?hl=zh_cn" alt="添加最后一个子组件后的应用图" style="zoom:67%;" />

* **图 3.** 您为 Android 应用示例构建的图的表示法

* 下面我们详细介绍该图的各个部分：

1. `NetworkModule`（以及由此产生的 `LoginRetrofitService`）包含在 `ApplicationComponent` 中，因为您在组件中指定了它。

2. `UserRepository` 保留在 `ApplicationComponent` 中，因为其作用域限定为 `ApplicationComponent`。如果项目扩大，您会希望跨不同功能（例如注册）共享同一实例。

   由于 `UserRepository` 是 `ApplicationComponent` 的一部分，其依赖项（即 `UserLocalDataSource` 和 `UserRemoteDataSource`）也必须位于此组件中，以便能够提供 `UserRepository` 的实例。

3. `LoginViewModel` 包含在 `LoginComponent` 中，因为只有 `LoginComponent` 注入的类才需要它。`LoginViewModel` 未包含在 `ApplicationComponent` 中，因为 `ApplicationComponent` 中的任何依赖项都不需要 `LoginViewModel`。

   同样，如果您尚未将 `UserRepository` 的作用域限定为 `ApplicationComponent`，Dagger 会自动将 `UserRepository` 及其依赖项作为 `LoginComponent` 的一部分包含在内，因为这是目前使用 `UserRepository` 的唯一位置。

* 除了将对象作用域限定为不同的生命周期之外，**创建子组件是分别封装应用的不同部分的良好做法**。

* 根据应用流程构建应用以创建不同的 Dagger 子图有助于在内存和启动时间方面实现**性能和扩容性更强的应用**。

* **注意**：如果您需要使容器在出现设备旋转等配置更改后继续存在，请遵循[保存界面状态指南](https://developer.android.google.cn/topic/libraries/architecture/saving-states?hl=zh_cn)。您可能需要采用与处理进程终止相同的方式处理配置更改；否则，您的应用可能会在低端设备上丢失状态。

## 四、构建 Dagger 图的最佳做法

* 为应用构建 Dagger 图时：
  * 创建组件时，您应该考虑什么元素会决定该组件的生命周期。在本示例中，应用类负责 `ApplicationComponent`，而 `LoginActivity` 负责 `LoginComponent`。
  * 请仅在必要时使用作用域限定。过度使用作用域限定可能会对应用的运行时性能产生负面影响：只要组件在内存中，对象就会在内存中；获取限定作用域的对象的成本更高。当 Dagger 提供对象时，它使用 `DoubleCheck` 锁定，而不是 factory 类型提供程序。

## 五、测试使用 Dagger 的项目

* 使用 Dagger 之类的依赖项注入框架的一个好处是，它可以让您更轻松地测试代码。

### 5.1 单元测试

* 您不必使用 Dagger 进行单元测试。测试使用构造函数注入的类时，您无需使用 Dagger 实例化该类。您可以直接调用其构造函数，传入虚假或模拟依赖项，就像它们不带注释时所做的一样。

* 例如，在测试 `LoginViewModel` 时：

```kotlin
@ActivityScope
class LoginViewModel @Inject constructor(
    private val userRepository: UserRepository
) { ... }

class LoginViewModelTest {

    @Test
    fun `Happy path`() {
        // You don't need Dagger to create an instance of LoginViewModel
        // You can pass a fake or mock UserRepository
        val viewModel = LoginViewModel(fakeUserRepository)
        assertEquals(...)
    }
}
```

### 5.2 端到端测试

* 对于集成测试，最佳做法是创建用于测试的 `TestApplicationComponent`。正式版和测试版使用不同的组件配置。

* 这需要在应用中进行更多前期[模块设计](https://dagger.dev/testing.html#organize-modules-for-testability)。测试组件扩展了正式版组件并安装了一组不同的模块。

```kotlin
// TestApplicationComponent extends from ApplicationComponent to have them both
// with the same interface methods. You need to include the modules of the
// component here as well, and you can replace the ones you want to override.
// This sample uses FakeNetworkModule instead of NetworkModule
@Singleton
@Component(modules = [FakeNetworkModule::class, SubcomponentsModule::class])
interface TestApplicationComponent : ApplicationComponent {
}
```

* `FakeNetworkModule` 包含原始 `NetworkModule` 的虚假实现。您可以通过它提供任何想要替换的虚假实例或模拟。

```kotlin
// In the FakeNetworkModule, pass a fake implementation of LoginRetrofitService
// that you can use in your tests.
@Module
class FakeNetworkModule {
    @Provides
    fun provideLoginRetrofitService(): LoginRetrofitService {
        return FakeLoginService()
    }
}
```

* 在集成或端到端测试中，您应使用创建 `TestApplicationComponent`（而非 `ApplicationComponent`）的 `TestApplication`。

```kotlin
// Your test application needs an instance of the test graph
class MyTestApplication: MyApplication() {
    override val appComponent = DaggerTestApplicationComponent.create()
}
```

* 然后，此测试应用会用于自定义 `TestRunner`，您将使用它来运行插桩测试。如需了解详情，请参阅[“在 Android 应用中使用 Dagger”Codelab](https://codelabs.developers.google.com/codelabs/android-dagger/?hl=zh_cn)。

## 六、用 Dagger 模块

* Dagger 模块是一种以语义方式封装如何提供对象的方法。您可以在组件中添加模块，但也可以在其他模块内添加模块。此功能非常强大，但很容易被误用。

* 将模块添加到组件或其他模块后，该模块就已经在 Dagger 图中；Dagger 可以在该组件中提供这些对象。添加模块之前，请通过以下方法检查该模块是否已成为 Dagger 图的一部分：检查该模块是否已添加到组件中；或者编译项目，然后检查 Dagger 能否找到该模块所需的依赖项。

* 良好做法是模块只在组件中声明一次，特定高级 Dagger 用例除外。

* 假设图的配置方式如下。`ApplicationComponent` 包括 `Module1` 和 `Module2`，`Module1` 包括 `ModuleX`。

```kotlin
@Component(modules = [Module1::class, Module2::class])
interface ApplicationComponent { ... }

@Module(includes = [ModuleX::class])
class Module1 { ... }

@Module
class Module2 { ... }
```

* 如果 `Module2` 现在依赖于 `ModuleX` 提供的类。**错误做法**是将 `ModuleX` 包含在 `Module2` 中，因为这样 `ModuleX` 在图中就出现了两次，如以下代码段所示：

```kotlin
// Bad practice: ModuleX is declared multiple times in this Dagger graph
@Component(modules = [Module1::class, Module2::class])
interface ApplicationComponent { ... }

@Module(includes = [ModuleX::class])
class Module1 { ... }

@Module(includes = [ModuleX::class])
class Module2 { ... }
```

* 您应改为执行以下某项操作：

1. 重构模块，并将共同模块提取到组件中。
2. 使用两个模块共享的对象创建一个新模块，并将其提取到组件中。

* 如果不以这种方式进行重构，就会导致许多模块相互包含而没有清晰的结构，并且更难以了解每个依赖项的来源。

* **良好做法（选项 1）**：在 Dagger 图中声明一次 ModuleX。

```kotlin
@Component(modules = [Module1::class, Module2::class, ModuleX::class])
interface ApplicationComponent { ... }

@Module
class Module1 { ... }

@Module
class Module2 { ... }
```

* **良好做法（选项 2）**：将 `ModuleX` 中 `Module1` 和 `Module2` 的共同依赖项提取到包含在该组件中的名为 `ModuleXCommon` 的新模块。然后，使用特定于每个模块的依赖项创建名为 `ModuleXWithModule1Dependencies` 和 `ModuleXWithModule2Dependencies` 的另外两个模块。所有模块在 Dagger 图中都只声明一次。

```kotlin
@Component(modules = [Module1::class, Module2::class, ModuleXCommon::class])
interface ApplicationComponent { ... }

@Module
class ModuleXCommon { ... }

@Module
class ModuleXWithModule1SpecificDependencies { ... }

@Module
class ModuleXWithModule2SpecificDependencies { ... }

@Module(includes = [ModuleXWithModule1SpecificDependencies::class])
class Module1 { ... }

@Module(includes = [ModuleXWithModule2SpecificDependencies::class])
class Module2 { ... }
```

## 七、总结

* 查看[最佳做法部分](https://developer.android.google.cn/training/dependency-injection/dagger-android?hl=zh_cn#best-practices)（如果您尚未查看）。如需了解如何在 Android 应用中使用 Dagger，请参阅[“在 Android 应用中使用 Dagger”Codelab](https://codelabs.developers.google.com/codelabs/android-dagger/?hl=zh_cn)。