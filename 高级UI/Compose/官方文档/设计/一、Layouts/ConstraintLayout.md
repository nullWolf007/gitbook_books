## ConstraintLayout

`ConstraintLayout` can help you place composables relative to others on the screen and is an alternative to using multiple `Row`s, `Column`s and `Box`es. ConstraintLayout is useful when implementing larger layouts with more complicated alignment requirements.

**Note:** In the View system, [`ConstraintLayout`](https://developer.android.com/training/constraint-layout) was the recommended way to create large and complex layouts as the flat view hierarchy was better for performance. However, this is not a concern in Compose, which is able to efficiently handle deep layout hierarchies.

You can find the Compose Constraint Layout dependency in your project's `build.gradle` file:

```
// build.gradle
implementation "androidx.constraintlayout:constraintlayout-compose:1.0.0-alpha07"
```

`ConstraintLayout` in Compose works with a [DSL](https://kotlinlang.org/docs/reference/type-safe-builders.html):

- References are created using `createRefs()` (or `createRef()`) and each composable in `ConstraintLayout` needs to have a reference associated.
- Constraints are provided using the `constrainAs` modifier which takes the reference as a parameter and lets you specify its constraints in the body lambda.
- Constraints are specified using `linkTo` or other helpful methods.
- `parent` is an existing reference that can be used to specify constraints towards the `ConstraintLayout` composable itself.

Let's start with a simple example:

```
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

@Preview
@Composable
fun ConstraintLayoutContentPreview() {
    LayoutsCodelabTheme {
        ConstraintLayoutContent()
    }
}
```

This constrains the top of the `Button` to the parent with a margin of `16.dp` and a `Text` to the bottom of the `Button` also with a margin of `16.dp`.

![72fcb81ab2c0483c.png](https://developer.android.com/codelabs/jetpack-compose-layouts/img/72fcb81ab2c0483c.png)

If we wanted to center the text horizontally, we can use the `centerHorizontallyTo` function that sets both the `start` and `end` of the `Text` to the edges of the `parent`:

```
@Composable
fun ConstraintLayoutContent() {
    ConstraintLayout {
        ... // Same as before

        Text("Text", Modifier.constrainAs(text) {
            top.linkTo(button.bottom, margin = 16.dp)
            // Centers Text horizontally in the ConstraintLayout
            centerHorizontallyTo(parent)
        })
    }
}
```

With preview:

![729a1b4c03f1f187.png](https://developer.android.com/codelabs/jetpack-compose-layouts/img/729a1b4c03f1f187.png)

`ConstraintLayout`'s size will be as small as possible to wrap its content. That's why it seems `Text` is centered around the `Button` instead of the parent. If other sizing behavior is desired, sizing modifiers (e.g. `fillMaxSize`, `size`) should be applied to the `ConstraintLayout` composable as with any other layout in Compose.

## Helpers

The DSL also supports creating guidelines, barriers and chains. For example:

```
@Composable
fun ConstraintLayoutContent() {
    ConstraintLayout {
        // Creates references for the three composables
        // in the ConstraintLayout's body
        val (button1, button2, text) = createRefs()

        Button(
            onClick = { /* Do something */ },
            modifier = Modifier.constrainAs(button1) {
                top.linkTo(parent.top, margin = 16.dp)
            }
        ) { 
            Text("Button 1") 
        }

        Text("Text", Modifier.constrainAs(text) {
            top.linkTo(button1.bottom, margin = 16.dp)
            centerAround(button1.end)
        })

        val barrier = createEndBarrier(button1, text)
        Button(
            onClick = { /* Do something */ },
            modifier = Modifier.constrainAs(button2) {
                top.linkTo(parent.top, margin = 16.dp)
                start.linkTo(barrier)
            }
        ) { 
            Text("Button 2") 
        }
    }
}
```

With preview:

![a4117576ef1768a2.png](https://developer.android.com/codelabs/jetpack-compose-layouts/img/a4117576ef1768a2.png)

Note that

- barriers (and all the other helpers) can be created in the body of `ConstraintLayout`, but not inside `constrainAs`.
- `linkTo` can be used to constrain with guidelines and barriers the same way it works for edges of layouts.

## Customizing dimensions

By default, the children of `ConstraintLayout` will be allowed to choose the size they need to wrap their content. For example, this means that a Text is able to go outside the screen bounds when the text is too long:

```
@Composable
fun LargeConstraintLayout() {
    ConstraintLayout {
        val text = createRef()

        val guideline = createGuidelineFromStart(fraction = 0.5f)
        Text(
            "This is a very very very very very very very long text",
            Modifier.constrainAs(text) {
                linkTo(start = guideline, end = parent.end)
            }
        )
    }
}

@Preview
@Composable
fun LargeConstraintLayoutPreview() {
    LayoutsCodelabTheme {
        LargeConstraintLayout()
    }
}
```

![616c19b971811cfa.png](https://developer.android.com/codelabs/jetpack-compose-layouts/img/616c19b971811cfa.png)

Obviously, you'd like the text to line break in the available space. To achieve this, we can change the `width` behavior of the text:

```
@Composable
fun LargeConstraintLayout() {
    ConstraintLayout {
        val text = createRef()

        val guideline = createGuidelineFromStart(0.5f)
        Text(
            "This is a very very very very very very very long text",
            Modifier.constrainAs(text) {
                linkTo(guideline, parent.end)
                width = Dimension.preferredWrapContent
            }
        )
    }
}
```

With preview:

![fc41cacd547bbea.png](https://developer.android.com/codelabs/jetpack-compose-layouts/img/fc41cacd547bbea.png)

Available `Dimension` behaviors are:

- `preferredWrapContent` - the layout is wrap content, subject to the constraints in that dimension.
- `wrapContent` - the layout is wrap content even if the constraints would not allow it.
- `fillToConstraints` - the layout will expand to fill the space defined by its constraints in that dimension.
- `preferredValue` - the layout is a fixed dp value, subject to the constraints in that dimension.
- `value` - the layout is a fixed dp value, regardless of the constraints in that dimension

Also, certain `Dimension`s can be coerced:

```
width = Dimension.preferredWrapContent.atLeast(100.dp)
```

## Decoupled API

So far, in the examples, constraints have been specified inline, with a modifier in the composable they're applied to. However, there are cases when keeping the constraints decoupled from the layouts they apply to is valuable: the common example is for easily changing the constraints based on the screen configuration or animating between 2 constraint sets.

For these cases, you can use `ConstraintLayout` in a different way:

1. Pass in a `ConstraintSet` as a parameter to `ConstraintLayout`.
2. Assign references created in the `ConstraintSet` to composables using the `layoutId` modifier.

This API shape applied to the first `ConstraintLayout` example shown above, optimized for the width of the screen, looks like this:

```
@Composable
fun DecoupledConstraintLayout() {
    BoxWithConstraints {
        val constraints = if (maxWidth < maxHeight) {
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
            top.linkTo(parent.top, margin= margin)
        }
        constrain(text) {
            top.linkTo(button.bottom, margin)
        }
    }
}
```