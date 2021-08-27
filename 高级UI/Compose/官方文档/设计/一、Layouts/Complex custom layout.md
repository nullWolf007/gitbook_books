### Complex custom layout

Once the basics of `Layout` are covered. Let's create a more complex example to showcase the flexibility of the API. We'll build the custom [Material Study Owl](https://material.io/design/material-studies/owl.html)'s staggered grid that you can see in the middle of the following picture:

![7a54fe8390fe39d2.png](https://developer.android.com/codelabs/jetpack-compose-layouts/img/7a54fe8390fe39d2.png)

Owl's staggered grid lays out items vertically, filled out a column at a time given a `n` number of rows. Doing this with a `Row` of `Columns` is not possible since you wouldn't get the staggering of the layout. Having a `Column` of `Rows` could be possible if you prepare the data so that it displays vertically.

However, the custom layout also gives you the opportunity to constrain the height of all the items in the staggered grid. So to have more control over the layout and learn how to create a custom one, we'll measure and position the children on our own.

If we wanted to make the grid reusable on different orientations, we could take as a parameter the number of rows we want to have on the screen. Since that information should come when the layout is invoked, we pass it as a parameter:

```
@Composable
fun StaggeredGrid(
    modifier: Modifier = Modifier,
    rows: Int = 3,
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

As before, the first thing to do is measure our children. Remember **you can only measure your children once**.

For our use case, we won't constrain our child views further. When measuring our children, we should also keep track of what's the `width` and the max `height` of each row:

```
Layout(
    modifier = modifier,
    content = content
) { measurables, constraints ->

    // Keep track of the width of each row
    val rowWidths = IntArray(rows) { 0 }

    // Keep track of the max height of each row
    val rowHeights = IntArray(rows) { 0 }

    // Don't constrain child views further, measure them with given constraints
    // List of measured children
    val placeables = measurables.mapIndexed { index, measurable ->

        // Measure each child
        val placeable = measurable.measure(constraints)

        // Track the width and max height of each row
        val row = index % rows
        rowWidths[row] += placeable.width
        rowHeights[row] = max(rowHeights[row], placeable.height)

        placeable
    }
    ...
}
```

Now that we have the list of measured children in our logic, before positioning them on the screen, we need to calculate the size of our grid (full `width` and `height`) . Also, since we already know the maximum height of each row, we can calculate where we'll position the elements for each row in the Y position. We save the Y positions in the `rowY` variable:

```
Layout(
    content = content,
    modifier = modifier
) { measurables, constraints ->
    ... 

    // Grid's width is the widest row
    val width = rowWidths.maxOrNull()
        ?.coerceIn(constraints.minWidth.rangeTo(constraints.maxWidth)) ?: constraints.minWidth

    // Grid's height is the sum of the tallest element of each row
    // coerced to the height constraints 
    val height = rowHeights.sumOf { it }
        .coerceIn(constraints.minHeight.rangeTo(constraints.maxHeight))

    // Y of each row, based on the height accumulation of previous rows
    val rowY = IntArray(rows) { 0 }
    for (i in 1 until rows) {
        rowY[i] = rowY[i-1] + rowHeights[i-1]
    }

    ...
}
```

Lastly, we position our children on the screen by calling `placeable.placeRelative(x, y)`. In our use case, we also keep track of the X coordinate for each row in the `rowX` variable:

```
Layout(
    content = content,
    modifier = modifier
) { measurables, constraints ->
    ... 

    // Set the size of the parent layout
    layout(width, height) {
        // x cord we have placed up to, per row
        val rowX = IntArray(rows) { 0 }

        placeables.forEachIndexed { index, placeable ->
            val row = index % rows
            placeable.placeRelative(
                x = rowX[row],
                y = rowY[row]
            )
            rowX[row] += placeable.width
        }
    }
}
```

## Using the custom StaggeredGrid in an example

Now that we have our custom grid layout that knows how to measure and position children, let's use it in our app. To simulate Owl's chips in the grid, we can easily create a composable that does something similar:

```
@Composable
fun Chip(modifier: Modifier = Modifier, text: String) {
    Card(
        modifier = modifier,
        border = BorderStroke(color = Color.Black, width = Dp.Hairline),
        shape = RoundedCornerShape(8.dp)
    ) {
        Row(
            modifier = Modifier.padding(start = 8.dp, top = 4.dp, end = 8.dp, bottom = 4.dp), 
            verticalAlignment = Alignment.CenterVertically
        ) {
            Box(
                modifier = Modifier.size(16.dp, 16.dp)
                    .background(color = MaterialTheme.colors.secondary)
            )
            Spacer(Modifier.width(4.dp))
            Text(text = text)
        }
    }
}

@Preview
@Composable
fun ChipPreview() {
    LayoutsCodelabTheme {
        Chip(text = "Hi there")
    }
}
```

With preview:

![f1f8c6bb7f12cf1.png](https://developer.android.com/codelabs/jetpack-compose-layouts/img/f1f8c6bb7f12cf1.png)

Now, let's create a list of topics that we can show in our `BodyContent` and display them in the `StaggeredGrid`:

```
val topics = listOf(
    "Arts & Crafts", "Beauty", "Books", "Business", "Comics", "Culinary",
    "Design", "Fashion", "Film", "History", "Maths", "Music", "People", "Philosophy",
    "Religion", "Social sciences", "Technology", "TV", "Writing"
)


@Composable
fun BodyContent(modifier: Modifier = Modifier) {
    StaggeredGrid(modifier = modifier) {
        for (topic in topics) {
            Chip(modifier = Modifier.padding(8.dp), text = topic)
        }
    }
}

@Preview
@Composable
fun LayoutsCodelabPreview() {
    LayoutsCodelabTheme {
        BodyContent()
    }
}
```

With preview:

![e9861768e4e27dd4.png](https://developer.android.com/codelabs/jetpack-compose-layouts/img/e9861768e4e27dd4.png)

Notice that we can change the number of rows of our grid and it still works as expected:

```
@Composable
fun BodyContent(modifier: Modifier = Modifier) {
    StaggeredGrid(modifier = modifier, rows = 5) {
        for (topic in topics) {
            Chip(modifier = Modifier.padding(8.dp), text = topic)
        }
    }
}
```

With preview:

![555f88fd41e4dff4.png](https://developer.android.com/codelabs/jetpack-compose-layouts/img/555f88fd41e4dff4.png)

As depending on the number of rows, our topics can go off the screen, we can make our `BodyContent` scrollable by just wrapping the `StaggeredGrid` in a scrollable `Row` and passing the modifier to it instead of `StaggeredGrid`.

```
@Composable
fun BodyContent(modifier: Modifier = Modifier) {
    Row(modifier = modifier.horizontalScroll(rememberScrollState())) {
        StaggeredGrid {
            for (topic in topics) {
                Chip(modifier = Modifier.padding(8.dp), text = topic)
            }
        }
    }
}
```

If you use the Interactive Preview button ![bb4c8dfe4b8debaa.png](https://developer.android.com/codelabs/jetpack-compose-layouts/img/bb4c8dfe4b8debaa.png)or run the app on the device by tapping on the Android Studio run button, you'll see how you can scroll the content horizontally.