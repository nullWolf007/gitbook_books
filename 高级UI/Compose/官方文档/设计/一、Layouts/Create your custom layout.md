## [Create your custom layout](https://developer.android.com/courses/pathways/compose#6)

Compose promotes reusability of composables as small chunks that can be enough for some custom layouts by combining built-in composables such as `Column`, `Row`, or `Box`.

However, you might need to build something unique to your app that requires measuring and laying out children manually. For that, you can use the [`Layout`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main-release:compose/ui/ui/src/commonMain/kotlin/androidx/compose/ui/layout/Layout.kt) composable. In fact all higher level layouts like `Column` and `Row` are built with this.

**Note:** In the View system, creating a custom layout required extending `ViewGroup` and implementing measure and layout functions. In Compose you simply write a function using the `Layout` composable.

Before diving into how to create custom layouts, we need to know more about the principles of Layouts in Compose.

## Principles of layouts in Compose

Some composable functions emit a piece of UI when invoked that is added to a UI tree that will get rendered on the screen. Each emission (or element) has one parent and potentially many children. Also, it has a location within its parent: an (x, y) position, and a size: a `width` and `height`.

Elements are asked to measure themselves with Constraints that should be satisfied. Constraints restrict the minimum and maximum `width` and `height` of an element. If an element has child elements it may measure each of them to help determine its own size. Once an element reports its own size, it has an opportunity to place its child elements relative to itself. This will be further explained when creating the custom layout.

**Compose UI does not permit multi-pass measurement**. This means that a layout element may not measure any of its children more than once in order to try different measurement configurations. Single-pass measurement is good for performance, allowing Compose to handle efficiently deep UI trees. If a layout element measured its child twice and that child measured one of its children twice and so on, a single attempt to lay out a whole UI would have to do a lot of work, making it hard to keep your app performing well. However, there are times when you really need additional information on top of what a single child measurement would tell you - for these cases there are ways of doing this, we will talk about them later.

## Using the layout modifier

Use the `layout` modifier to manually control how to measure and position an element. Usually, the common structure of a custom `layout` modifier is as follows:

```
fun Modifier.customLayoutModifier(...) = Modifier.layout { measurable, constraints ->
  ...
})
```

When using the `layout` modifier, you get two lambda parameters:

- `measurable`: child to be measured and placed
- `constraints`: minimum and maximum for the width and height of the child

Let's say you want to display a `Text` on the screen and control the distance from the top to the baseline of the first line of texts. In order to do that, you'd need to manually place the composable on the screen using the `layout` modifier. See the desired behavior in the next picture where the distance from top to first baseline is `24.dp`:

![4ee1054702073598.png](https://developer.android.com/codelabs/jetpack-compose-layouts/img/4ee1054702073598.png)

Let's create a `firstBaselineToTop` modifier first:

```
fun Modifier.firstBaselineToTop(
  firstBaselineToTop: Dp
) = this.then(
    layout { measurable, constraints ->
        ...
    }
)
```

The first thing to do is measure the composable. As we mentioned in the Principles of Layout in Compose section, **you can only measure your children once**.

Measure the composable by calling `measurable.measure(constraints)`. When calling `measure(constraints)`, you can pass in the given constraints of the composable available in the `constraints` lambda parameter or create your own. The result of a `measure()` call on a `Measurable` is a `Placeable` that can be positioned by calling `placeRelative(x, y)`, as we will do later.

For this use case, don't constrain measurement further, just use the given constraints:

```
fun Modifier.firstBaselineToTop(
    firstBaselineToTop: Dp
) = this.then(
    layout { measurable, constraints ->
        val placeable = measurable.measure(constraints)

        ...
    }
)
```

Now that the composable has been measured, you need to calculate its size and specify it by calling the `layout(width, height)` method which also accepts a lambda used for placing the content.

In this case, the width of our composable will be the `width` of the measured composable and the height will be the composable's `height` with the desired top-to-baseline height minus the first baseline:

```
fun Modifier.firstBaselineToTop(
    firstBaselineToTop: Dp
) = this.then(
    layout { measurable, constraints ->
        val placeable = measurable.measure(constraints)

        // Check the composable has a first baseline
        check(placeable[FirstBaseline] != AlignmentLine.Unspecified)
        val firstBaseline = placeable[FirstBaseline]

        // Height of the composable with padding - first baseline
        val placeableY = firstBaselineToTop.roundToPx() - firstBaseline
        val height = placeable.height + placeableY
        layout(placeable.width, height) {
            ...
        }
    }
)
```

Now, you can position the composable on the screen by calling `placeable.placeRelative(x, y)`. If you don't call `placeRelative`, the composable won't be visible. `placeRelative` automatically adjusts the position of the placeable based on the current `layoutDirection`.

**Warning**: When creating a custom `Layout` or `LayoutModifier`, Android Studio will give a warning until the `layout` function is called.

In this case, the `y` position of the text corresponds to the top padding minus the position of the first baseline:

```
fun Modifier.firstBaselineToTop(
    firstBaselineToTop: Dp
) = this.then(
    layout { measurable, constraints ->
        ...

        // Height of the composable with padding - first baseline
        val placeableY = firstBaselineToTop.roundToPx() - firstBaseline
        val height = placeable.height + placeableY
        layout(placeable.width, height) {
            // Where the composable gets placed
            placeable.placeRelative(0, placeableY)
        }
    }
)
```

To verify this works as expected, you can use this modifier on a `Text` as you saw in the picture above:

```
@Preview
@Composable
fun TextWithPaddingToBaselinePreview() {
  LayoutsCodelabTheme {
    Text("Hi there!", Modifier.firstBaselineToTop(32.dp))
  }
}

@Preview
@Composable
fun TextWithNormalPaddingPreview() {
  LayoutsCodelabTheme {
    Text("Hi there!", Modifier.padding(top = 32.dp))
  }
}
```

With preview:

![dccb4473e2ca09c6.png](https://developer.android.com/codelabs/jetpack-compose-layouts/img/dccb4473e2ca09c6.png)

## Using the Layout composable

Instead of controlling how a single composable gets measured and laid out on the screen, you might have the same necessity for a group of composables. For that, you can use the [`Layout`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main-release:compose/ui/ui/src/commonMain/kotlin/androidx/compose/ui/layout/Layout.kt) composable to manually control how to measure and position the layout's children. Usually, the common structure of a composable that uses `Layout` is as follows:

```
@Composable
fun CustomLayout(
    modifier: Modifier = Modifier,
    // custom layout attributes 
    content: @Composable () -> Unit
) {
    Layout(
        modifier = modifier,
        content = content
    ) { measurables, constraints ->
        // measure and position children given constraints logic here
    }
}
```

The minimum required parameters for a `CustomLayout` are a `modifier` and `content`; these parameters are then passed to `Layout`. In the trailing lambda of `Layout` (of type `MeasurePolicy`), you get the same lambda parameters as you get with the `layout` modifier.

To show `Layout` in action, let's start implementing a very basic `Column` using `Layout` to understand the API. Later, we'll build something more complex to showcase flexibility of the `Layout` composable.

## Implementing a basic Column

Our custom implementation of `Column` **lays out items vertically**. Also, for simplicity, our layout **occupies as much space as it can in its parent**.

Create a new composable called `MyOwnColumn` and add the common structure of a `Layout` composable:

```
@Composable
fun MyOwnColumn(
    modifier: Modifier = Modifier,
    content: @Composable () -> Unit
) {
    Layout(
        modifier = modifier,
        content = content
    ) { measurables, constraints ->
        // measure and position children given constraints logic here
    }
}
```

As before, the first thing to do is measure our children that can only be measured once. Similarly to how the layout modifier works, in the `measurables` lambda parameter, you get all the `content` that you can measure by calling `measurable.measure(constraints)`.

For this use case, you won't constrain our child views further. When measuring the children, you should also keep track of the `width` and the maximum `height` of each row to be able to place them correctly on the screen later:

```
@Composable
fun MyOwnColumn(
    modifier: Modifier = Modifier,
    content: @Composable () -> Unit
) {
    Layout(
        modifier = modifier,
        content = content
    ) { measurables, constraints ->

        // Don't constrain child views further, measure them with given constraints
        // List of measured children
        val placeables = measurables.map { measurable ->
            // Measure each child
            measurable.measure(constraints)
        }
    }
}
```

Now that you have the list of measured children in our logic, before positioning them on the screen, you need to calculate the size of our version of `Column`. As you're making it as big as its parent, the size of it is the constraints passed in by the parent. Specify the size of our own `Column` by calling the `layout(width, height)` method, which also gives you the lambda used for placing the children:

```
@Composable
fun MyOwnColumn(
    modifier: Modifier = Modifier,
    content: @Composable () -> Unit
) {
    Layout(
        modifier = modifier,
        content = content
    ) { measurables, constraints ->
        // Measure children - code in the previous code snippet
        ...

        // Set the size of the layout as big as it can
        layout(constraints.maxWidth, constraints.maxHeight) {
            // Place children
        }
    }
}
```

Lastly, we position our children on the screen by calling `placeable.placeRelative(x, y)`. In order to place the children vertically, we keep track of the `y` coordinate we have placed children up to. The final code of `MyOwnColumn` looks like this:

```
@Composable
fun MyOwnColumn(
    modifier: Modifier = Modifier,
    content: @Composable () -> Unit
) {
    Layout(
        modifier = modifier,
        content = content
    ) { measurables, constraints ->
        // Don't constrain child views further, measure them with given constraints
        // List of measured children
        val placeables = measurables.map { measurable ->
            // Measure each child
            measurable.measure(constraints)
        }

        // Track the y co-ord we have placed children up to
        var yPosition = 0

        // Set the size of the layout as big as it can
        layout(constraints.maxWidth, constraints.maxHeight) {
            // Place children in the parent layout
            placeables.forEach { placeable ->
                // Position item on the screen
                placeable.placeRelative(x = 0, y = yPosition)

                // Record the y co-ord placed up to
                yPosition += placeable.height
            }
        }
    }
}
```

## MyOwnColumn in action

Let's see `MyOwnColumn` on the screen by using it in the `BodyContent` composable. Replace the content inside BodyContent with the following:

```
@Composable
fun BodyContent(modifier: Modifier = Modifier) {
    MyOwnColumn(modifier.padding(8.dp)) {
        Text("MyOwnColumn")
        Text("places items")
        Text("vertically.")
        Text("We've done it by hand!")
    }
}
```

With preview:

![e69cdb015e4d8abe.png](https://developer.android.com/codelabs/jetpack-compose-layouts/img/e69cdb015e4d8abe.png)