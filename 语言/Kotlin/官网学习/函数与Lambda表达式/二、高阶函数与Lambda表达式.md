[TOC]

# 高阶函数与 lambda 表达式

* Kotlin 函数都是[*头等的*](https://zh.wikipedia.org/wiki/头等函数)，这意味着它们可以存储在变量与数据结构中、作为参数传递给其他[高阶函数](https://www.kotlincn.net/docs/reference/lambdas.html#高阶函数)以及从其他高阶函数返回。可以像操作任何其他非函数值一样操作函数。

* 为促成这点，作为一门静态类型编程语言的 Kotlin 使用一系列[函数类型](https://www.kotlincn.net/docs/reference/lambdas.html#函数类型)来表示函数并提供一组特定的语言结构，例如 [lambda 表达式](https://www.kotlincn.net/docs/reference/lambdas.html#lambda-表达式与匿名函数)。

## 高阶函数

* 高阶函数是将函数用作参数或返回值的函数。

* 一个不错的示例是集合的[函数式风格的 `fold`](https://en.wikipedia.org/wiki/Fold_(higher-order_function))， 它接受一个初始累积值与一个接合函数，并通过将当前累积值与每个集合元素连续接合起来代入累积值来构建返回值：

```kotlin
fun <T, R> Collection<T>.fold(
    initial: R, 
    combine: (acc: R, nextElement: T) -> R
): R {
    var accumulator: R = initial
    for (element: T in this) {
        accumulator = combine(accumulator, element)
    }
    return accumulator
}
```

* 在上述代码中，参数 `combine` 具有[函数类型](https://www.kotlincn.net/docs/reference/lambdas.html#函数类型) `(R, T) -> R`，因此 `fold` 接受一个函数作为参数， 该函数接受类型分别为 `R` 与 `T` 的两个参数并返回一个 `R` 类型的值。 在 *for*-循环内部调用该函数，然后将其返回值赋值给 `accumulator`。

* 为了调用 `fold`，需要传给它一个[函数类型的实例](https://www.kotlincn.net/docs/reference/lambdas.html#函数类型实例化)作为参数，而在高阶函数调用处，（[下文详述的](https://www.kotlincn.net/docs/reference/lambdas.html#lambda-表达式与匿名函数)）lambda 表达 式广泛用于此目的。

```kotlin
val items = listOf(1, 2, 3, 4, 5)

// Lambdas 表达式是花括号括起来的代码块。
items.fold(0, { 
    // 如果一个 lambda 表达式有参数，前面是参数，后跟“->”
    acc: Int, i: Int -> 
    print("acc = $acc, i = $i, ") 
    val result = acc + i
    println("result = $result")
    // lambda 表达式中的最后一个表达式是返回值：
    result
})

// lambda 表达式的参数类型是可选的，如果能够推断出来的话：
val joinedToString = items.fold("Elements:", { acc, i -> acc + " " + i })
println(joinedToString)

// 函数引用也可以用于高阶函数调用：
val product = items.fold(1, Int::times)
println(product)

//输出
acc = 0, i = 1, result = 1
acc = 1, i = 2, result = 3
acc = 3, i = 3, result = 6
acc = 6, i = 4, result = 10
acc = 10, i = 5, result = 15
Elements: 1 2 3 4 5
120
```

* 以下各节会更详细地解释上文提到的这些概念。

## 函数类型

* Kotlin 使用类似 `(Int) -> String` 的一系列函数类型来处理函数的声明： `val onClick: () -> Unit = ……`。

* 这些类型具有与函数签名相对应的特殊表示法，即它们的参数和返回值：
  * 所有函数类型都有一个圆括号括起来的参数类型列表以及一个返回类型：`(A, B) -> C` 表示接受类型分别为 `A` 与 `B` 两个参数并返回一个 `C` 类型值的函数类型。 参数类型列表可以为空，如 `() -> A`。[`Unit` 返回类型](https://www.kotlincn.net/docs/reference/functions.html#返回-unit-的函数)不可省略。
  * 函数类型可以有一个额外的*接收者*类型，它在表示法中的点之前指定： 类型 `A.(B) -> C` 表示可以在 `A` 的接收者对象上以一个 `B` 类型参数来调用并返回一个 `C` 类型值的函数。 [带有接收者的函数字面值](https://www.kotlincn.net/docs/reference/lambdas.html#带有接收者的函数字面值)通常与这些类型一起使用。
  * [挂起函数](https://www.kotlincn.net/docs/reference/coroutines.html#挂起函数)属于特殊种类的函数类型，它的表示法中有一个 *suspend* 修饰符 ，例如 `suspend () -> Unit` 或者 `suspend A.(B) -> C`。

* 函数类型表示法可以选择性地包含函数的参数名：`(x: Int, y: Int) -> Point`。 这些名称可用于表明参数的含义。

> 如需将函数类型指定为[可空](https://www.kotlincn.net/docs/reference/null-safety.html#可空类型与非空类型)，请使用圆括号：`((Int, Int) -> Int)?`。
>
> 函数类型可以使用圆括号进行接合：`(Int) -> ((Int) -> Unit)`
>
> 箭头表示法是右结合的，`(Int) -> (Int) -> Unit` 与前述示例等价，但不等于 `((Int) -> (Int)) -> Unit`。

* 还可以通过使用[类型别名](https://www.kotlincn.net/docs/reference/type-aliases.html)给函数类型起一个别称：

```kotlin
typealias ClickHandler = (Button, ClickEvent) -> Unit
```

### 函数类型实例化

* 有几种方法可以获得函数类型的实例：
  * 使用函数字面值的代码块，采用以下形式之一：

    - [lambda 表达式](https://www.kotlincn.net/docs/reference/lambdas.html#lambda-表达式与匿名函数): `{ a, b -> a + b }`,
    - [匿名函数](https://www.kotlincn.net/docs/reference/lambdas.html#匿名函数): `fun(s: String): Int { return s.toIntOrNull() ?: 0 }`

    [带有接收者的函数字面值](https://www.kotlincn.net/docs/reference/lambdas.html#带有接收者的函数字面值)可用作带有接收者的函数类型的值。

  * 使用已有声明的可调用引用：

    - 顶层、局部、成员、扩展[函数](https://www.kotlincn.net/docs/reference/reflection.html#函数引用)：`::isOdd`、 `String::toInt`，
    - 顶层、成员、扩展[属性](https://www.kotlincn.net/docs/reference/reflection.html#属性引用)：`List<Int>::size`，
    - [构造函数](https://www.kotlincn.net/docs/reference/reflection.html#构造函数引用)：`::Regex`

    这包括指向特定实例成员的[绑定的可调用引用](https://www.kotlincn.net/docs/reference/reflection.html#绑定的函数与属性引用自-11-起)：`foo::toString`。

  * 使用实现函数类型接口的自定义类的实例：

```kotlin
class IntTransformer: (Int) -> Int {
    override operator fun invoke(x: Int): Int = TODO()
}

val intFunction: (Int) -> Int = IntTransformer()
```

* 如果有足够信息，编译器可以推断变量的函数类型：

```kotlin
val a = { i: Int -> i + 1 } // 推断出的类型是 (Int) -> Int
```

* 带与不带接收者的函数类型*非字面*值可以互换，其中接收者可以替代第一个参数，反之亦然。例如，`(A, B) -> C` 类型的值可以传给或赋值给期待 `A.(B) -> C` 的地方，反之亦然：

```kotlin
val repeatFun: String.(Int) -> String = { times -> this.repeat(times) }
val twoParameters: (String, Int) -> String = repeatFun // OK

fun runTransformation(f: (String, Int) -> String): String {
    return f("hello", 3)
}
val result = runTransformation(repeatFun) // OK

```

> 请注意，默认情况下推断出的是没有接收者的函数类型，即使变量是通过扩展函数引用来初始化的。 如需改变这点，请显式指定变量类型。

### 函数类型实例调用

* 函数类型的值可以通过其 [`invoke(……)` 操作符](https://www.kotlincn.net/docs/reference/operator-overloading.html#invoke)调用：`f.invoke(x)` 或者直接 `f(x)`。

* 如果该值具有接收者类型，那么应该将接收者对象作为第一个参数传递。 调用带有接收者的函数类型值的另一个方式是在其前面加上接收者对象， 就好比该值是一个[扩展函数](https://www.kotlincn.net/docs/reference/extensions.html)：`1.foo(2)`，

* 例如：

```kotlin
val stringPlus: (String, String) -> String = String::plus
val intPlus: Int.(Int) -> Int = Int::plus

println(stringPlus.invoke("<-", "->"))
println(stringPlus("Hello, ", "world!")) 

println(intPlus.invoke(1, 1))
println(intPlus(1, 2))
println(2.intPlus(3)) // 类扩展调用

//plus表示+
//String::plus表示拼接字符串
//Int::plus表示相加
<-->
Hello, world!
2
3
5
```

### 内联函数

* 有时使用[内联函数](https://www.kotlincn.net/docs/reference/inline-functions.html)可以为高阶函数提供灵活的控制流。

## Lambda 表达式与匿名函数

* lambda 表达式与匿名函数是“函数字面值”，即未声明的函数， 但立即做为表达式传递。考虑下面的例子：

```kotlin
max(strings, { a, b -> a.length < b.length })
```

* 函数 `max` 是一个高阶函数，它接受一个函数作为第二个参数。 其第二个参数是一个表达式，它本身是一个函数，即函数字面值，它等价于以下具名函数：

```kotlin
fun compare(a: String, b: String): Boolean = a.length < b.length
```

### Lambda 表达式语法

* Lambda 表达式的完整语法形式如下：

```kotlin
val sum: (Int, Int) -> Int = { x: Int, y: Int -> x + y }
```

* lambda 表达式总是括在花括号中， 完整语法形式的参数声明放在花括号内，并有可选的类型标注， 函数体跟在一个 `->` 符号之后。如果推断出的该 lambda 的返回类型不是 `Unit`，那么该 lambda 主体中的最后一个（或可能是单个） 表达式会视为返回值。

* 如果我们把所有可选标注都留下，看起来如下：

```kotlin
val sum = { x: Int, y: Int -> x + y }
```

### 传递末尾的 lambda 表达式

* 在 Kotlin 中有一个约定：如果函数的最后一个参数是函数，那么作为相应参数传入的 lambda 表达式可以放在圆括号之外：

```kotlin
val product = items.fold(1) { acc, e -> acc * e }
```

* 这种语法也称为*拖尾 lambda 表达式*。

* 如果该 lambda 表达式是调用时唯一的参数，那么圆括号可以完全省略：

```kotlin
run { println("...") }
```

### `it`：单个参数的隐式名称

* 一个 lambda 表达式只有一个参数是很常见的。

* 如果编译器自己可以识别出签名，也可以不用声明唯一的参数并忽略 `->`。 该参数会隐式声明为 `it`：

```kotlin
ints.filter { it > 0 } // 这个字面值是“(it: Int) -> Boolean”类型的
```

### 从 lambda 表达式中返回一个值

* 我们可以使用[限定的返回](https://www.kotlincn.net/docs/reference/returns.html#返回到标签)语法从 lambda 显式返回一个值。 否则，将隐式返回最后一个表达式的值。

* 因此，以下两个片段是等价的：

```kotlin
ints.filter {
    val shouldFilter = it > 0 
    shouldFilter
}

ints.filter {
    val shouldFilter = it > 0 
    return@filter shouldFilter
}
```

* 这一约定连同[在圆括号外传递 lambda 表达式](https://www.kotlincn.net/docs/reference/lambdas.html#passing-a-lambda-to-the-last-parameter)一起支持 [LINQ-风格](https://docs.microsoft.com/en-us/previous-versions/dotnet/articles/bb308959(v=msdn.10)) 的代码：

```
strings.filter { it.length == 5 }.sortedBy { it }.map { it.toUpperCase() }
```

### 下划线用于未使用的变量（自 1.1 起）

* 如果 lambda 表达式的参数未使用，那么可以用下划线取代其名称：

```kotlin
map.forEach { _, value -> println("$value!") }
```

### 在 lambda 表达式中解构（自 1.1 起）

* 在 lambda 表达式中解构是作为[解构声明](https://www.kotlincn.net/docs/reference/multi-declarations.html#在-lambda-表达式中解构自-11-起)的一部分描述的。

### 匿名函数

* 上面提供的 lambda 表达式语法缺少的一个东西是指定函数的返回类型的能力。在大多数情况下，这是不必要的。因为返回类型可以自动推断出来。然而，如果确实需要显式指定，可以使用另一种语法： *匿名函数* 。

```kotlin
fun(x: Int, y: Int): Int = x + y
```

* 匿名函数看起来非常像一个常规函数声明，除了其名称省略了。其函数体可以是表达式（如上所示）或代码块：

```kotlin
fun(x: Int, y: Int): Int {
    return x + y
}
```

* 参数和返回类型的指定方式与常规函数相同，除了能够从上下文推断出的参数类型可以省略：

```kotlin
ints.filter(fun(item) = item > 0)
```

* 匿名函数的返回类型推断机制与正常函数一样：对于具有表达式函数体的匿名函数将自动推断返回类型，而具有代码块函数体的返回类型必须显式指定（或者已假定为 `Unit`）。

* 请注意，匿名函数参数总是在括号内传递。 允许将函数留在圆括号外的简写语法仅适用于 lambda 表达式。

* Lambda表达式与匿名函数之间的另一个区别是[非局部返回](https://www.kotlincn.net/docs/reference/inline-functions.html#非局部返回)的行为。一个不带标签的 *return* 语句总是在用 *fun* 关键字声明的函数中返回。这意味着 lambda 表达式中的 *return* 将从包含它的函数返回，而匿名函数中的 *return* 将从匿名函数自身返回。

### 闭包

* Lambda 表达式或者匿名函数（以及[局部函数](https://www.kotlincn.net/docs/reference/functions.html#局部函数)和[对象表达式](https://www.kotlincn.net/docs/reference/object-declarations.html#对象表达式)） 可以访问其 *闭包* ，即在外部作用域中声明的变量。 在 lambda 表达式中可以修改闭包中捕获的变量：

```kotlin
var sum = 0
ints.filter { it > 0 }.forEach {
    sum += it
}
print(sum)
```

### 带有接收者的函数字面值

* 带有接收者的[函数类型](https://www.kotlincn.net/docs/reference/lambdas.html#函数类型)，例如 `A.(B) -> C`，可以用特殊形式的函数字面值实例化—— 带有接收者的函数字面值。

* 如上所述，Kotlin 提供了[调用](https://www.kotlincn.net/docs/reference/lambdas.html#函数类型实例调用)带有接收者（提供*接收者对象*）的函数类型实例的能力。

* 在这样的函数字面值内部，传给调用的接收者对象成为*隐式*的*this*，以便访问接收者对象的成员而无需任何额外的限定符，亦可使用 [`this` 表达式](https://www.kotlincn.net/docs/reference/this-expressions.html) 访问接收者对象。

* 这种行为与[扩展函数](https://www.kotlincn.net/docs/reference/extensions.html)类似，扩展函数也允许在函数体内部访问接收者对象的成员。

* 这里有一个带有接收者的函数字面值及其类型的示例，其中在接收者对象上调用了 `plus` ：

```kotlin
val sum: Int.(Int) -> Int = { other -> plus(other) }
```

* 匿名函数语法允许你直接指定函数字面值的接收者类型。 如果你需要使用带接收者的函数类型声明一个变量，并在之后使用它，这将非常有用。

```kotlin
val sum = fun Int.(other: Int): Int = this + other
```

* 当接收者类型可以从上下文推断时，lambda 表达式可以用作带接收者的函数字面值。 
* One of the most important examples of their usage is [type-safe builders](https://www.kotlincn.net/docs/reference/type-safe-builders.html):

```kotlin
class HTML {
    fun body() { …… }
}

fun html(init: HTML.() -> Unit): HTML {
    val html = HTML()  // 创建接收者对象
    html.init()        // 将该接收者对象传给该 lambda
    return html
}

html {       // 带接收者的 lambda 由此开始
    body()   // 调用该接收者对象的一个方法
}
```