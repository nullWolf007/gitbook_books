[TOC]

# Compose 编程思想

* Jetpack Compose 是一个适用于 Android 的新式声明性界面工具包。Compose 提供声明性 API，让您可在不以命令方式改变前端视图的情况下呈现应用界面，从而使编写和维护应用界面变得更加容易。此术语需要一些解释说明，它的含义对应用设计非常重要。

## 一、声明性编程范式

* 长期以来，Android 视图层次结构一直可以表示为界面微件树。由于应用的状态会因用户交互等因素而发生变化，因此界面层次结构需要进行更新以显示当前数据。最常见的界面更新方式是使用 [`findViewById()`](https://developer.android.com/reference/android/view/View#findViewById(int)) 等函数遍历树，并通过调用 `button.setText(String)`、`container.addChild(View)` 或 `img.setImageBitmap(Bitmap)` 等方法更改节点。这些方法会改变微件的内部状态。

* 手动操纵视图会提高出错的可能性。如果一条数据在多个位置呈现，很容易忘记更新显示它的某个视图。此外，当两项更新以意外的方式发生冲突时，也很容易造成异常状态。例如，某项更新可能会尝试设置刚刚从界面中移除的节点的值。一般来说，软件维护复杂性会随着需要更新的视图数量而增长。

* 在过去的几年中，整个行业已开始转向声明性界面模型，该模型大大简化了与构建和更新界面关联的工程设计。该技术的工作原理是在概念上从头开始重新生成整个屏幕，然后仅执行必要的更改。此方法可避免手动更新有状态视图层次结构的复杂性。Compose 是一个声明性界面框架。

* 重新生成整个屏幕所面临的一个难题是，在时间、计算能力和电池用量方面可能成本高昂。为了减轻这一成本，Compose 会智能地选择在任何给定时间需要重新绘制界面的哪些部分。这会对您设计界面组件的方式有一定影响，如[重组](https://developer.android.com/jetpack/compose/mental-model#recomposition)中所述。

## 二、简单的可组合函数

* 使用 Compose，您可以通过定义一组接受数据而发出界面元素的可组合函数来构建界面。一个简单的示例是 `Greeting` 微件，它接受 `String` 而发出一个显示问候消息的 `Text` 微件。

![显示文本“Hello World”的手机的屏幕截图，以及用于生成该界面的简单可组合函数的代码](https://developer.android.com/images/jetpack/compose/mmodel-simple.png)

* **图 1.** 一个简单的可组合函数，系统向它传递了数据，它使用该数据在屏幕上呈现文本微件。

* 关于此函数，有几点值得注意：
  * 此函数带有 `@Composable` 注释。所有可组合函数都必须带有此注释；此注释可告知 Compose 编译器：此函数旨在将数据转换为界面。
  * 此函数接受数据。可组合函数可以接受一些参数，这些参数可让应用逻辑描述界面。在本例中，我们的微件接受一个 `String`，因此它可以按名称问候用户。
  * 此函数可以在界面中显示文本。为此，它会调用 `Text()` 可组合函数，该函数实际上会创建文本界面元素。可组合函数通过调用其他可组合函数来发出界面层次结构。
  * 此函数不会返回任何内容。发出界面的 Compose 函数不需要返回任何内容，因为它们描述所需的屏幕状态，而不是构造界面微件。
  * 此函数快速、[幂等](https://en.wikipedia.org/wiki/Idempotence#Computer_science_meaning)且没有副作用。
    - 使用同一参数多次调用此函数时，它的行为方式相同，并且它不使用其他值，如全局变量或对 `random()` 的调用。
    - 此函数描述界面而没有任何副作用，如修改属性或全局变量。

* 一般来说，出于[重组](https://developer.android.com/jetpack/compose/mental-model#recomposition)部分所述的原因，所有可组合函数都应使用这些属性来编写。

## 三、声明性范式转变

* 在许多面向对象的命令式界面工具包中，您可以通过实例化微件树来初始化界面。您通常通过膨胀 XML 布局文件来实现此目的。每个微件都维护自己的内部状态，并且提供 getter 和 setter 方法，允许应用逻辑与微件进行交互。

* 在 Compose 的声明性方法中，微件相对无状态，并且不提供 setter 或 getter 函数。实际上，微件不会以对象形式提供。您可以通过调用带有不同参数的同一可组合函数来更新界面。这使得向架构模式（如 [`ViewModel`](https://developer.android.com/reference/androidx/lifecycle/ViewModel)）提供状态变得很容易，如[应用架构指南](https://developer.android.com/jetpack/guide)中所述。然后，可组合项负责在每次可观察数据更新时将当前应用状态转换为界面。

![Compose 界面中数据从高级对象向下流向其子级的图示。](https://developer.android.com/images/jetpack/compose/mmodel-flow-data.png)

* **图 2.** 应用逻辑为顶级可组合函数提供数据。该函数通过调用其他可组合函数来使用这些数据描述界面，将适当的数据传递给这些可组合函数，并沿层次结构向下传递数据。

* 当用户与界面交互时，界面会发起 `onClick` 等事件。这些事件应通知应用逻辑，应用逻辑随后可以改变应用的状态。当状态发生变化时，系统会使用新数据再次调用可组合函数。这会导致重新绘制界面元素，此过程称为“重组”。

![说明界面元素如何通过触发由应用逻辑处理的事件来响应交互的图示。](https://developer.android.com/images/jetpack/compose/mmodel-flow-events.png)

* **图 3.** 用户与界面元素进行了交互，导致触发一个事件。应用逻辑响应该事件，然后系统根据需要使用新参数自动再次调用可组合函数。

## 四、动态内容

* 由于可组合函数是用 Kotlin 而不是 XML 编写的，因此它们可以像其他任何 Kotlin 代码一样动态。例如，假设您想要构建一个界面，用来问候一些用户：

```kotlin
@Composable
fun Greeting(names: List<String>) {
    for (name in names) {
        Text("Hello $name")
    }
}
```

* 此函数接受名称的列表，并为每个用户生成一句问候语。可组合函数可能非常复杂。您可以使用 `if` 语句来确定是否要显示特定的界面元素。您可以使用循环。您可以调用辅助函数。您拥有底层语言的全部灵活性。这种强大的功能和灵活性是 Jetpack Compose 的主要优势之一。

## 五、重组

* 在命令式界面模型中，如需更改某个微件，您可以在该微件上调用 setter 以更改其内部状态。在 Compose 中，您可以使用新数据再次调用可组合函数。这样做会导致函数进行重组 -- 系统会根据需要使用新数据重新绘制函数发出的微件。Compose 框架可以智能地仅重组已更改的组件。

* 例如，假设有以下可组合函数，它用于显示一个按钮：

```kotlin
@Composable
fun ClickCounter(clicks: Int, onClick: () -> Unit) {
    Button(onClick = onClick) {
        Text("I've been clicked $clicks times")
    }
}
```

* 每次点击该按钮时，调用方都会更新 `clicks` 的值。Compose 会再次调用 lambda 与 `Text` 函数以显示新值；此过程称为“重组”。不依赖于该值的其他函数不会进行重组。

* 如前文所述，重组整个界面树在计算上成本高昂，因为会消耗计算能力并缩短电池续航时间。Compose 使用智能重组来解决此问题。

* 重组是指在输入更改时再次调用可组合函数的过程。当函数的输入更改时，会发生这种情况。当 Compose 根据新输入重组时，它仅调用可能已更改的函数或 lambda，而跳过其余函数或 lambda。通过跳过所有未更改参数的函数或 lambda，Compose 可以高效地重组。

* 切勿依赖于执行可组合函数所产生的附带效应，因为可能会跳过函数的重组。如果您这样做，用户可能会在您的应用中遇到奇怪且不可预测的行为。附带效应是指对应用的其余部分可见的任何更改。例如，以下操作全部都是危险的附带效应：
  * 写入共享对象的属性
  * 更新 `ViewModel` 中的可观察项
  * 更新共享偏好设置

* 可组合函数可能会像每一帧一样频繁地重新执行，例如在呈现动画时。可组合函数应快速执行，以避免在播放动画期间出现卡顿。如果您需要执行成本高昂的操作（例如从共享偏好设置读取数据），请在后台协程中执行，并将值结果作为参数传递给可组合函数。

* 例如，以下代码会创建一个可组合项以更新 `SharedPreferences` 中的值。该可组合项不应从共享偏好设置本身读取或写入，于是此代码将读取和写入操作移至后台协程中的 `ViewModel`。应用逻辑会使用回调传递当前值以触发更新。

```kotlin
@Composable
fun SharedPrefsToggle(
    text: String,
    value: Boolean,
    onValueChanged: (Boolean) -> Unit
) {
    Row {
        Text(text)
        Checkbox(checked = value, onCheckedChange = onValueChanged)
    }
}
```

* 本文档讨论了您在 Compose 中编程时需要注意的事项：
  * 可组合函数可以按任何顺序执行。
  * 可组合函数可以并行执行。
  * 重组会跳过尽可能多的可组合函数和 lambda。
  * 重组是乐观的操作，可能会被取消。
  * 可组合函数可能会像动画的每一帧一样非常频繁地运行。

* 下面几部分将介绍如何构建可组合函数以支持重组。在每种情况下，最佳做法都是使可组合函数保持快速、幂等且没有附带效应。

### 5.1 可组合函数可以按任何顺序执行

* 如果您看一下可组合函数的代码，可能会认为这些代码按其出现的顺序运行。但其实未必是这样。如果某个可组合函数包含对其他可组合函数的调用，这些函数可以按任何顺序运行。Compose 可以选择识别出某些界面元素的优先级高于其他界面元素，因而首先绘制这些元素。

* 例如，假设您有如下代码，用于在标签页布局中绘制三个屏幕：

```kotlin
@Composable
fun ButtonRow() {
    MyFancyNavigation {
        StartScreen()
        MiddleScreen()
        EndScreen()
    }
}
```

* 对 `StartScreen`、`MiddleScreen` 和 `EndScreen` 的调用可以按任何顺序进行。这意味着，举例来说，您不能让 `StartScreen()` 设置某个全局变量（附带效应）并让 `MiddleScreen()` 利用这项更改。相反，其中每个函数都需要保持独立。

### 5.2 可组合函数可以并行运行

* Compose 可以通过并行运行可组合函数来优化重组。这样一来，Compose 就可以利用多个核心，并以较低的优先级运行可组合函数（不在屏幕上）。

* 这种优化意味着，可组合函数可能会在后台线程池中执行。如果某个可组合函数对 `ViewModel` 调用一个函数，则 Compose 可能会同时从多个线程调用该函数。

* 为了确保应用正常运行，所有可组合函数都不应有附带效应，而应通过始终在界面线程上执行的 `onClick` 等回调触发附带效应。

* 调用某个可组合函数时，调用可能发生在与调用方不同的线程上。这意味着，应避免使用修改可组合 lambda 中的变量的代码，既因为此类代码并非线程安全代码，又因为它是可组合 lambda 不允许的附带效应。

* 以下示例展示了一个可组合项，它显示一个列表及其项数：

```kotlin
@Composable
fun ListComposable(myList: List<String>) {
    Row(horizontalArrangement = Arrangement.SpaceBetween) {
        Column {
            for (item in myList) {
                Text("Item: $item")
            }
        }
        Text("Count: ${myList.size}")
    }
}
```

* 此代码没有附带效应，它会将输入列表转换为界面。此代码非常适合显示小列表。不过，如果函数写入局部变量，则这并非线程安全或正确的代码：

```kotlin
@Composable
@Deprecated("Example with bug")
fun ListWithBug(myList: List<String>) {
    var items = 0

    Row(horizontalArrangement = Arrangement.SpaceBetween) {
        Column {
            for (item in myList) {
                Text("Item: $item")
                items++ // Avoid! Side-effect of the column recomposing.
            }
        }
        Text("Count: $items")
    }
}
```

* 在本例中，每次重组时，都会修改 `items`。这可以是动画的每一帧，或是在列表更新时。但不管怎样，界面都会显示错误的项数。因此，Compose 不支持这样的写入操作；通过禁止此类写入操作，我们允许框架更改线程以执行可组合 lambda。

### 5.3 重组会跳过尽可能多的内容

* 如果界面的某些部分无效，Compose 会尽力只重组需要更新的部分。这意味着，它可以跳过某些内容以重新运行单个按钮的可组合项，而不执行界面树中在其上面或下面的任何可组合项。

* 每个可组合函数和 lambda 都可以自行重组。以下示例演示了在呈现列表时重组如何跳过某些元素：

```kotlin
/**
 * Display a list of names the user can click with a header
 */
@Composable
fun NamePicker(
    header: String,
    names: List<String>,
    onNameClicked: (String) -> Unit
) {
    Column {
        // this will recompose when [header] changes, but not when [names] changes
        Text(header, style = MaterialTheme.typography.h5)
        Divider()

        // LazyColumn is the Compose version of a RecyclerView.
        // The lambda passed to items() is similar to a RecyclerView.ViewHolder.
        LazyColumn {
            items(names) { name ->
                // When an item's [name] updates, the adapter for that item
                // will recompose. This will not recompose when [header] changes
                NamePickerItem(name, onNameClicked)
            }
        }
    }
}

/**
 * Display a single name the user can click.
 */
@Composable
private fun NamePickerItem(name: String, onClicked: (String) -> Unit) {
    Text(name, Modifier.clickable(onClick = { onClicked(name) }))
}
```

* 这些作用域中的每一个都可能是在重组期间执行的唯一一个作用域。当 `header` 发生更改时，Compose 可能会跳至 `Column` lambda，而不执行它的任何父项。此外，执行 `Column` 时，如果 `names` 未更改，Compose 可能会选择跳过 `LazyColumnItems`。

* 同样，执行所有可组合函数或 lambda 都应该没有附带效应。当您需要执行附带效应时，应通过回调触发。

### 5.4 重组是乐观的操作

* 只要 Compose 认为某个可组合项的参数可能已更改，就会开始重组。重组是乐观的操作，也就是说，Compose 预计会在参数再次更改之前完成重组。如果某个参数在重组完成之前发生更改，Compose 可能会取消重组，并使用新参数重新开始。

* 取消重组后，Compose 会从重组中舍弃界面树。如有任何附带效应依赖于显示的界面，则即使取消了组成操作，也会应用该附带效应。这可能会导致应用状态不一致。

* 确保所有可组合函数和 lambda 都幂等且没有附带效应，以处理乐观的重组。

### 5.5 可组合函数可能会非常频繁地运行

* 在某些情况下，可能会针对界面动画的每一帧运行一个可组合函数。如果该函数执行成本高昂的操作（例如从设备存储空间读取数据），可能会导致界面卡顿。

* 例如，如果您的微件尝试读取设备设置，它可能会在一秒内读取这些设置数百次，这会对应用的性能造成灾难性的影响。

* 如果您的可组合函数需要数据，它应为相应的数据定义参数。然后，您可以将成本高昂的工作移至组成操作线程之外的其他线程，并使用 `mutableStateOf` 或 `LiveData` 将相应的数据传递给 Compose。