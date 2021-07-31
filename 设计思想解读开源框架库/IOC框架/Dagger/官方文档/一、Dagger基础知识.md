[TOC]

## Dagger 基础知识

* Android 应用中的手动依赖项注入或服务定位器可能会出现问题，具体取决于项目的大小。您可以使用 [Dagger](https://dagger.dev/) 来管理依赖项，从而在项目规模扩大时限制项目的复杂性。

* Dagger 会自动生成代码，该代码与您原本需要手动编写的代码相似。由于该代码是在编译时生成的，因此具有可追溯性，而且性能要高于其他基于反射的解决方案（如 [Guice](https://en.wikipedia.org/wiki/Google_Guice)）。

* **注意**：使用 [Hilt](https://developer.android.google.cn/training/dependency-injection/hilt-android?hl=zh_cn) 可在 Android 上实现依赖项注入。Hilt 在 Dagger 的基础上构建而成，提供了一种将 Dagger 依赖项注入纳入 Android 应用的标准方法。

### 一、使用 Dagger 的优势

* Dagger 可以执行以下操作，使您无需再编写冗长乏味又容易出错的样板代码：
  * 生成您在手动 DI 部分手动实现的 `AppContainer` 代码（应用图）。
  * 为应用图中提供的类创建 factory。这就是在内部满足依赖关系的方式。
  * 重复使用依赖项或创建类型的新实例，具体取决于您如何使用作用域配置该类型。
  * 为特定流程创建容器，操作方法与上一部分中使用 Dagger 子组件为登录流程创建容器的方法相同。这样可以释放内存中不再需要的对象，从而提升应用性能。

* 只要您声明类的依赖项并指定如何使用注释满足它们的依赖关系，Dagger 便会在构建时自动执行以上所有操作。Dagger 生成的代码与您手动编写的代码类似。在内部，Dagger 会创建一个对象图，然后它可以参考该图来找到提供类实例的方式。对于图中的每个类，Dagger 都会生成一个 [factory 类型](https://en.wikipedia.org/wiki/Factory_method_pattern)类，它会使用该类在内部获取该类型的实例。

* 在构建时，Dagger 会走查您的代码，并执行以下操作：
  * 构建并验证依赖关系图，确保：
    - 每个对象的依赖关系都可以得到满足，从而避免出现运行时异常。
    - 不存在任何依赖循环，从而避免出现无限循环。
  - 生成在运行时用于创建实际对象及其依赖项的类。

### 二、Dagger 中的一个简单用例：生成 factory

* 为了演示如何使用 Dagger，我们为 `UserRepository` 类创建一个简单的 [factory](https://en.wikipedia.org/wiki/Factory_method_pattern)，如下图所示：

<img src="https://developer.android.google.cn/images/training/dependency-injection/3-factory-diagram.png?hl=zh_cn" alt="img" style="zoom:67%;" />

* 按以下方式定义 `UserRepository`：

```kotlin
class UserRepository(
    private val localDataSource: UserLocalDataSource,
    private val remoteDataSource: UserRemoteDataSource
) { ... }
```

* 向 `UserRepository` 构造函数添加 `@Inject` 注释，以便告知 Dagger 如何创建 `UserRepository`：

```kotlin
// @Inject lets Dagger know how to create instances of this object
class UserRepository @Inject constructor(
    private val localDataSource: UserLocalDataSource,
    private val remoteDataSource: UserRemoteDataSource
) { ... }
```

* 在上面的代码段中，您告知 Dagger：

1. 如何使用带有 `@Inject` 注释的构造函数创建 `UserRepository` 实例。
2. 它的依赖项为：`UserLocalDataSource` 和 `UserRemoteDataSource`。

* 现在，Dagger 已经知道了如何创建 `UserRepository` 的实例，但不知道如何创建其依赖项。如果您也为其他类添加了注释，Dagger 便会知道如何创建它们：

```kotlin
// @Inject lets Dagger know how to create instances of these objects
class UserLocalDataSource @Inject constructor() { ... }
class UserRemoteDataSource @Inject constructor() { ... }
```

### 三、Dagger 组件

* Dagger 可以在您的项目中创建一个依赖关系图，然后它可以从该图中了解在需要这些依赖项时从何处获取它们。为了让 Dagger 执行此操作，您需要创建一个接口，并使用 `@Component` 为其添加注释。Dagger 会创建一个容器，就像您在手动注入依赖项时执行的操作一样。

* 在 `@Component` 接口内，您可以定义返回所需类（即 `UserRepository`）的实例的函数。`@Component` 会让 Dagger 生成一个容器，其中应包含满足其提供的类型所需的所有依赖项。这称为 Dagger 组件；它包含一个图，其中包括 Dagger 知道如何提供的对象及其各自的依赖项。

```kotlin
// @Component makes Dagger create a graph of dependencies
@Component
interface ApplicationGraph {
    // The return type  of functions inside the component interface is
    // what can be provided from the container
    fun repository(): UserRepository
}
```

* 在您构建项目时，Dagger 会为您生成 `ApplicationGraph` 接口的实现：`DaggerApplicationGraph`。Dagger 通过其注释处理器创建了一个依赖关系图，其中包含三个类（`UserRepository`、`UserLocalDatasource` 和 `UserRemoteDataSource`）之间的关系，并且只有一个入口点：用于获取 `UserRepository` 实例。您可以按以下方式使用：

```kotlin
// Create an instance of the application graph
val applicationGraph: ApplicationGraph = DaggerApplicationGraph.create()
// Grab an instance of UserRepository from the application graph
val userRepository: UserRepository = applicationGraph.repository()
```

* Dagger 在每次收到请求时都会创建 `UserRepository` 的新实例。

```kotlin
val applicationGraph: ApplicationGraph = DaggerApplicationGraph.create()

val userRepository: UserRepository = applicationGraph.repository()
val userRepository2: UserRepository = applicationGraph.repository()

assert(userRepository != userRepository2)
```

* 有时，您需要在容器中包含某个依赖项的唯一实例。原因如下：

1. 您希望以此类型作为依赖项的其他类型共享同一实例，例如登录流程中使用同一 `LoginUserData` 的多个 `ViewModel` 对象。
2. 对象的创建成本很高，因此您不希望在每次将其声明为依赖项时都创建新实例（例如，JSON 解析器）。

* 在本例中，您可能希望在图中包含 `UserRepository` 的唯一实例，以便每次请求 `UserRepository` 时，都能获得同一实例。这在您的示例中非常有用，因为在具有更复杂应用图的真实应用中，您可能会具有多个依赖于 `UserRepository` 的 `ViewModel` 对象，而且您不希望在每次需要提供 `UserRepository` 时都创建 `UserLocalDataSource` 和 `UserRemoteDataSource` 的新实例。

* 在手动注入依赖项时，您可以通过将 `UserRepository` 的同一实例传递给 ViewModel 类的构造函数来实现此目的；但在 Dagger 中，由于您不用手动编写该代码，因此您必须告知 Dagger 您希望使用同一实例。这可以通过作用域注释来实现。

#### 3.1 Dagger 中的作用域限定

* 您可以使用作用域注释将某个对象的生命周期限定为其组件的生命周期。这意味着，每次需要提供该类型时，都会使用依赖项的同一实例。

* 为了在请求 `ApplicationGraph` 中的代码库时获得 `UserRepository` 的唯一实例，应针对 `@Component` 接口和 `UserRepository` 使用同一作用域注释。您可以使用 Dagger 使用的 `javax.inject` 软件包随附的 `@Singleton` 注释：

```kotlin
// Scope annotations on a @Component interface informs Dagger that classes annotated
// with this annotation (i.e. @Singleton) are bound to the life of the graph and so
// the same instance of that type is provided every time the type is requested.
@Singleton
@Component
interface ApplicationGraph {
    fun repository(): UserRepository
}

// Scope this class to a component using @Singleton scope (i.e. ApplicationGraph)
@Singleton
class UserRepository @Inject constructor(
    private val localDataSource: UserLocalDataSource,
    private val remoteDataSource: UserRemoteDataSource
) { ... }
```

* 或者，您也可以创建并使用自定义作用域注释。您可以按如下方式创建作用域注释：

```kotlin
// Creates MyCustomScope
@Scope
@MustBeDocumented
@Retention(value = AnnotationRetention.RUNTIME)
annotation class MyCustomScope
```

* 然后，您可以像往常那样使用它：

```kotlin
@MyCustomScope
@Component
interface ApplicationGraph {
    fun repository(): UserRepository
}

@MyCustomScope
class UserRepository @Inject constructor(
    private val localDataSource: UserLocalDataSource,
    private val service: UserService
) { ... }
```

* 在这两种情况下，为对象提供了同一作用域，该作用域用于为 `@Component` 接口添加注释。因此，您每次调用 `applicationGraph.repository()` 时，都会获得 `UserRepository` 的同一实例。

```kotlin
val applicationGraph: ApplicationGraph = DaggerApplicationGraph.create()

val userRepository: UserRepository = applicationGraph.repository()
val userRepository2: UserRepository = applicationGraph.repository()

assert(userRepository == userRepository2)
```

### 四、总结

* 务必先了解 Dagger 的优势及其工作原理的基础知识，然后才能在更复杂的场景中使用它。

* 在[下一页](https://developer.android.google.cn/training/dependency-injection/dagger-android?hl=zh_cn)中，您将学习如何将 Dagger 添加到 Android 应用中。