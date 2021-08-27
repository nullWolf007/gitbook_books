[TOC]

# 使用 CompositionLocal 将数据的作用域限定在局部

* [`CompositionLocal`](https://developer.android.com/reference/kotlin/androidx/compose/runtime/CompositionLocal) 是通过组合隐式向下传递数据的工具。在本页中，您将更详细地了解什么是 `CompositionLocal`、如何创建自己的 `CompositionLocal`，以及对您的用例而言 `CompositionLocal` 是否为合适的解决方案。

## 一、CompositionLocal 简介

* 通常情况下，在 Compose 中，数据以参数形式[向下流经](https://developer.android.com/jetpack/compose/architecture)整个界面树传递给每个可组合函数。这会使可组合项的依赖项变为显式依赖项。但是，对于广泛使用的常用数据（如颜色或类型样式），这可能会很麻烦。请参阅以下示例：

```kotlin
@Composable
fun MyApp() {
    // Theme information tends to be defined near the root of the application
    val colors = …
}

// Some composable deep in the hierarchy
@Composable
fun SomeTextLabel(labelText: String) {
    Text(
        text = labelText,
        color = // ← need to access colors here
    )
}
```

* 为了支持无需将颜色作为显式参数依赖项传递给大多数可组合项，**Compose 提供了 [`CompositionLocal`](https://developer.android.com/reference/kotlin/androidx/compose/runtime/CompositionLocal)，可让您创建以树为作用域的具名对象，这可以用作让数据流经界面树的一种隐式方式。**

* `CompositionLocal` 元素通常在界面树的某个节点以值的形式提供。该值可供其可组合项的后代使用，而无需在可组合函数中将 `CompositionLocal` 声明为参数。

* **关键术语**：在本指南中，我们使用了术语**组合**、**界面树**和**界面层次结构**。虽然在其他指南中可能互换使用这些术语，但其含义有所不同：

* **组合**是可组合函数的调用图的记录。

* **界面树**或**界面层次结构**是由组合进程构造、更新和维护的 `LayoutNode` 树。

* `CompositionLocal` 是 Material 主题在后台使用的内容。 [`MaterialTheme`](https://developer.android.com/reference/kotlin/androidx/compose/material/MaterialTheme) 对象提供了三个 `CompositionLocal` 实例，即 colors、typography 和 shapes。您可以之后在组合的任何后代部分中检索这些实例。具体来说，这些是可以通过 `MaterialTheme` `colors`、`shapes` 和 `typography` 属性访问的 `LocalColors`、`LocalShapes` 和 `LocalTypography` 属性。

```kotlin
@Composable
fun MyApp() {
    // Provides a Theme whose values are propagated down its `content`
    MaterialTheme {
        // New values for colors, typography, and shapes are available
        // in MaterialTheme's content lambda.

        // ... content here ...
    }
}

// Some composable deep in the hierarchy of MaterialTheme
@Composable
fun SomeTextLabel(labelText: String) {
    Text(
        text = labelText,
        // `primary` is obtained from MaterialTheme's
        // LocalColors CompositionLocal
        color = MaterialTheme.colors.primary
    )
}
```

* **`CompositionLocal` 实例的作用域限定为组合的一部分**，因此您可以在结构树的不同级别提供不同的值。`CompositionLocal` 的 [`current`](https://developer.android.com/reference/kotlin/androidx/compose/runtime/CompositionLocal#current()) 值对应于该组合部分中的某个祖先提供的最接近的值。

* **如需为 `CompositionLocal` 提供新值，请使用 [`CompositionLocalProvider`](https://developer.android.com/reference/kotlin/androidx/compose/runtime/package-summary#CompositionLocalProvider(kotlin.Array,kotlin.Function0))** 及其 [`provides`](https://developer.android.com/reference/kotlin/androidx/compose/runtime/ProvidableCompositionLocal#provides(kotlin.Any)) infix 函数，该函数将 `CompositionLocal` 键与 `value` 相关联。在访问 `CompositionLocal` 的 `current` 属性时，`CompositionLocalProvider` 的 `content` lambda 将获取提供的值。提供新值后，Compose 会重组读取 `CompositionLocal` 的组合部分。

* 例如，[`LocalContentAlpha`](https://developer.android.com/reference/kotlin/androidx/compose/material/package-summary#LocalContentAlpha ()) `CompositionLocal` 包含用于文本和图标的首选内容 Alpha 值，以强调或弱化界面的不同部分。在以下示例中，`CompositionLocalProvider` 用于为组合的不同部分提供不同的值。

```kotlin
@Composable
fun CompositionLocalExample() {
    MaterialTheme { // MaterialTheme sets ContentAlpha.high as default
        Column {
            Text("Uses MaterialTheme's provided alpha")
            CompositionLocalProvider(LocalContentAlpha provides ContentAlpha.medium) {
                Text("Medium value provided for LocalContentAlpha")
                Text("This Text also uses the medium value")
                CompositionLocalProvider(LocalContentAlpha provides ContentAlpha.disabled) {
                    DescendantExample()
                }
            }
        }
    }
}

@Composable
fun DescendantExample() {
    // CompositionLocalProviders also work across composable functions
    Text("This Text uses the disabled alpha now")
}
```

<img src="https://developer.android.com/images/jetpack/compose/compositionlocal-alpha.png" alt="img" style="zoom:50%;" />

* **图 1.** `CompositionLocalExample` 可组合项预览。

* 在上面的所有示例中，由 Material 可组合项在内部使用 `CompositionLocal` 实例。如需访问 `CompositionLocal` 的当前值，请使用其 [`current`](https://developer.android.com/reference/kotlin/androidx/compose/runtime/CompositionLocal#current()) 属性。在以下示例中，使用 Android 应用中常用的 [`LocalContext`](https://developer.android.com/reference/kotlin/androidx/compose/ui/platform/package-summary#LocalContext()) `CompositionLocal` 的当前 [`Context`](https://developer.android.com/reference/android/content/Context) 值来设置文本格式：

```kotlin
@Composable
fun FruitText(fruitSize: Int) {
    // Get `resources` from the current value of LocalContext
    val resources = LocalContext.current.resources
    val fruitText = remember(resources, fruitSize) {
        resources.getQuantityString(R.plurals.fruit_title, fruitSize)
    }
    Text(text = fruitText)
}
```

* **注意**：`CompositionLocal` 对象或常量通常带有 `Local` 前缀，以便在 IDE 中利用自动填充功能提高可检测性。

## 二、创建您自己的 CompositionLocal

* `CompositionLocal` 是**通过组合隐式向下传递数据的工具**。

* **使用 `CompositionLocal` 的另一个关键信号是该参数为横切参数且中间层的实现不应知道该参数的存在**，因为让这些中间层知道会限制可组合项的功用。例如，对 Android 权限的查询是由 `CompositionLocal` 在后台提供的。媒体选择器可组合项可以添加新功能，即访问设备上受权限保护的内容而无需更改其 API，并且需要媒体选择器的调用方知道从环境中使用的这一新增的上下文。

* 但是，`CompositionLocal` 并非始终是最好的解决方案。我们不建议过度使用 `CompositionLocal`，因为它存在一些缺点：

* **`CompositionLocal` 使得可组合项的行为更难推断**。在创建隐式依赖项时，使用这些依赖项的可组合项的调用方需要确保为每个 `CompositionLocal` 提供一个值。

* 此外，该依赖项可能没有明确的可信来源，因为它可能会在组合中的任何部分发生改变。因此，**在出现问题时调试应用可能更具有挑战性**因为需要向上查看组合，了解提供 `current` 值的位置。Android Studio 中的“Find usages”或 [Compose 布局检查器](https://developer.android.com/jetpack/compose/tooling#layout-inspector)等工具提供了足够的信息来缓解这个问题。

* **注意**：`CompositionLocal` 非常适合基础架构，而且 Jetpack Compose 大量使用该工具。

### 2.1 决定是否使用 CompositionLocal

* 在某些情况下，`CompositionLocal` 对于您的用例而言可能是很好的解决方案：

* **`CompositionLocal` 应具有合适的默认值**。如果没有默认值，您必须保证开发者极其难陷入不提供 `CompositionLocal` 值的状况。如果创建测试或预览使用该 `CompositionLocal` 的可组合项时始终需要显式提供默认值，那么不提供默认值可能会导致出现问题并带来糟糕的体验。

* **有些概念并非以树或子层次结构为作用域，请避免对这些概念使用 `CompositionLocal`**。建议使用 `CompositionLocal` 的情况为：其可能会被任何（而非少数几个）后代使用。

* 如果您的用例不满足这些要求，请在创建 `CompositionLocal` 之前查看[需考虑的替代方案](https://developer.android.com/jetpack/compose/compositionlocal#alternatives)一节。

* 一种错误做法的示例是创建包含特定屏幕的 `ViewModel` 的 `CompositionLocal`，以便该屏幕中的所有可组合项都可以获取 `ViewModel` 来执行某些逻辑。这是一种错误做法，因为并非特定界面树下的所有可组合项都需要知道 `ViewModel`。最佳做法是遵循[状态向下传递而事件向上传递](https://developer.android.com/jetpack/compose/architecture)的模式，只向可组合项传递所需信息。这样做会使可组合项的可重用性更高，并且更易于测试。

### 2.2 创建 CompositionLocal

* 有两个 API 可用于创建 `CompositionLocal`：
  * [`compositionLocalOf`](https://developer.android.com/reference/kotlin/androidx/compose/runtime/package-summary#compositionLocalOf(androidx.compose.runtime.SnapshotMutationPolicy,kotlin.Function0))：在重组期间更改提供的值只会使读取其 [`current`](https://developer.android.com/reference/kotlin/androidx/compose/runtime/CompositionLocal#current()) 值的内容无效。
  * [`staticCompositionLocalOf`](https://developer.android.com/reference/kotlin/androidx/compose/runtime/package-summary#staticCompositionLocalOf(kotlin.Function0))：与 `compositionLocalOf` 不同，Compose 不会跟踪 `staticCompositionLocalOf` 的读取。更改该值会导致提供 `CompositionLocal` 的整个 `content` lambda 被重组，而不仅仅是在组合中读取 `current` 值的位置。

* 如果为 `CompositionLocal` 提供的值发生更改的可能性微乎其微或永远不会更改，使用 `staticCompositionLocalOf` 可提高性能。

* 例如，应用的设计系统在使用界面组件的阴影来提升可组合项的方式上可能会保持自己的特色。由于应该在整个界面树中传播应用的不同高度，因此我们使用 `CompositionLocal`。由于 `CompositionLocal` 值是根据系统主题有条件派生的，因此我们使用 `compositionLocalOf` API：

```kotlin
// LocalElevations.kt file

data class Elevations(val card: Dp = 0.dp, val default: Dp = 0.dp)

// Define a CompositionLocal global object with a default
// This instance can be accessed by all composables in the app
val LocalElevations = compositionLocalOf { Elevations() }
```

### 2.3 为 CompositionLocal 提供值

* **[`CompositionLocalProvider`](https://developer.android.com/reference/kotlin/androidx/compose/runtime/package-summary#CompositionLocalProvider(kotlin.Array,kotlin.Function0)) 可组合项可将值绑定到给定层次结构的 `CompositionLocal` 实例**。如需为 `CompositionLocal` 提供新值，请使用 [`provides`](https://developer.android.com/reference/kotlin/androidx/compose/runtime/ProvidableCompositionLocal#provides (kotlin.Any)) infix 函数，该函数将 `CompositionLocal` 键与 `value` 相关联，如下所示：

```kotlin
// MyActivity.kt file

class MyActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        setContent {
            // Calculate elevations based on the system theme
            val elevations = if (isSystemInDarkTheme()) {
                Elevations(card = 1.dp, default = 1.dp)
            } else {
                Elevations(card = 0.dp, default = 0.dp)
            }

            // Bind elevation as the value for LocalElevations
            CompositionLocalProvider(LocalElevations provides elevations) {
                // ... Content goes here ...
                // This part of Composition will see the `elevations` instance
                // when accessing LocalElevations.current
            }
        }
    }
}
```

### 2.4 使用 CompositionLocal

* [`CompositionLocal.current`](https://developer.android.com/reference/kotlin/androidx/compose/runtime/ProvidableCompositionLocal#current()) 返回由最接近的 `CompositionLocalProvider`（其向该 `CompositionLocal` 提供一个值）提供的值：

```kotlin
@Composable
fun SomeComposable() {
    // Access the globally defined LocalElevations variable to get the
    // current Elevations in this part of the Composition
    Card(elevation = LocalElevations.current.card) {
        // Content
    }
}
```

## 三、需考虑的替代方案

* 对于某些用例，`CompositionLocal` 可能是一种过度的解决方案。如果您的用例不符合[决定是否使用 CompositionLocal](https://developer.android.com/jetpack/compose/compositionlocal#deciding) 一节中指定的条件，其他解决方案可能更适合您的用例。

### 3.1 传递显式参数

* 显式使用可组合项的依赖项是一种很好的习惯。我们建议您**仅传递所需可组合项**。为了鼓励分离和重用可组合项，每个可组合项包含的信息应该可能少。

```kotlin
@Composable
fun MyComposable(myViewModel: MyViewModel = viewModel()) {
    // ...
    MyDescendant(myViewModel.data)
}

// Don't pass the whole object! Just what the descendant needs.
// Also, don't  pass the ViewModel as an implicit dependency using
// a CompositionLocal.
@Composable
fun MyDescendant(myViewModel: MyViewModel) { ... }

// Pass only what the descendant needs
@Composable
fun MyDescendant(data: DataToDisplay) {
    // Display data
}
```

### 3.2 控制反转

* 另一种避免将不必要的依赖项传递给可组合项的方法是通过控制反转。不是由后代接受依赖项来执行某些逻辑，而是由父级接受依赖项来执行某些逻辑。

* 在以下示例中，后代需要触发请求以加载某些数据：

```kotlin
@Composable
fun MyComposable(myViewModel: MyViewModel = viewModel()) {
    // ...
    MyDescendant(myViewModel)
}

@Composable
fun MyDescendant(myViewModel: MyViewModel) {
    Button(onClick = { myViewModel.loadData() }) {
        Text("Load data")
    }
}
```

* `MyDescendant` 可能需要承担很多工作，因具体情况而异。此外，将 `MyViewModel` 作为依赖项传递会降低 `MyDescendant` 的可重用性，因为它们现在已耦合在一起。请考虑使用不会将依赖项传递到后代并使用控制反转原则（让祖先实体负责执行逻辑）的替代方案：

```kotlin
@Composable
fun MyComposable(myViewModel: MyViewModel = viewModel()) {
    // ...
    ReusableLoadDataButton(
        onLoadClick = {
            myViewModel.loadData()
        }
    )
}

@Composable
fun ReusableLoadDataButton(onLoadClick: () -> Unit) {
    Button(onClick = onLoadClick) {
        Text("Load data")
    }
}
```

* 此方法更适合某些用例，因为它将子级与其直接祖先实体分离开来**将子级与其直接祖先实体分离**。祖先实体可组合项往往越来越复杂，这样就可以使更低级别的可组合项更灵活。

* 同样，**可以用相同的方式使用 `@Composable` 内容 lambda，以获得相同的优势**：

```kotlin
@Composable
fun MyComposable(myViewModel: MyViewModel = viewModel()) {
    // ...
    ReusablePartOfTheScreen(
        content = {
            Button(
                onClick = {
                    myViewModel.loadData()
                }
            ) {
                Text("Confirm")
            }
        }
    )
}

@Composable
fun ReusablePartOfTheScreen(content: @Composable () -> Unit) {
    Column {
        // ...
        content()
    }
}
```