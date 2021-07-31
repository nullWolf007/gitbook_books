[TOC]

## 在多模块应用中使用 Dagger

* **注意**：本文中提到的模块是指 Gradle 模块，而不是 Dagger 模块。

* 包含多个 Gradle 模块的项目称为多模块项目。在作为单个 APK 发布且不包含动态功能模块的多模块项目中，通常具有一个可依赖于项目中大多数模块的 `app` 模块和一个其他模块通常可依赖的 `base` 或 `core` 模块。`app` 模块通常包含 `Application` 类，而 `base` 模块包含在项目中所有模块之间共享的所有通用类。

* `app` 模块非常适合用来声明应用组件（例如，下图中的 `ApplicationComponent`），应用组件可以提供其他组件可能需要的对象以及应用的单例。例如，像 `OkHttpClient` 这样的类、JSON 解析器、数据库的访问函数或可能在 `core` 模块中定义的 `SharedPreferences` 对象，都将由 `app` 模块中定义的 `ApplicationComponent` 提供。

* 在 `app` 模块中，还可以包含一些生命周期较短的其他组件。例如，在用户登录后，该模块中可能会包含具有用户专属配置的 `UserComponent`（比如 `UserSession`）。

* 在项目的不同模块中，您可以定义至少一个具有该模块的专属逻辑的子组件，如图 1 所示。

<img src="https://developer.android.google.cn/images/training/dependency-injection/5-graph-modules.png?hl=zh_cn" alt="img" style="zoom:80%;" />

* **图 1.** 多模块项目中的 Dagger 图示例

* 例如，在 `login` 模块中，您可以定义采用自定义 `@ModuleScope` 注释来限定范围的 `LoginComponent` 组件，该组件可提供相应功能（例如 `LoginRepository`）的常用对象。在该模块内，您还可以定义依赖于具有其他自定义范围的 `LoginComponent` 的其他组件，例如将 `@FeatureScope` 用于 `LoginActivityComponent` 或 `TermsAndConditionsComponent`，您可以在这两个组件中定义更加特定于功能的逻辑，例如 `ViewModel` 对象。

* 对于其他模块（例如 `Registration`），您需要进行类似的设置。

* 多模块项目的一般规则是同级模块不应相互依赖。如果它们相互依赖，请考虑相应共享逻辑（它们之间的依赖关系）是否应该是父级模块的一部分。如果应该，请进行重构以将这些类移动到父级模块；如果不应该，请创建一个可扩展父级模块的新模块，并使两个原始模块都扩展新模块。

* 最佳做法：在以下情况下，您通常需要在模块中创建一个组件：
  * 对于 `LoginActivityComponent`，您需要执行字段注入。
  * 对于 `LoginComponent`，您需要限定对象的范围。

* 如果并非以上两种情况并且您需要告知 Dagger 如何从相应模块提供对象，请使用 `@Provides` 或 `@Binds` 方法创建和提供 Dagger 模块，但前提是这些类不支持构造函数注入。

## 一、使用 Dagger 子组件进行实现

* [在 Android 应用中使用 Dagger](https://developer.android.google.cn/training/dependency-injection/dagger-android?hl=zh_cn#dagger-subcomponents) 文档页面介绍了如何创建和使用子组件。但是，由于功能模块无法识别 `app` 模块，因此您无法使用相同的代码。例如，如果您考虑使用我们在上一页中介绍的代码来构建典型的登录流程，系统将无法再进行编译：

```kotlin
    class LoginActivity: Activity() {
      ...

      override fun onCreate(savedInstanceState: Bundle?) {
        // Creation of the login graph using the application graph
        loginComponent = (applicationContext as MyDaggerApplication)
                            .appComponent.loginComponent().create()

        // Make Dagger instantiate @Inject fields in LoginActivity
        loginComponent.inject(this)
        ...
      }
    }
    
```

* 原因在于 `login` 模块无法识别 `MyApplication` 和 `appComponent`。为了使其正常运行，您需要在功能模块中定义一个接口，该接口提供 `MyApplication` 需要实现的 `FeatureComponent`。

* 在以下示例中，您可以定义一个 `LoginComponentProvider` 接口，该接口在 `login` 模块中为登录流程提供 `LoginComponent`：

```kotlin
    interface LoginComponentProvider {
        fun provideLoginComponent(): LoginComponent
    }
    
```

* 现在，`LoginActivity` 将使用该接口，而不是上面定义的代码段：

```kotlin
    class LoginActivity: Activity() {
      ...

      override fun onCreate(savedInstanceState: Bundle?) {
        loginComponent = (applicationContext as LoginComponentProvider)
                            .provideLoginComponent()

        loginComponent.inject(this)
        ...
      }
    }
    
```

* 现在，`MyApplication` 需要实现该接口以及所需的方法：

```kotlin
    class MyApplication: Application(), LoginComponentProvider {
      // Reference to the application graph that is used across the whole app
      val appComponent = DaggerApplicationComponent.create()

      override fun provideLoginComponent(): LoginComponent {
        return appComponent.loginComponent().create()
      }
    }

    
```

* 这就是在多模块项目中使用 Dagger 子组件的方法。项目中包含动态功能模块时，解决方案因模块之间相互依赖的方式而异。

## 二、包含动态功能模块时的组件依赖关系

* 包含[动态功能模块](https://developer.android.google.cn/studio/projects/dynamic-delivery?hl=zh_cn#customize_delivery)时，模块之间相互依赖的方式通常是相反的。不是 `app` 模块包含功能模块，而是动态功能模块依赖于 `app` 模块。如需了解这些模块之间的依赖关系，请参见图 2。

<img src="https://developer.android.google.cn/images/training/dependency-injection/5-graph-dynamic-modules.png?hl=zh_cn" alt="img" style="zoom:80%;" />

* **图 2.** 包含动态功能模块的项目中的 Dagger 图示例

* 在 Dagger 中，组件需要能够识别各自的子组件。此类信息包含在添加到父级组件的 Dagger 模块中（例如 [在 Android 应用中使用 Dagger](https://developer.android.google.cn/training/dependency-injection/dagger-android?hl=zh_cn#dagger-subcomponents) 中所述的 `SubcomponentsModule` 模块）。

* 遗憾的是，应用和动态功能模块之间依赖关系的反转使得子组件不会在 `app` 模块中显示，因为它不在构建路径中。例如，`login` 动态功能模块中定义的 `LoginComponent` 不能是 `app` 模块中定义的 `ApplicationComponent` 的子组件。

* 您可以使用 Dagger 的**组件依赖关系**机制来解决此问题。在此机制中，子级组件不是父级组件的子组件，而是依赖于父级组件。因此，这里不存在父级/子级关系；**组件**现在会依赖于其他内容来获取特定的**依赖项**。组件需要从图表中提供类型，以供依赖组件使用。

* **注意**：只要您需要创建 `ApplicationComponent` 的子组件，便会出现此问题。如果您需要创建依赖于动态功能模块的常规 Gradle 模块，并且需要创建依赖于该动态功能模块中定义的组件的组件，可以照常使用子组件。

* 例如，名为 `login` 的动态功能模块需要构建一个依赖于 `app` Gradle 模块中提供的 `AppComponent` 的 `LoginComponent`。

* 以下是作为 `app` Gradle 模块一部分的类和 `AppComponent` 的定义：

```kotlin
    // UserRepository's dependencies
    class UserLocalDataSource @Inject constructor() { ... }
    class UserRemoteDataSource @Inject constructor() { ... }

    // UserRepository is scoped to AppComponent
    @Singleton
    class UserRepository @Inject constructor(
        private val localDataSource: UserLocalDataSource,
        private val remoteDataSource: UserRemoteDataSource
    ) { ... }

    @Singleton
    @Component
    interface AppComponent { ... }
    
```

* 在包含 `app` Gradle 模块的 `login` Gradle 模块中，有一个需要注入 `LoginViewModel` 实例的 `LoginActivity`：

```kotlin
    // LoginViewModel depends on UserRepository that is scoped to AppComponent
    class LoginViewModel @Inject constructor(
        private val userRepository: UserRepository
    ) { ... }
    
```

* `LoginViewModel` 依赖于可用且范围限定到 `AppComponent` 的 `UserRepository`。我们创建一个依赖于 `AppComponent` 注入 `LoginActivity` 的 `LoginComponent`：

```kotlin
    // Use the dependencies attribute in the Component annotation to specify the
    // dependencies of this Component
    @Component(dependencies = [AppComponent::class])
    interface LoginComponent {
        fun inject(activity: LoginActivity)
    }
    
```

* `LoginComponent` 通过将 `AppComponent` 添加到组件注释的 dependencies 参数中来指定对 AppComponent 的依赖。由于 `LoginActivity` 将由 Dagger 注入，因此应将 `inject()` 方法添加到接口。

* **注意**：`LoginComponent` 是使用 `@Component` 而非 `@Subcomponent` 注释的，操作方法与[在 Android 应用中使用 Dagger](https://developer.android.google.cn/training/dependency-injection/dagger-android?hl=zh_cn#dagger-subcomponents) 页面中介绍的方法相同。

* 在创建 `LoginComponent` 时，需要传入 `AppComponent` 的实例。请使用组件 factory 执行此操作：

```kotlin
    @Component(dependencies = [AppComponent::class])
    interface LoginComponent {

        @Component.Factory
        interface Factory {
            // Takes an instance of AppComponent when creating
            // an instance of LoginComponent
            fun create(appComponent: AppComponent): LoginComponent
        }

        fun inject(activity: LoginActivity)
    }
    
```

* 现在，`LoginActivity` 即可创建 `LoginComponent` 的实例并调用 `inject()` 方法。

```kotlin
    class LoginActivity: Activity() {

        // You want Dagger to provide an instance of LoginViewModel from the Login graph
        @Inject lateinit var loginViewModel: LoginViewModel

        override fun onCreate(savedInstanceState: Bundle?) {
            // Gets appComponent from MyApplication available in the base Gradle module
            val appComponent = (applicationContext as MyApplication).appComponent

            // Creates a new instance of LoginComponent
            // Injects the component to populate the @Inject fields
            DaggerLoginComponent.factory().create(appComponent).inject(this)

            super.onCreate(savedInstanceState)

            // Now you can access loginViewModel
        }
    }
    
```

* `LoginViewModel` 依赖于 `UserRepository`；为了让 `LoginComponent` 能够从 `AppComponent` 访问 LoginViewModel，`AppComponent` 需要在其接口中提供 LoginViewModel。

```kotlin
    @Singleton
    @Component
    interface AppComponent {
        fun userRepository(): UserRepository
    }
    
```

* 限定范围规则对依赖组件和子组件的作用相同。由于 `LoginComponent` 使用 `AppComponent` 的实例，因此它们不能使用相同的范围注释。

* 如果您希望将 `LoginViewModel` 的范围限定到 `LoginComponent`，可以按照之前使用自定义 `@ActivityScope` 注释的方法进行操作。

```kotlin
    @ActivityScope
    @Component(dependencies = [AppComponent::class])
    interface LoginComponent { ... }

    @ActivityScope
    class LoginViewModel @Inject constructor(
        private val userRepository: UserRepository
    ) { ... }
    
```

* **注意**：不在组件中提供依赖组件所需的所有类型将会导致 Dagger 在编译时发生错误，这是因为它无法为依赖组件提供某些类型。

## 三、最佳做法

- `ApplicationComponent` 应始终位于 `app` 模块中。
- 如果您需要在某个模块中执行字段注入，或者需要为应用的特定流程限定对象的范围，请在该模块中创建 Dagger 组件。
- 对于要用作实用工具或帮助程序且无需构建图表（这就是您需要 Dagger 组件的原因）的 Gradle 模块，请使用不支持构造函数注入的类的 @Provides 和 @Binds 方法来创建和提供公共 Dagger 模块。
- 若要在包含动态功能模块的 Android 应用中使用 Dagger，请使用能够访问 `app` 模块中定义的 `ApplicationComponent` 所提供的依赖项的组件依赖项。