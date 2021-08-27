[TOC]

# Compose 中的 ConstraintLayout

* `ConstraintLayout` 有助于根据可组合项的相对位置将它们放置在屏幕上，它是使用多个嵌套 `Row`、`Column`、`Box` 和自定义布局元素的替代方案。在实现对齐要求比较复杂的较大布局时，`ConstraintLayout` 很有用。

* 如需使用 Compose 中的 `ConstraintLayout`，您需要在 `build.gradle` 中添加以下依赖项：

```
implementation "androidx.constraintlayout:constraintlayout-compose:1.0.0-beta02"
```

* **警告**：`constraintLayout-compose` 工件的版本不同于 Jetpack Compose。在 [ConstraintLayout 版本页面](https://developer.android.com/jetpack/androidx/releases/constraintlayout?hl=zh-cn)中检查最新版本。

* **注意**：在 View 系统中，我们建议使用 `ConstraintLayout` 创建复杂的大型布局，因为扁平视图层次结构比嵌套视图的性能更好。不过，这在 Compose 中不是什么问题，因为它能够高效地处理较深的布局层次结构。

* **注意**：是否将 `ConstraintLayout` 用于 Compose 中的特定界面取决于开发者的偏好。在 Android View 系统中，使用 `ConstraintLayout` 作为构建更高性能布局的一种方法，但这在 Compose 中并不是问题。在需要进行选择时，请考虑 `ConstraintLayout` 是否有助于提高可组合项的可读性和可维护性。

* Compose 中的 `ConstraintLayout` 支持 [DSL](https://kotlinlang.org/docs/reference/type-safe-builders.html)：
  * 引用是使用 `createRefs()` 或 `createRefFor()` 创建的，`ConstraintLayout` 中的每个可组合项都需要有与之关联的引用。
  * 约束条件是使用 `constrainAs()` 修饰符提供的，该修饰符将引用作为参数，可让您在主体 lambda 中指定其约束条件。
  * 约束条件是使用 `linkTo()` 或其他有用的方法指定的。
  * `parent` 是一个现有的引用，可用于指定对 `ConstraintLayout` 可组合项本身的约束条件。

* 下面是使用 `ConstraintLayout` 的可组合项的示例：

```kotlin
@Composable
fun ConstraintLayoutContent() {
    ConstraintLayout {
        // Create references for the composables to constrain
        val (button, text) = createRefs()

        Button(
            onClick = { /* Do something */ },
            // Assign reference "button" to the Button composable
            // and constrain it to the top of the ConstraintLayout
            modifier = Modifier.constrainAs(button) {
                top.linkTo(parent.top, margin = 16.dp)
            }
        ) {
            Text("Button")
        }

        // Assign reference "text" to the Text composable
        // and constrain it to the bottom of the Button composable
        Text("Text", Modifier.constrainAs(text) {
            top.linkTo(button.bottom, margin = 16.dp)
        })
    }
}
```

* 此代码使用 `16.dp` 的外边距来约束 `Button` 顶部到父项的距离，同样使用 `16.dp` 的外边距来约束 `Text` 到 `Button` 底部的距离。

<img src="https://developer.android.com/images/jetpack/compose/layout-button-text.png?hl=zh-cn" alt="显示了按 ConstraintLayout 排列的按钮和文本元素" style="zoom:50%;" />

* 如需查看有关如何使用 `ConstraintLayout` 的更多示例，请参考[布局 Codelab](https://developer.android.com/codelabs/jetpack-compose-layouts?hl=zh-cn)。

## 一、Decoupled API

* 在 `ConstraintLayout` 示例中，约束条件是在应用它们的可组合项中使用修饰符以内嵌方式指定的。不过，在某些情况下，最好将约束条件与应用它们的布局分离开来。例如，您可能会希望根据屏幕配置来更改约束条件，或在两个约束条件集之间添加动画效果。

* 对于此类情况，您可以通过不同的方式使用 `ConstraintLayout`：

1. 将 `ConstraintSet` 作为参数传递给 `ConstraintLayout`。
2. 使用 `layoutId` 修饰符将在 `ConstraintSet` 中创建的引用分配给可组合项。

```kotlin
@Composable
fun DecoupledConstraintLayout() {
    BoxWithConstraints {
        val constraints = if (minWidth < 600.dp) {
            decoupledConstraints(margin = 16.dp) // Portrait constraints
        } else {
            decoupledConstraints(margin = 32.dp) // Landscape constraints
        }

        ConstraintLayout(constraints) {
            Button(
                onClick = { /* Do something */ },
                modifier = Modifier.layoutId("button")
            ) {
                Text("Button")
            }

            Text("Text", Modifier.layoutId("text"))
        }
    }
}

private fun decoupledConstraints(margin: Dp): ConstraintSet {
    return ConstraintSet {
        val button = createRefFor("button")
        val text = createRefFor("text")

        constrain(button) {
            top.linkTo(parent.top, margin = margin)
        }
        constrain(text) {
            top.linkTo(button.bottom, margin)
        }
    }
}
```

* 然后，当您需要更改约束条件时，只需传递不同的 `ConstraintSet` 即可。

## 二、了解详情

* 如需详细了解 Compose 中的 ConstraintLayout，请参阅 [Jetpack Compose Codelab 中的布局的约束布局部分](https://developer.android.com/codelabs/jetpack-compose-layouts?hl=zh-cn#9)，以及[使用 `ConstraintLayout` 的 Compose 示例](https://github.com/android/compose-samples/search?q=ConstraintLayout)中的 API 实际运用。