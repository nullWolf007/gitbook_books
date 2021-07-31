[TOC]

## 在多模块应用中使用 Hilt

* Hilt 代码生成操作需要访问使用 Hilt 的所有 Gradle 模块。编译 `Application` 类的 Gradle 模块需要在其传递依赖项中包含所有 Hilt 模块和通过构造函数注入的类。

* 如果多模块项目由常规 Gradle 模块组成，则您可以按照[使用 Hilt 实现依赖项注入](https://developer.android.google.cn/training/dependency-injection/hilt-android?hl=zh_cn)中的说明使用 Hilt。不过，对于包含[动态功能模块](https://developer.android.google.cn/guide/app-bundle/dynamic-delivery?hl=zh_cn#customize_delivery) (DFM) 的应用，并不是这样。

### 一、在动态功能模块中使用 Hilt

* 在 DFM 中，通常模块之间相互依赖的方式颠倒过来。因此，Hilt 无法在动态功能模块中处理注释。您必须在 DFM 中使用 [Dagger](https://developer.android.google.cn/training/dependency-injection/dagger-basics?hl=zh_cn) 执行依赖项注入。

* 您必须使用组件依赖关系来解决 DFM 的这一问题。请按以下步骤操作：

1. 在具有 DFM 所需依赖项的 `app` 模块（或可由 Hilt 处理的其他任何模块）中声明一个 [`@EntryPoint` 接口](https://developer.android.google.cn/training/dependency-injection/hilt-android?hl=zh_cn#not-supported)。
2. 创建一个依赖于 `@EntryPoint` 接口的 Dagger 组件。
3. 在 DFM 中照常使用 Dagger。

* 下面考虑一下[使用 Hilt 实现依赖项注入](https://developer.android.google.cn/training/dependency-injection/hilt-android?hl=zh_cn)页面中的示例。假设您向项目中添加了一个 `login` 动态功能模块。您使用一个名为 `LoginActivity` 的 Activity 实现登录功能。这意味着，您只能从应用组件获取绑定。

* 对于此功能，您需要具有 `authInterceptor` 绑定的 `OkHttpClient`。

* 首先，创建一个安装在具有 `login` 模块所需绑定的 `ApplicationComponent` 中的 `@EntryPoint` 接口：

```kotlin
// LoginModuleDependencies.kt - File in the app module.

@EntryPoint
@InstallIn(ApplicationComponent::class)
interface LoginModuleDependencies {

  @AuthInterceptorOkHttpClient
  fun okHttpClient(): OkHttpClient
}
```

* 为了在 `LoginActivity` 中执行字段注入，创建一个依赖于 `@EntryPoint` 接口的 Dagger 组件：

```kotlin
// LoginComponent.kt - File in the login module.

@Component(dependencies = [LoginModuleDependencies::class])
interface LoginComponent {

  fun inject(activity: LoginActivity)

  @Component.Builder
  interface Builder {
    fun context(@BindsInstance context: Context): Builder
    fun appDependencies(loginModuleDependencies: LoginModuleDependencies): Builder
    fun build(): LoginComponent
  }
}
```

* 完成上述步骤后，在 DFM 中照常使用 Dagger。例如，您可以将来自 `ApplicationComponent` 的绑定用作类的依赖项：

```kotlin
// LoginAnalyticsAdapter.kt - File in the login module.

class LoginAnalyticsAdapter @Inject constructor(
  @AuthInterceptorOkHttpClient okHttpClient: OkHttpClient
) { ... }
```

* 为了执行字段注入，使用 `applicationContext` 创建 Dagger 组件的实例以获取 `ApplicationComponent` 依赖项：

```kotlin
// LoginActivity.kt - File in the login module.

class LoginActivity : AppCompatActivity() {

  @Inject
  lateinit var loginAnalyticsAdapter: LoginAnalyticsAdapter

  override fun onCreate(savedInstanceState: Bundle?) {
    DaggerLoginComponent.builder()
        .context(this)
        .appDependencies(
          EntryPointsAccessors.fromApplication(
            applicationContext,
            LoginModuleDependencies::class.java
          )
        )
        .build()
        .inject(this)

    super.onCreate(savedInstanceState)
    ...
  }
}
```

* 如需详细了解动态功能模块中的模块依赖关系，请参阅[包含动态功能模块时的组件依赖关系](https://developer.android.google.cn/training/dependency-injection/dagger-multi-module?hl=zh_cn#dagger-dfm)。

* 如需详细了解 Dagger 在 Android 上的应用，请参阅[在 Android 应用中使用 Dagger](https://developer.android.google.cn/training/dependency-injection/dagger-android?hl=zh_cn)。