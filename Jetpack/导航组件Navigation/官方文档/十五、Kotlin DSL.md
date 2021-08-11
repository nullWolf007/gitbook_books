[TOC]

# 使用 Kotlin DSL 以编程方式构建图形

* 导航组件提供一种基于 Kotlin 的领域特定语言 (DSL)，该语言依赖于 Kotlin 的[类型安全构建器](https://kotlinlang.org/docs/reference/type-safe-builders.html)。借助该 API，您可以在 Kotlin 代码中（而不是在 XML 资源内部）以声明方式构建图形。如果您希望为应用动态构建导航，该方法会非常有用。例如，您的应用可以从外部网络服务下载并缓存导航配置，然后使用该配置在 Activity 的 `onCreate()` 函数中动态构建导航图。

## 一、依赖项

* 如需使用 Kotlin DSL，请将以下依赖项添加到应用的 `build.gradle` 文件中：

```groovy
dependencies {
    def nav_version = "2.3.5"

    api "androidx.navigation:navigation-fragment-ktx:$nav_version"
}
```

## 二、构建图形

* 首先我们来看一个基于 [Sunflower 应用](https://github.com/android/sunflower)的基本示例。在该示例中，我们有两个目的地：`home` 和 `plant_detail`。用户首次启动应用时，系统会显示 `home` 目的地。该目标地会显示用户花园中的植物列表。当用户选择其中一种植物时，应用会导航到 `plant_detail` 目的地。

* 图 1 显示了这些目的地以及 `plant_detail` 目的地所需的参数和应用用于从 `home` 导航到 `plant_detail` 的操作 `to_plant_detail`。

![Sunflower 应用有两个目的地以及一个连接这两个目的地的操作。](https://developer.android.google.cn/images/guide/navigation/navigation-kotlin-dsl-1.png)* **图 1.** Sunflower 应用有两个目的地（`home` 和 `plant_detail`)以及一个连接这两个目的地的操作。

### 2.1 为 Kotlin DSL 导航图创建主机

* 无论如何构建图形，都需要将图托管在 [`NavHost`](https://developer.android.google.cn/reference/androidx/navigation/NavHost) 中。Sunflower 使用 Fragment，因此我们在 [`FragmentContainerView`](https://developer.android.google.cn/reference/androidx/fragment/app/FragmentContainerView) 中使用 [`NavHostFragment`](https://developer.android.google.cn/reference/androidx/navigation/fragment/NavHostFragment)，如以下示例所示：

```xml
<!-- activity_garden.xml -->
<FrameLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto">
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <androidx.fragment.app.FragmentContainerView
        android:id="@+id/nav_host"
        android:name="androidx.navigation.fragment.NavHostFragment"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:defaultNavHost="true" />

</FrameLayout>
```

* 请注意，该示例中未设置 `app:navGraph` 属性，因为该图形是用编程方式构建的，而不是以 [XML 资源](https://developer.android.google.cn/guide/topics/resources/providing-resources)的形式定义的。

### 2.2 为图形创建常量

* 使用[基于 XML 的导航图](https://developer.android.google.cn/guide/navigation/navigation-getting-started#create-nav-graph)时，Android 构建流程会解析图形资源文件，并为图形中定义的每个 `id` 属性指定数字常量。您代码中的这些常量可以通过生成的资源类 [`R.id`](https://developer.android.google.cn/reference/android/R) 来获取。

* 例如，以下 XML 图代码段使用 `id`、`home` 声明了一个 Fragment 目的地：

```xml
<navigation ...>
   <fragment android:id="@+id/home" ... />
   ...
</navigation>
```

* 构建流程会创建一个与该目的地关联的常量值 `R.id.home`,然后，您可以使用该常量值从代码中引用该目的地。

* 使用 Kotlin DSL 以编程方式构建图形时，不会发生这种解析和生成常量的过程。相反，您必须为具有 `id` 值的每个目的地、操作和参数定义自己的常量。每个 ID 在配置更改中必须是唯一且一致的。

* 创建常量的一种有序方法是创建一组用于静态定义常量的嵌套 Kotlin `object`，如以下示例所示：

```kotlin
object nav_graph {

    const val id = 1 // graph id

    object dest {
        const val home = 2
        const val plant_detail = 3
    }

    object action {
        const val to_plant_detail = 4
    }

    object args {
        const val plant_id = "plantId"
    }
}
```

* 使用该结构，可以将对象调用串联起来，以获取代码中的 ID 值，如以下示例所示：

```kotlin
nav_graph.id                     // graph id
nav_graph.dest.home              // home destination id
nav_graph.action.to_plant_detail // action home -> plant_detail id
nav_graph.args.plant_id          // destination argument name
```

* **注意**：此示例在使用 `snake_casing` 作为名称时打破了 Kotlin 类命名惯例。在这种情况下，这些 ID 旨在模拟由 Android 资源系统生成的 ID。您可以使用任意名称，只要 ID 值在所有配置中都是唯一且稳定的即可。如需了解构造图形常量的其他方法，请参阅[创建 ID](https://developer.android.google.cn/guide/navigation/navigation-kotlin-dsl#create-ids)。

### 2.3 使用 NavGraphBuilder DSL 构建图形

* 定义一组初始 ID 后，即可构建导航图。使用 [`NavController.createGraph()`](https://developer.android.google.cn/reference/kotlin/androidx/navigation/package-summary#(androidx.navigation.NavController).createGraph(kotlin.Int, kotlin.Int, kotlin.Function1)) 扩展函数创建一个 `NavGraph`，并传入图形的 `id`、`startDestination` 的 ID 值以及用于定义图形结构的尾随 lambda。

* 您可以在 Activity 的 `onCreate()` 函数中构建图形。`createGraph()` 会返回一个 [`Navgraph`](https://developer.android.google.cn/reference/kotlin/androidx/navigation/NavGraph)，然后您可以将其分配给与 [`NavHost`](https://developer.android.google.cn/reference/androidx/navigation/NavHost) 关联的 [`NavController`](https://developer.android.google.cn/reference/androidx/navigation/NavController) 的 `graph` 属性，如以下示例所示：

```kotlin
class GardenActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_garden)

        val navHostFragment = supportFragmentManager
                .findFragmentById(R.id.nav_host) as NavHostFragment

        navHostFragment.navController.apply {
            graph = createGraph(nav_graph.id, nav_graph.dest.home) {
                fragment<HomeViewPagerFragment>(nav_graph.dest.home) {
                    label = getString(R.string.home_title)
                    action(nav_graph.action.to_plant_detail) {
                        destinationId = nav_graph.dest.plant_detail
                    }
                }
                fragment<PlantDetailFragment>(nav_graph.dest.plant_detail) {
                    label = getString(R.string.plant_detail_title)
                    argument(nav_graph.args.plant_id) {
                        type = NavType.StringType
                    }
                }
            }
        }
    }
}
```

* 在该示例中，尾随 lambda 使用 [`fragment()`](https://developer.android.google.cn/reference/kotlin/androidx/navigation/fragment/package-summary#(androidx.navigation.NavGraphBuilder).fragment(kotlin.Int, kotlin.Function1)) DSL 构建器函数定义了两个 Fragment 目的地。该函数需要目的地的 ID。该函数还接受用于其他配置的可选 lambda（如目的地 `label`）以及用于操作、参数和深层链接的嵌入式构建器函数。

* 管理每个目的地界面的 [`Fragment`](https://developer.android.google.cn/reference/kotlin/androidx/fragment/app/Fragment) 类将作为放在尖括号 (`<>`) 中的参数化类型传入。这与在使用 XML 定义的 Fragment 目的地上设置 `android:name` 属性具有相同的效果。

### 2.4 浏览 Kotlin DSL 图

* 构建并设置图形后，即可使用 `NavController.navigate()` 从 `home` 导航到 `plant_detail`，如以下示例所示：

```kotlin
private fun navigateToPlant(plantId: String) {

    val args = bundleOf(nav_graph.args.plant_id to plantId)

    findNavController().navigate(nav_graph.action.to_plant_detail, args)
}
```

* **注意**：Safe Args 插件与 Kotlin DSL 不兼容，因为该插件会查找 XML 资源文件生成 `Directions` 和 `Arguments` 类。

## 三、支持的目的地类型

* Kotlin DSL 支持 `Fragment`、`Activity` 和 `NavGraph` 目的地，每个目的地都有自己的内嵌扩展函数，可用于构建和配置目的地。

### 3.1 Fragment 目的地

* 可以将 [`fragment()`](https://developer.android.google.cn/reference/kotlin/androidx/navigation/fragment/package-summary#(androidx.navigation.NavGraphBuilder).fragment(kotlin.Int)) DSL 函数参数化为实现 `Fragment` 类。该函数使用分配给该目的地的唯一 ID 以及您可以在其中提供其他配置的 lambda。

```kotlin
fragment<FragmentDestination>(nav_graph.dest.fragment_dest_id) {
   label = getString(R.string.fragment_title)
   // arguments, actions, deepLinks...
}
```

### 3.2 Activity 目的地

* [`activity()`](https://developer.android.google.cn/reference/kotlin/androidx/navigation/package-summary#(androidx.navigation.NavGraphBuilder).activity(kotlin.Int, kotlin.Function1)) DSL 函数使用分配给该目的地的唯一 ID，但未将其参数化为任何实现 Activity 类。相反，您可以在尾随 lambda 中设置可选 `activityClass`。这种灵活性可以让您为从[隐式 intent](https://developer.android.google.cn/guide/components/intents-filters#ExampleSend) 启动的 Activity 定义 Activity 目的地，在这种情况下，显式 Activity 类将毫无意义。与 Fragment 目的地一样，您还可以定义并配置标签和任何参数。

```kotlin
activity(nav_graph.dest.activity_dest_id) {
    label = getString(R.string.activity_title)
    // arguments, actions, deepLinks...

    activityClass = ActivityDestination::class
}
```

### 3.3 导航图目的地

* 您可以使用 [`navigation()`](https://developer.android.google.cn/reference/kotlin/androidx/navigation/package-summary#(androidx.navigation.NavGraphBuilder).navigation(kotlin.Int, kotlin.Int, kotlin.Function1)) DSL 函数构建[嵌套导航图](https://developer.android.google.cn/guide/navigation/navigation-nested-graphs)。与其他目的地类型一样，该 DSL 函数使用三个参数：分配给图形的 ID、图形的起始目的地 ID 以及用于进一步配置图形的 lambda。lambda 的有效元素包括参数、操作、其他目的地、深层链接和标签。

```kotlin
navigation(nav_graph.dest.nav_graph_dest, nav_graph.dest.start_dest) {
   // label, arguments, actions, other destinations, deep links
}
```

### 3.4 支持自定义目的地

* 您可以使用 [`addDestination()`](https://developer.android.google.cn/reference/androidx/navigation/NavGraph#addDestination(androidx.navigation.NavDestination)) 将默认不直接支持的自定义目的地类型添加到 Kotlin DSL，如以下示例所示：

```kotlin
// The NavigatorProvider is retrieved from the NavController
val customDestination = navigatorProvider[CustomNavigator::class].createDestination().apply {
    id = nav_graph.dest.custom_dest_id
}
addDestination(customDestination)
```

* 您还可以使用一元加号运算符 (`+`) 将新构造的目的地直接添加到图形中：

```kotlin
// The NavigatorProvider is retrieved from the NavController
+navigatorProvider[CustomNavigator::class].createDestination().apply {
    id = nav_graph.dest.custom_dest_id
}
```

### 3.5 提供目的地参数

* 您可以为任何目的地类型定义可选或必需参数。如需定义参数，请针对所有目的地构建器类型的基类 `NavDestinationBuilder` 调用 [`argument()`](https://developer.android.google.cn/reference/kotlin/androidx/navigation/NavDestinationBuilder#argument(kotlin.String, kotlin.Function1))。该函数将参数的名称作为可用于构造和配置 [`NavArgument`](https://developer.android.google.cn/reference/kotlin/androidx/navigation/NavDestinationBuilder#argument(kotlin.String, kotlin.Function1)) 的 `String` 和 lambda。在 lambda 中，您可以指定参数数据类型、默认值（如果适用）以及参数值是否可以为 `null`。

```kotlin
fragment<PlantDetailFragment>(nav_graph.dest.plant_detail) {
    label = getString(R.string.plant_details_title)
    argument(nav_graph.args.plant_name) {
        type = NavType.StringType
        defaultValue = getString(R.string.default_plant_name)
        nullable = true  // default false
    }
}
```

* 如果指定了 `defaultValue`，则 `type` 是可选的。在这种情况下，如果未指定 `type`，则从 `defaultValue` 推断类型。如果同时提供了 `defaultValue` 和 `type`，这些类型必须匹配。如需查看参数类型的完整列表，请参阅 [`NavType`](https://developer.android.google.cn/reference/androidx/navigation/NavType)。

## 四、操作

* 您可以定义任何目的地内的操作，包括根导航图中的[全局操作](https://developer.android.google.cn/guide/navigation/navigation-global-action)。如需定义操作，请使用 [`NavDestinationBuilder.action()`](https://developer.android.google.cn/reference/kotlin/androidx/navigation/NavDestinationBuilder#action(kotlin.Int, kotlin.Function1)) 函数，为该函数提供一个 ID 并提供一个 lambda 以提供其他配置。

* 以下示例构建了一项具有 `destinationId`、过渡动画以及弹出行为和单一顶级行为的操作。

```kotlin
action(nav_graph.action.to_plant_detail) {
    destinationId = nav_graph.dest.plant_detail
    navOptions {
        anim {
            enter = R.anim.nav_default_enter_anim
            exit = R.anim.nav_default_exit_anim
            popEnter = R.anim.nav_default_pop_enter_anim
            popExit = R.anim.nav_default_pop_exit_anim
        }
        popUpTo(nav_graph.dest.start_dest) {
            inclusive = true // default false
        }
        // if popping exclusively, you can specify popUpTo as
        // a property. e.g. popUpTo = nav_graph.dest.start_dest
        launchSingleTop = true // default false
    }
}
```

* **注意** ：[导航 `2.3.0-alpha05`](https://developer.android.google.cn/jetpack/androidx/releases/navigation#2.3.0-alpha05) 支持使用 `defaultArguments` `Map` 为操作的参数添加默认值。

## 五、深层链接

* 您可以将深层链接添加到任何目的地，就像使用基于 XML 的导航图一样。[为目的地创建深层链接](https://developer.android.google.cn/guide/navigation/navigation-deep-link#implicit)中定义的相同程序也适用于使用 Kotlin DSL 创建[显式深层链接](https://developer.android.google.cn/guide/navigation/navigation-deep-link#explicit)的过程。

* 但是，在创建隐式深层链接时，您没有可以针对 `<deepLink>` 元素进行分析的 XML 导航资源。因此，您不能依赖于在 `AndroidManifest.xml` 文件中放置 `<nav-graph>` 元素，而是必须手动向 Activity [添加 Intent 过滤器](https://developer.android.google.cn/training/app-links/deep-linking)。您提供的 Intent 过滤器应与应用深层链接的基本网址格式匹配。

* 对于每个单独的深层链接目的地，您可以使用 [`deepLink()`](https://developer.android.google.cn/reference/kotlin/androidx/navigation/NavDestinationBuilder#deepLink(kotlin.String)) DSL 函数提供更具体的 URI 模式。该函数接受 `String` 作为 URI 模式，如以下示例所示：

```kotlin
deepLink("http://www.example.com/plants/")
```

* 您可以添加的深层链接 URI 数量没有限制。每次调用 `deepLink()` 时，都会将新的深层链接附加到特定于该目的地的内部列表。

* 以下是更复杂的隐式深层链接场景，该场景还定义了基于路径和基于查询的参数：

```kotlin
val baseUri = "http://www.example.com/plants"

fragment<PlantDetailFragment>(nav_graph.dest.plant_detail) {
    label = getString(R.string.plant_details_title)
    deepLink("${baseUri}/{id}")
    deepLink("${baseUri}/{id}?name={plant_name}")
    argument(nav_graph.args.plant_id) {
       type = NavType.IntType
    }
    argument(nav_graph.args.plant_name) {
        type = NavType.StringType
        nullable = true
    }
}
```

* 请注意，可以使用[字符串插值](https://kotlinlang.org/docs/reference/idioms.html#string-interpolation)简化定义。

## 六、创建 ID

* 导航库要求用于图形元素的 ID 值是唯一整数，这些整数在配置更改时保持不变。创建这些 ID 的一种方法是将其定义为静态常量，如[为图形创建常量](https://developer.android.google.cn/guide/navigation/navigation-kotlin-dsl#constants)中所示。您还可以将 XML 中的静态资源 ID 定义为资源。或者，您也可以动态构造 ID。例如，您可以创建一个在每次引用时都会递增的序列计数器。

```kotlin
object nav_graph {
    // Counter for id's. First ID will be 1.
    var id_counter = 1

    val id = id_counter++

    object dest {
       val home = id_counter++
       val plant_detail = id_counter++
    }

    object action {
       val to_plant_detail = id_counter++
    }

    object args {
       const val plant_id = "plantId"
    }
}
```

## 七、限制

- Safe Args 插件与 Kotlin DSL 不兼容，因为该插件会查找 XML 资源文件以生成 `Directions` 和 `Arguments` 类。