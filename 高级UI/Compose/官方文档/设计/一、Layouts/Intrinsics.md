## [Intrinsics](https://developer.android.com/courses/pathways/compose#10)

One of the rules of Compose is that you should only measure your children once; measuring children twice throws a runtime exception. However, there are times when you need some information about your children before measuring them.

**Intrinsics lets you query children before they're actually measured**.

To a composable, you can ask for its `intrinsicWidth` or `intrinsicHeight`:

- `(min|max)IntrinsicWidth`: Given this height, what's the minimum/maximum width you can paint your content properly.
- `(min|max)IntrinsicHeight`: Given this width, what's the minimum/maximum height you can paint your content properly.

For example, if you ask the `minIntrinsicHeight` of a `Text` with infinite `width`, it'll return the `height` of the `Text` as if the text was drawn in a single line.

## Intrinsics in action

Imagine that we want to create a composable that displays two texts on the screen separated by a divider like this:

![835f0b8c9f07cd9.png](https://developer.android.com/codelabs/jetpack-compose-layouts/img/835f0b8c9f07cd9.png)

How can we do this? We can have a `Row` with two `Text`s inside that expands as much as they can and a `Divider` in the middle. We want the Divider to be as tall as the tallest `Text` and thin (`width = 1.dp`).

```kotlin
@Composable
fun TwoTexts(modifier: Modifier = Modifier, text1: String, text2: String) {
    Row(modifier = modifier) {
        Text(
            modifier = Modifier
                .weight(1f)
                .padding(start = 4.dp)
                .wrapContentWidth(Alignment.Start),
            text = text1
        )

        Divider(color = Color.Black, modifier = Modifier.fillMaxHeight().width(1.dp))
        Text(
            modifier = Modifier
                .weight(1f)
                .padding(end = 4.dp)
                .wrapContentWidth(Alignment.End),

            text = text2
        )
    }
}

@Preview
@Composable
fun TwoTextsPreview() {
    LayoutsCodelabTheme {
        Surface {
            TwoTexts(text1 = "Hi", text2 = "there")
        }
    }
}
```

If we preview this, we see that the Divider expands to the whole screen and that's not what we want:

![d61f179394ded825.png](https://developer.android.com/codelabs/jetpack-compose-layouts/img/d61f179394ded825.png)

This happens because `Row` measures each child individually and the height of `Text` cannot be used to constraint the `Divider`. We want the `Divider` to fill the available space with a given height. For that, we can use the `height(IntrinsicSize.Min)` modifier .

`height(IntrinsicSize.Min)` sizes its children being forced to be as tall as their minimum intrinsic height. As it's recursive, it'll query `Row` and its children `minIntrinsicHeight`.

Applying that to our code, it'll work as expected

```kotlin
@Composable
fun TwoTexts(modifier: Modifier = Modifier, text1: String, text2: String) {
    Row(modifier = modifier.height(IntrinsicSize.Min)) {
        Text(
            modifier = Modifier
                .weight(1f)
                .padding(start = 4.dp)
                .wrapContentWidth(Alignment.Start),
            text = text1
        )

        Divider(color = Color.Black, modifier = Modifier.fillMaxHeight().width(1.dp))
        Text(
            modifier = Modifier
                .weight(1f)
                .padding(end = 4.dp)
                .wrapContentWidth(Alignment.End),
            text = text2
        )
    }
}

@Preview
@Composable
fun TwoTextsPreview() {
    LayoutsCodelabTheme {
        Surface {
            TwoTexts(text1 = "Hi", text2 = "there")
        }
    }
}
```

With preview:

![835f0b8c9f07cd9.png](https://developer.android.com/codelabs/jetpack-compose-layouts/img/835f0b8c9f07cd9.png)

Row's `minIntrinsicHeight` will be the maximum `minIntrinsicHeight` of its children. Divider's `minIntrinsicHeight` is 0 as it doesn't occupy space if no constraints are given; Text's `minIntrinsicHeight` will be that of the text given a specific `width`. Therefore, Row's `height` constraint will be the max `minIntrinsicHeight` of the `Text`s. `Divider` will then expand its `height` to the `height` constraint given by Row.

## DIY

Whenever you are creating your custom layout, you can modify how intrinsics are calculated with the `(min|max)Intrinsic(Width|Height)MeasurePolicy` [parameters](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main-release:compose/ui/ui/src/commonMain/kotlin/androidx/compose/ui/layout/Layout.kt); however, the defaults should be enough most of the time.

Also, you can modify intrinsics with modifiers overriding the `Density.(min|max)Intrinsic(Width|Height)Of` methods of the [Modifier](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main-release:compose/ui/ui/src/commonMain/kotlin/androidx/compose/ui/layout/LayoutModifier.kt) interface which also have a good default.