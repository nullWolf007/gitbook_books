[TOC]

## 异步流Flow

* 挂起函数可以异步的返回单个值，但是该如何异步返回多个计算好的值呢？这正是 Kotlin 流（Flow）的用武之地。

### 一、表示多个值

* 在 Kotlin 中可以使用[集合](https://kotlinlang.org/docs/reference/collections-overview.html)来表示多个值。 比如说，我们有一个 `simple` 函数，它返回一个包含三个数字的 [List](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/-list/index.html)， 然后使用 [forEach](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/for-each.html) 打印它们：

```kotlin
fun simple(): List<Int> = listOf(1, 2, 3)
 
fun main() {
    simple().forEach { value -> println(value) } 
}
```

* 这段代码输出如下：

```
1
2
3
```

#### 1.1 序列

* 如果使用一些消耗 CPU 资源的阻塞代码计算数字（每次计算需要 100 毫秒）那么我们可以使用 [Sequence](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.sequences/index.html) 来表示数字：

```kotlin
fun simple(): Sequence<Int> = sequence { // 序列构建器
    for (i in 1..3) {
        Thread.sleep(100) // 假装我们正在计算
        yield(i) // 产生下一个值
    }
}

fun main() {
    simple().forEach { value -> println(value) } 
}
```

* 这段代码输出相同的数字，但在打印每个数字之前等待 100 毫秒。

#### 1.2 挂起函数

* 然而，计算过程阻塞运行该代码的主线程。 当这些值由异步代码计算时，我们可以使用 `suspend` 修饰符标记函数 `simple`， 这样它就可以在不阻塞的情况下执行其工作并将结果作为列表返回：

```kotlin
suspend fun simple(): List<Int> {
    delay(1000) // 假装我们在这里做了一些异步的事情
    return listOf(1, 2, 3)
}

fun main() = runBlocking<Unit> {
    simple().forEach { value -> println(value) } 
}
```

* 这段代码将会在等待一秒之后打印数字。

#### 1.3 流

* 使用 List 结果类型，意味着我们只能一次返回所有值。 为了表示异步计算的值流（stream），我们可以使用 Flow 类型（正如同步计算值会使用 Sequence 类型）：

```kotlin
fun simple(): Flow<Int> = flow { // 流构建器
    for (i in 1..3) {
        delay(100) // 假装我们在这里做了一些有用的事情
        emit(i) // 发送下一个值
    }
}

fun main() = runBlocking<Unit> {
    // 启动并发的协程以验证主线程并未阻塞
    launch {
        for (k in 1..3) {
            println("I'm not blocked $k")
            delay(100)
        }
    }
    // 收集这个流
    simple().collect { value -> println(value) } 
}
```

这段代码在不阻塞主线程的情况下每等待 100 毫秒打印一个数字。在主线程中运行一个单独的协程每 100 毫秒打印一次 “I'm not blocked” 已经经过了验证。

```
I'm not blocked 1
1
I'm not blocked 2
2
I'm not blocked 3
3
```

* 注意使用 [Flow](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-flow/index.html) 的代码与先前示例的下述区别：
  * 名为 [flow](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-flow/index.html) 的 [Flow](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flow.html) 类型构建器函数。
  * `flow { ... }` 构建块中的代码可以挂起。
  * 函数 `simple` 不再标有 `suspend` 修饰符。
  * 流使用 [emit](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-flow-collector/emit.html) 函数 *发射* 值。
  * 流使用 [collect](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/collect.html) 函数 *收集* 值。

> 我们可以在 `simple` 的 `flow { ... }` 函数体内使用 `Thread.sleep` 代替 [delay](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/delay.html) 以观察主线程在本案例中被阻塞了。

### 二、流是冷的

* Flow 是一种类似于序列的冷流 — 这段 [flow](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flow.html) 构建器中的代码直到流被收集collect的时候才运行。这在以下的示例中非常明显：

```kotlin
fun simple(): Flow<Int> = flow { 
    println("Flow started")
    for (i in 1..3) {
        delay(100)
        emit(i)
    }
}

fun main() = runBlocking<Unit> {
    println("Calling simple function...")
    val flow = simple()
    println("Calling collect...")
    flow.collect { value -> println(value) } 
    println("Calling collect again...")
    flow.collect { value -> println(value) } 
}
```

* 打印如下：

```
Calling simple function...
Calling collect...
Flow started
1
2
3
Calling collect again...
Flow started
1
2
3
```

* 这是返回一个流的 `simple` 函数没有标记 `suspend` 修饰符的主要原因。 通过它自己，`simple()` 调用会尽快返回且不会进行任何等待。该流在每次收集的时候启动， 这就是为什么当我们再次调用 `collect` 时我们会看到“Flow started”。

### 三、流取消基础

* 流采用与协程同样的协作取消。像往常一样，流的收集可以在当流在一个可取消的挂起函数（例如 [delay](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/delay.html)）中挂起的时候取消。 以下示例展示了当 [withTimeoutOrNull](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/with-timeout-or-null.html) 块中代码在运行的时候流是如何在超时的情况下取消并停止执行其代码的：

```kotlin
fun simple(): Flow<Int> = flow { 
    for (i in 1..3) {
        delay(100)          
        println("Emitting $i")
        emit(i)
    }
}

fun main() = runBlocking<Unit> {
    withTimeoutOrNull(250) { // 在 250 毫秒后超时
        simple().collect { value -> println(value) } 
    }
    println("Done")
}
```

* 注意，在 `simple` 函数中流仅发射两个数字，产生以下输出：

```
Emitting 1
1
Emitting 2
2
Done
```

* See [Flow cancellation checks](https://www.kotlincn.net/docs/reference/coroutines/flow.html#flow-cancellation-checks) section for more details.

### 四、流构建器

* 先前示例中的 `flow { ... }` 构建器是最基础的一个。还有其他构建器使流的声明更简单：
  * [flowOf](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flow-of.html) 构建器定义了一个发射固定值集的流。
  * 使用 `.asFlow()` 扩展函数，可以将各种集合与序列转换为流。

* 因此，从流中打印从 1 到 3 的数字的示例可以写成：

```kotlin
// 将一个整数区间转化为流
(1..3).asFlow().collect { value -> println(value) }
```

### 五、过渡流操作符

* 可以使用操作符转换流，就像使用集合与序列一样。 过渡操作符应用于上游流，并返回下游流。 这些操作符也是冷操作符，就像流一样。这类操作符本身不是挂起函数。它运行的速度很快，返回新的转换流的定义。

* 基础的操作符拥有相似的名字，比如 [map](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/map.html) 与 [filter](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/filter.html)。 流与序列的主要区别在于这些操作符中的代码可以调用挂起函数。

* 举例来说，一个请求中的流可以使用 [map](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/map.html) 操作符映射出结果，即使执行一个长时间的请求操作也可以使用挂起函数来实现：

```kotlin
suspend fun performRequest(request: Int): String {
    delay(1000) // 模仿长时间运行的异步工作
    return "response $request"
}

fun main() = runBlocking<Unit> {
    (1..3).asFlow() // 一个请求流
        .map { request -> performRequest(request) }
        .collect { response -> println(response) }
}
```

* 它产生以下三行，每一行每秒出现一次：

```
response 1
response 2
response 3
```

#### 5.1 转换操作符

* 在流转换操作符中，最通用的一种称为 [transform](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/transform.html)。它可以用来模仿简单的转换，例如 [map](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/map.html) 与 [filter](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/filter.html)，以及实施更复杂的转换。 使用 `transform` 操作符，我们可以 [发射](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-flow-collector/emit.html) 任意值任意次。

* 比如说，使用 `transform` 我们可以在执行长时间运行的异步请求之前发射一个字符串并跟踪这个响应：

```kotlin
suspend fun performRequest(request: Int): String {
    delay(1000) // 模仿长时间运行的异步任务
    return "response $request"
}

fun main() = runBlocking<Unit> {
    (1..3).asFlow() // 一个请求流
        .transform { request ->
            emit("Making request $request")
            emit(performRequest(request))
        }
        .collect { response -> println(response) }
}
```

* 这段代码的输出如下：

```
Making request 1
response 1
Making request 2
response 2
Making request 3
response 3
```

#### 5.2 限长操作符

* 限长过渡操作符（例如 [take](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/take.html)）在流触及相应限制的时候会将它的执行取消。协程中的取消操作总是通过抛出异常来执行，这样所有的资源管理函数（如 `try {...} finally {...}` 块）会在取消的情况下正常运行：

```kotlin
fun numbers(): Flow<Int> = flow {
    try {                          
        emit(1)
        emit(2) 
        println("This line will not execute")
        emit(3)    
    } finally {
        println("Finally in numbers")
    }
}

fun main() = runBlocking<Unit> {
    numbers() 
        .take(2) // 只获取前两个
        .collect { value -> println(value) }
}            
```

* 这段代码的输出清楚地表明，`numbers()` 函数中对 `flow {...}` 函数体的执行在发射出第二个数字后停止：

```
1
2
Finally in numbers
```

### 六、末端流操作符

* 末端操作符是在流上用于启动流收集的*挂起函数*。 [collect](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/collect.html) 是最基础的末端操作符，但是还有另外一些更方便使用的末端操作符：
  * 转化为各种集合，例如 [toList](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/to-list.html) 与 [toSet](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/to-set.html)。
  * 获取第一个（[first](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/first.html)）值与确保流发射单个（[single](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/single.html)）值的操作符。
  * 使用 [reduce](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/reduce.html) 与 [fold](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/fold.html) 将流规约到单个值。

* 举例来说：

```kotlin
fun main() = runBlocking<Unit> {
    val sum = (1..5).asFlow()
        .map { it * it } // 数字 1 至 5 的平方                        
        .reduce { a, b -> a + b } // 求和（末端操作符）
    println(sum)     
}
```

* 打印单个数字：

```
55
```

### 七、流是连续的

* 流的每次单独收集都是按顺序执行的，除非进行特殊操作的操作符使用多个流。该收集过程直接在协程中运行，该协程调用末端操作符。 默认情况下不启动新协程。 从上游到下游每个过渡操作符都会处理每个发射出的值然后再交给末端操作符。

* 请参见以下示例，该示例过滤偶数并将其映射到字符串：

```kotlin
fun main() = runBlocking<Unit> {
    (1..5).asFlow()
        .filter {
            println("Filter $it")
            it % 2 == 0              
        }              
        .map { 
            println("Map $it")
            "string $it"
        }.collect { 
            println("Collect $it")
        }                      
}
```

* 执行：

```
Filter 1
Filter 2
Map 2
Collect string 2
Filter 3
Filter 4
Map 4
Collect string 4
Filter 5
```

### 八、流上下文

* 流的收集总是在调用协程的上下文中发生。例如，如果有一个流 `simple`，然后以下代码在它的编写者指定的上下文中运行，而无论流 `simple` 的实现细节如何：

```kotlin
withContext(context) {
    simple().collect { value ->
        println(value) // 运行在指定上下文中
    }
}
```

* 流的该属性称为 *上下文保存* 。

* 所以默认的，`flow { ... }` 构建器中的代码运行在相应流的收集器提供的上下文中。举例来说，考虑打印线程的一个 `simple` 函数的实现， 它被调用并发射三个数字：

```kotlin
fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")
           
fun simple(): Flow<Int> = flow {
    log("Started simple flow")
    for (i in 1..3) {
        emit(i)
    }
}  

fun main() = runBlocking<Unit> {
    simple().collect { value -> log("Collected $value") } 
}  
```

* 运行这段代码：

```
[main @coroutine#1] Started simple flow
[main @coroutine#1] Collected 1
[main @coroutine#1] Collected 2
[main @coroutine#1] Collected 3
```

* 由于 `simple().collect` 是在主线程调用的，那么 `simple` 的流主体也是在主线程调用的。 这是快速运行或异步代码的理想默认形式，它不关心执行的上下文并且不会阻塞调用者。

#### 8.1 withContext 发出错误

* 然而，长时间运行的消耗 CPU 的代码也许需要在 [Dispatchers.Default](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-default.html) 上下文中执行，并且更新 UI 的代码也许需要在 [Dispatchers.Main](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-main.html) 中执行。通常，[withContext](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/with-context.html) 用于在 Kotlin 协程中改变代码的上下文，但是 `flow {...}` 构建器中的代码必须遵循上下文保存属性，并且不允许从其他上下文中发射（[emit](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-flow-collector/emit.html)）。

* 尝试运行下面的代码：

```kotlin
fun simple(): Flow<Int> = flow {
    // 在流构建器中更改消耗 CPU 代码的上下文的错误方式
    kotlinx.coroutines.withContext(Dispatchers.Default) {
        for (i in 1..3) {
            Thread.sleep(100) // 假装我们以消耗 CPU 的方式进行计算
            emit(i) // 发射下一个值
        }
    }
}

fun main() = runBlocking<Unit> {
    simple().collect { value -> println(value) } 
}            
```

* 这段代码产生如下的异常：

```
Exception in thread "main" java.lang.IllegalStateException: Flow invariant is violated:
        Flow was collected in [CoroutineId(1), "coroutine#1":BlockingCoroutine{Active}@5511c7f8, BlockingEventLoop@2eac3323],
        but emission happened in [CoroutineId(1), "coroutine#1":DispatchedCoroutine{Active}@2dae0000, Dispatchers.Default].
        Please refer to 'flow' documentation or use 'flowOn' instead
    at ...
```

#### 8.2 flowOn 操作符

* 例外的是 [flowOn](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flow-on.html) 函数，该函数用于更改流发射的上下文。 以下示例展示了更改流上下文的正确方法，该示例还通过打印相应线程的名字以展示它们的工作方式：

```kotlin
fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")

fun simple(): Flow<Int> = flow {
    for (i in 1..3) {
        Thread.sleep(100) // 假装我们以消耗 CPU 的方式进行计算
        log("Emitting $i")
        emit(i) // 发射下一个值
    }
}.flowOn(Dispatchers.Default) // 在流构建器中改变消耗 CPU 代码上下文的正确方式

fun main() = runBlocking<Unit> {
    simple().collect { value ->
        log("Collected $value")
    }
}   
```

* 输出结果

```
[DefaultDispatcher-worker-1 @coroutine#2] Emitting 1
[main @coroutine#1] Collected 1
[DefaultDispatcher-worker-1 @coroutine#2] Emitting 2
[main @coroutine#1] Collected 2
[DefaultDispatcher-worker-1 @coroutine#2] Emitting 3
[main @coroutine#1] Collected 3
```

* 注意，当收集发生在主线程中，`flow { ... }` 是如何在后台线程中工作的：

* 这里要观察的另一件事是 [flowOn](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flow-on.html) 操作符已改变流的默认顺序性。 现在收集发生在一个协程中（“coroutine#1”）而发射发生在运行于另一个线程中与收集协程并发运行的另一个协程（“coroutine#2”）中。当上游流必须改变其上下文中的 [CoroutineDispatcher](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-dispatcher/index.html) 的时候，[flowOn](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flow-on.html) 操作符创建了另一个协程。

### 九、缓冲

* 从收集流所花费的时间来看，将流的不同部分运行在不同的协程中将会很有帮助，特别是当涉及到长时间运行的异步操作时。例如，考虑一种情况， 一个 `simple` 流的发射很慢，它每花费 100 毫秒才产生一个元素；而收集器也非常慢， 需要花费 300 毫秒来处理元素。让我们看看从该流收集三个数字要花费多长时间：

```kotlin
fun simple(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100) // 假装我们异步等待了 100 毫秒
        emit(i) // 发射下一个值
    }
}

fun main() = runBlocking<Unit> { 
    val time = measureTimeMillis {
        simple().collect { value -> 
            delay(300) // 假装我们花费 300 毫秒来处理它
            println(value) 
        } 
    }   
    println("Collected in $time ms")
}
```

* 它会产生这样的结果，整个收集过程大约需要 1200 毫秒（3 个数字，每个花费 400 毫秒）：

```
1
2
3
Collected in 1220 ms
```

* 我们可以在流上使用 [buffer](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/buffer.html) 操作符来并发运行这个 `simple` 流中发射元素的代码以及收集的代码， 而不是顺序运行它们：

```kotlin
fun simple(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100) // 假装我们异步等待了 100 毫秒
        emit(i) // 发射下一个值
    }
}

fun main() = runBlocking<Unit> {
    val time = measureTimeMillis {
        simple()
            .buffer() // 缓冲发射项，无需等待
            .collect { value ->
                delay(300) // 假装我们花费 300 毫秒来处理它
                println(value)
            }
    }
    println("Collected in $time ms")
}
```

* 它产生了相同的数字，只是更快了，由于我们高效地创建了处理流水线， 仅仅需要等待第一个数字产生的 100 毫秒以及处理每个数字各需花费的 300 毫秒。这种方式大约花费了 1000 毫秒来运行：

```
1
2
3
Collected in 1071 ms
```

> 注意，当必须更改 [CoroutineDispatcher](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-dispatcher/index.html) 时，[flowOn](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flow-on.html) 操作符使用了相同的缓冲机制， 但是我们在这里显式地请求缓冲而不改变执行上下文。

#### 9.1 合并

* 当流代表部分操作结果或操作状态更新时，可能没有必要处理每个值，而是只处理最新的那个。在本示例中，当收集器处理它们太慢的时候， [conflate](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/conflate.html) 操作符可以用于跳过中间值。构建前面的示例：

```kotlin
fun simple(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100) // 假装我们异步等待了 100 毫秒
        emit(i) // 发射下一个值
    }
}

fun main() = runBlocking<Unit> { 
    val time = measureTimeMillis {
        simple()
            .conflate() // 合并发射项，不对每个值进行处理
            .collect { value -> 
                delay(300) // 假装我们花费 300 毫秒来处理它
                println(value) 
            } 
    }   
    println("Collected in $time ms")
}
```

* 我们看到，虽然第一个数字仍在处理中，但第二个和第三个数字已经产生，因此第二个是 *conflated* ，只有最新的（第三个）被交付给收集器：

```
1
3
Collected in 758 ms
```

#### 9.2 处理最新值

* 当发射器和收集器都很慢的时候，合并是加快处理速度的一种方式。它通过删除发射值来实现。 另一种方式是取消缓慢的收集器，并在每次发射新值的时候重新启动它。有一组与 `xxx` 操作符执行相同基本逻辑的 `xxxLatest` 操作符，但是在新值产生的时候取消执行其块中的代码。让我们在先前的示例中尝试更换 [conflate](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/conflate.html) 为 [collectLatest](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/collect-latest.html)：

```kotlin
fun simple(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100) // 假装我们异步等待了 100 毫秒
        emit(i) // 发射下一个值
    }
}

fun main() = runBlocking<Unit> { 
    val time = measureTimeMillis {
        simple()
            .collectLatest { value -> // 取消并重新发射最后一个值
                println("Collecting $value") 
                delay(300) // 假装我们花费 300 毫秒来处理它
                println("Done $value") 
            } 
    }   
    println("Collected in $time ms")
}
```

* 由于 [collectLatest](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/collect-latest.html) 的函数体需要花费 300 毫秒，但是新值每 100 秒发射一次，我们看到该代码块对每个值运行，但是只收集最后一个值：

```
Collecting 1
Collecting 2
Collecting 3
Done 3
Collected in 741 ms
```

### 十、组合多个流

* 组合多个流有很多种方式。

#### 10.1 Zip

* 就像 Kotlin 标准库中的 [Sequence.zip](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.sequences/zip.html) 扩展函数一样， 流拥有一个 [zip](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/zip.html) 操作符用于组合两个流中的相关值：

```kotlin
fun main() = runBlocking<Unit> { 
    val nums = (1..3).asFlow() // 数字 1..3
    val strs = flowOf("one", "two", "three") // 字符串
    nums.zip(strs) { a, b -> "$a -> $b" } // 组合单个字符串
        .collect { println(it) } // 收集并打印
}
```

* 示例打印如下：

```
1 -> one
2 -> two
3 -> three
```

#### 10.2 Combine

* 当流表示一个变量或操作的最新值时（请参阅相关小节 [conflation](https://www.kotlincn.net/docs/reference/coroutines/flow.html#合并)），可能需要执行计算，这依赖于相应流的最新值，并且每当上游流产生值的时候都需要重新计算。这种相应的操作符家族称为 [combine](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/combine.html)。

* 例如，先前示例中的数字如果每 300 毫秒更新一次，但字符串每 400 毫秒更新一次， 然后使用 [zip](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/zip.html) 操作符合并它们，但仍会产生相同的结果， 尽管每 400 毫秒打印一次结果：

> 我们在本示例中使用 [onEach](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/on-each.html) 过渡操作符来延时每次元素发射并使该流更具说明性以及更简洁。

```kotlin
fun main() = runBlocking<Unit> { 
    val nums = (1..3).asFlow().onEach { delay(300) } // 发射数字 1..3，间隔 300 毫秒
    val strs = flowOf("one", "two", "three").onEach { delay(400) } // 每 400 毫秒发射一次字符串
    val startTime = System.currentTimeMillis() // 记录开始的时间
    nums.zip(strs) { a, b -> "$a -> $b" } // 使用“zip”组合单个字符串
        .collect { value -> // 收集并打印
            println("$value at ${System.currentTimeMillis() - startTime} ms from start") 
        } 
}
```

* 输出结果

```
1 -> one at 467 ms from start
2 -> two at 870 ms from start
3 -> three at 1281 ms from start
```

* 然而，当在这里使用 [combine](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/combine.html) 操作符来替换 [zip](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/zip.html)：

```kotlin
fun main() = runBlocking<Unit> {
    val nums = (1..3).asFlow().onEach { delay(300) } // 发射数字 1..3，间隔 300 毫秒
    val strs = flowOf("one", "two", "three").onEach { delay(400) } // 每 400 毫秒发射一次字符串
    val startTime = System.currentTimeMillis() // 记录开始的时间
    nums.combine(strs) { a, b -> "$a -> $b" } // 使用“combine”组合单个字符串
        .collect { value -> // 收集并打印
            println("$value at ${System.currentTimeMillis() - startTime} ms from start")
        }
}
```

* 我们得到了完全不同的输出，其中，`nums` 或 `strs` 流中的每次发射都会打印一行：

```
1 -> one at 500 ms from start
2 -> one at 703 ms from start
2 -> two at 909 ms from start
3 -> two at 1018 ms from start
3 -> three at 1314 ms from start
```

### 十一、展平流

* 流表示异步接收的值序列，所以很容易遇到这样的情况： 每个值都会触发对另一个值序列的请求。比如说，我们可以拥有下面这样一个返回间隔 500 毫秒的两个字符串流的函数：

```kotlin
fun requestFlow(i: Int): Flow<String> = flow {
    emit("$i: First") 
    delay(500) // 等待 500 毫秒
    emit("$i: Second")    
}
```

* 现在，如果我们有一个包含三个整数的流，并为每个整数调用 `requestFlow`，如下所示：

```
(1..3).asFlow().map { requestFlow(it) }
```

* 然后我们得到了一个包含流的流（`Flow<Flow<String>>`），需要将其进行*展平*为单个流以进行下一步处理。集合与序列都拥有 [flatten](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.sequences/flatten.html) 与 [flatMap](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.sequences/flat-map.html) 操作符来做这件事。然而，由于流具有异步的性质，因此需要不同的展平*模式*， 为此，存在一系列的流展平操作符。

#### 11.1 flatMapConcat

* 连接模式由 [flatMapConcat](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flat-map-concat.html) 与 [flattenConcat](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flatten-concat.html) 操作符实现。它们是相应序列操作符最相近的类似物。它们在等待内部流完成之前开始收集下一个值，如下面的示例所示：

```kotlin
fun requestFlow(i: Int): Flow<String> = flow {
    emit("$i: First") 
    delay(500) // 等待 500 毫秒
    emit("$i: Second")    
}

fun main() = runBlocking<Unit> { 
    val startTime = System.currentTimeMillis() // 记录开始时间
    (1..3).asFlow().onEach { delay(100) } // 每 100 毫秒发射一个数字 
        .flatMapConcat { requestFlow(it) }                                                                           
        .collect { value -> // 收集并打印
            println("$value at ${System.currentTimeMillis() - startTime} ms from start") 
        } 
}
```

* 在输出中可以清楚地看到 [flatMapConcat](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flat-map-concat.html) 的顺序性质：

```
1: First at 121 ms from start
1: Second at 622 ms from start
2: First at 727 ms from start
2: Second at 1227 ms from start
3: First at 1328 ms from start
3: Second at 1829 ms from start
```

#### 11.2 flatMapMerge

* 另一种展平模式是并发收集所有传入的流，并将它们的值合并到一个单独的流，以便尽快的发射值。 它由 [flatMapMerge](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flat-map-merge.html) 与 [flattenMerge](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flatten-merge.html) 操作符实现。他们都接收可选的用于限制并发收集的流的个数的 `concurrency` 参数（默认情况下，它等于 [DEFAULT_CONCURRENCY](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-d-e-f-a-u-l-t_-c-o-n-c-u-r-r-e-n-c-y.html)）。

```kotlin
fun requestFlow(i: Int): Flow<String> = flow {
    emit("$i: First") 
    delay(500) // 等待 500 毫秒
    emit("$i: Second")    
}

fun main() = runBlocking<Unit> { 
    val startTime = System.currentTimeMillis() // 记录开始时间
    (1..3).asFlow().onEach { delay(100) } // 每 100 毫秒发射一个数字 
        .flatMapMerge { requestFlow(it) }                                                                           
        .collect { value -> // 收集并打印
            println("$value at ${System.currentTimeMillis() - startTime} ms from start") 
        } 
}
```

* [flatMapMerge](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flat-map-merge.html) 的并发性质很明显：

```
1: First at 136 ms from start
2: First at 231 ms from start
3: First at 333 ms from start
1: Second at 639 ms from start
2: Second at 732 ms from start
3: Second at 833 ms from start
```

> 注意，[flatMapMerge](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flat-map-merge.html) 会顺序调用代码块（本示例中的 `{ requestFlow(it) }`），但是并发收集结果流，相当于执行顺序是首先执行 `map { requestFlow(it) }` 然后在其返回结果上调用 [flattenMerge](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flatten-merge.html)。

#### 11.3 flatMapLatest

* 与 [collectLatest](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/collect-latest.html) 操作符类似（在["处理最新值"](https://www.kotlincn.net/docs/reference/coroutines/flow.html#处理最新值) 小节中已经讨论过），也有相对应的“最新”展平模式，在发出新流后立即取消先前流的收集。 这由 [flatMapLatest](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flat-map-latest.html) 操作符来实现。

```kotlin
fun requestFlow(i: Int): Flow<String> = flow {
    emit("$i: First") 
    delay(500) // 等待 500 毫秒
    emit("$i: Second")    
}

fun main() = runBlocking<Unit> { 
    val startTime = System.currentTimeMillis() // 记录开始时间
    (1..3).asFlow().onEach { delay(100) } // 每 100 毫秒发射一个数字 
        .flatMapLatest { requestFlow(it) }                                                                           
        .collect { value -> // 收集并打印
            println("$value at ${System.currentTimeMillis() - startTime} ms from start") 
        } 
}
```

* 该示例的输出很好的展示了 [flatMapLatest](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flat-map-latest.html) 的工作方式：

```
1: First at 142 ms from start
2: First at 322 ms from start
3: First at 425 ms from start
3: Second at 931 ms from start
```

> 注意，[flatMapLatest](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flat-map-latest.html) 在一个新值到来时取消了块中的所有代码 (本示例中的 `{ requestFlow(it) }`）。 这在该特定示例中不会有什么区别，由于调用 `requestFlow` 自身的速度是很快的，不会发生挂起， 所以不会被取消。然而，如果我们要在块中调用诸如 `delay` 之类的挂起函数，这将会被表现出来。

### 十二、流异常

* 当运算符中的发射器或代码抛出异常时，流收集可以带有异常的完成。 有几种处理异常的方法。

#### 12.1 收集器 try 与 catch

* 收集者可以使用 Kotlin 的 [`try/catch`](https://kotlinlang.org/docs/reference/exceptions.html) 块来处理异常：

```kotlin
fun simple(): Flow<Int> = flow {
    for (i in 1..3) {
        println("Emitting $i")
        emit(i) // 发射下一个值
    }
}

fun main() = runBlocking<Unit> {
    try {
        simple().collect { value ->         
            println(value)
            check(value <= 1) { "Collected $value" }
        }
    } catch (e: Throwable) {
        println("Caught $e")
    } 
}            
```

* 这段代码成功的在末端操作符 [collect](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/collect.html) 中捕获了异常，并且， 如我们所见，在这之后不再发出任何值：

```
Emitting 1
1
Emitting 2
2
Caught java.lang.IllegalStateException: Collected 2
```

#### 12.2 一切都已捕获

* 前面的示例实际上捕获了在发射器或任何过渡或末端操作符中发生的任何异常。 例如，让我们修改代码以便将发出的值[映射](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/map.html)为字符串， 但是相应的代码会产生一个异常：

```kotlin
fun simple(): Flow<String> = 
    flow {
        for (i in 1..3) {
            println("Emitting $i")
            emit(i) // 发射下一个值
        }
    }
    .map { value ->
        check(value <= 1) { "Crashed on $value" }                 
        "string $value"
    }

fun main() = runBlocking<Unit> {
    try {
        simple().collect { value -> println(value) }
    } catch (e: Throwable) {
        println("Caught $e")
    } 
}            
```

* 仍然会捕获该异常并停止收集：

```
Emitting 1
string 1
Emitting 2
Caught java.lang.IllegalStateException: Crashed on 2
```

### 十三、异常透明性

* 但是，发射器的代码如何封装其异常处理行为？

* 流必须*对异常透明*，即在 `flow { ... }` 构建器内部的 `try/catch` 块中[发射](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-flow-collector/emit.html)值是违反异常透明性的。这样可以保证收集器抛出的一个异常能被像先前示例中那样的 `try/catch` 块捕获。

* 发射器可以使用 [catch](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/catch.html) 操作符来保留此异常的透明性并允许封装它的异常处理。catch 操作符的代码块可以分析异常并根据捕获到的异常以不同的方式对其做出反应：
  * 可以使用 `throw` 重新抛出异常。
  * 可以使用 [catch](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/catch.html) 代码块中的 [emit](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-flow-collector/emit.html) 将异常转换为值发射出去。
  * 可以将异常忽略，或用日志打印，或使用一些其他代码处理它。

* 例如，让我们在捕获异常的时候发射文本：

```kotlin
fun simple(): Flow<String> = 
    flow {
        for (i in 1..3) {
            println("Emitting $i")
            emit(i) // 发射下一个值
        }
    }
    .map { value ->
        check(value <= 1) { "Crashed on $value" }
        "string $value"
    }

fun main() = runBlocking<Unit> {
    simple()
        .catch { e -> emit("Caught $e") } // 发射一个异常
        .collect { value -> println(value) }
}
```

* 即使我们不再在代码的外层使用 `try/catch`，示例的输出也是相同的。

#### 13.1 透明捕获

* [catch](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/catch.html) 过渡操作符遵循异常透明性，仅捕获上游异常（`catch` 操作符上游的异常，但是它下面的不是）。 如果 `collect { ... }` 块（位于 `catch` 之下）抛出一个异常，那么异常会逃逸：

```
fun simple(): Flow<Int> = flow {
    for (i in 1..3) {
        println("Emitting $i")
        emit(i)
    }
}

fun main() = runBlocking<Unit> {
    simple()
        .catch { e -> println("Caught $e") } // 不会捕获下游异常
        .collect { value ->
            check(value <= 1) { "Collected $value" }                 
            println(value) 
        }
}            
```

* 尽管有 `catch` 操作符，但不会打印“Caught …”消息：

```
Emitting 1
1
Emitting 2
Exception in thread "main" java.lang.IllegalStateException: Collected 2
    at ...
```

#### 13.2 声明式捕获

* 我们可以将 [catch](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/catch.html) 操作符的声明性与处理所有异常的期望相结合，将 [collect](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/collect.html) 操作符的代码块移动到 [onEach](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/on-each.html) 中，并将其放到 `catch` 操作符之前。收集该流必须由调用无参的 `collect()` 来触发：

```kotlin
fun simple(): Flow<Int> = flow {
    for (i in 1..3) {
        println("Emitting $i")
        emit(i)
    }
}

fun main() = runBlocking<Unit> {
    simple()
        .onEach { value ->
            check(value <= 1) { "Collected $value" }                 
            println(value) 
        }
        .catch { e -> println("Caught $e") }
        .collect()
}
```

* 现在我们可以看到已经打印了“Caught …”消息，并且我们可以在没有显式使用 `try/catch` 块的情况下捕获所有异常：

```
Emitting 1
1
Emitting 2
Caught java.lang.IllegalStateException: Collected 2
```

### 十四、流完成

* 当流收集完成时（普通情况或异常情况），它可能需要执行一个动作。 你可能已经注意到，它可以通过两种方式完成：命令式或声明式。

#### 14.1 命令式 finally 块

* 除了 `try`/`catch` 之外，收集器还能使用 `finally` 块在 `collect` 完成时执行一个动作。

```kotlin
fun simple(): Flow<Int> = (1..3).asFlow()

fun main() = runBlocking<Unit> {
    try {
        simple().collect { value -> println(value) }
    } finally {
        println("Done")
    }
}            
```

* 这段代码打印出 `simple` 流产生的三个数字，后面跟一个“Done”字符串：

```
1
2
3
Done
```

#### 14.2 声明式处理

* 对于声明式，流拥有 [onCompletion](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/on-completion.html) 过渡操作符，它在流完全收集时调用。

* 可以使用 [onCompletion](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/on-completion.html) 操作符重写前面的示例，并产生相同的输出：

```kotlin
fun simple(): Flow<Int> = (1..3).asFlow()

fun main() = runBlocking<Unit> {
    simple()
        .onCompletion { println("Done") }
        .collect { value -> println(value) }
}

1
2
3
Done
```

* [onCompletion](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/on-completion.html) 的主要优点是其 lambda 表达式的可空参数 `Throwable` 可以用于确定流收集是正常完成还是有异常发生。在下面的示例中 `simple` 流在发射数字 1 之后抛出了一个异常：

```kotlin
fun simple(): Flow<Int> = flow {
    emit(1)
    throw RuntimeException()
}

fun main() = runBlocking<Unit> {
    simple()
        .onCompletion { cause -> if (cause != null) println("Flow completed exceptionally") }
        .catch { cause -> println("Caught exception") }
        .collect { value -> println(value) }
}            
```

* 如你所期望的，它打印了：

```
1
Flow completed exceptionally
Caught exception
```

* [onCompletion](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/on-completion.html) 操作符与 [catch](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/catch.html) 不同，它不处理异常。我们可以看到前面的示例代码，异常仍然流向下游。它将被提供给后面的 `onCompletion` 操作符，并可以由 `catch` 操作符处理。

#### 14.3 成功完成

* 与 [catch](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/catch.html) 操作符的另一个不同点是 [onCompletion](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/on-completion.html) 能观察到所有异常并且仅在上游流成功完成（没有取消或失败）的情况下接收一个 `null` 异常。

```kotlin
fun simple(): Flow<Int> = (1..3).asFlow()

fun main() = runBlocking<Unit> {
    simple()
        .onCompletion { cause -> println("Flow completed with $cause") }
        .collect { value ->
            check(value <= 1) { "Collected $value" }                 
            println(value) 
        }
}
```

* 我们可以看到完成时 cause 不为空，因为流由于下游异常而中止：

```
1
Flow completed with java.lang.IllegalStateException: Collected 2
Exception in thread "main" java.lang.IllegalStateException: Collected 2
```

### 十五、命令式还是声明式

* 现在我们知道如何收集流，并以命令式与声明式的方式处理其完成及异常情况。 这里有一个很自然的问题是，哪种方式应该是首选的？为什么？ 作为一个库，我们不主张采用任何特定的方式，并且相信这两种选择都是有效的， 应该根据自己的喜好与代码风格进行选择。

### 十六、启动流

* 使用流表示来自一些源的异步事件是很简单的。 在这个案例中，我们需要一个类似 `addEventListener` 的函数，该函数注册一段响应的代码处理即将到来的事件，并继续进行进一步的处理。[onEach](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/on-each.html) 操作符可以担任该角色。 然而，`onEach` 是一个过渡操作符。我们也需要一个末端操作符来收集流。 否则仅调用 `onEach` 是无效的。

* 如果我们在 `onEach` 之后使用 [collect](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/collect.html) 末端操作符，那么后面的代码会一直等待直至流被收集：

```kotlin
// 模仿事件流
fun events(): Flow<Int> = (1..3).asFlow().onEach { delay(100) }

fun main() = runBlocking<Unit> {
    events()
        .onEach { event -> println("Event: $event") }
        .collect() // <--- 等待流收集
    println("Done")
}            
```

* 你可以看到它的输出：

```
Event: 1
Event: 2
Event: 3
Done
```

* [launchIn](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/launch-in.html) 末端操作符可以在这里派上用场。使用 `launchIn` 替换 `collect` 我们可以在单独的协程中启动流的收集，这样就可以立即继续进一步执行代码：

```kotlin
// 模仿事件流
fun events(): Flow<Int> = (1..3).asFlow().onEach { delay(100) }

fun main() = runBlocking<Unit> {
    events()
        .onEach { event -> println("Event: $event") }
        .launchIn(this) // <--- 在单独的协程中执行流
    println("Done")
}  
```

* 它打印了：

```
Done
Event: 1
Event: 2
Event: 3
```

* `launchIn` 必要的参数 [CoroutineScope](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/index.html) 指定了用哪一个协程来启动流的收集。在先前的示例中这个作用域来自 [runBlocking](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/run-blocking.html) 协程构建器，在这个流运行的时候，[runBlocking](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/run-blocking.html) 作用域等待它的子协程执行完毕并防止 main 函数返回并终止此示例。

* 在实际的应用中，作用域来自于一个寿命有限的实体。在该实体的寿命终止后，相应的作用域就会被取消，即取消相应流的收集。这种成对的 `onEach { ... }.launchIn(scope)` 工作方式就像 `addEventListener` 一样。而且，这不需要相应的 `removeEventListener` 函数， 因为取消与结构化并发可以达成这个目的。

* 注意，[launchIn](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/launch-in.html) 也会返回一个 [Job](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/index.html)，可以在不取消整个作用域的情况下仅[取消](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/cancel.html)相应的流收集或对其进行 [join](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/join.html)。

### 十七、流取消检测

* 为方便起见，[流](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flow.html)构建器对每个发射值执行附加的 [ensureActive](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/ensure-active.html) 检测以进行取消。 这意味着从 `flow { ... }` 发出的繁忙循环是可以取消的：

```kotlin
fun foo(): Flow<Int> = flow { 
    for (i in 1..5) {
        println("Emitting $i") 
        emit(i) 
    }
}

fun main() = runBlocking<Unit> {
    foo().collect { value -> 
        if (value == 3) cancel()  
        println(value)
    } 
}
```

* 仅得到不超过 3 的数字，在尝试发出 4 之后抛出 [CancellationException](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-cancellation-exception/index.html)；

```
Emitting 1
1
Emitting 2
2
Emitting 3
3
Emitting 4
Exception in thread "main" kotlinx.coroutines.JobCancellationException: BlockingCoroutine was cancelled; job="coroutine#1":BlockingCoroutine{Cancelled}@6d7b4f4c
```

* 但是，出于性能原因，大多数其他流操作不会自行执行其他取消检测。 例如，如果使用 [IntRange.asFlow](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/kotlin.ranges.-int-range/as-flow.html) 扩展来编写相同的繁忙循环， 并且没有在任何地方暂停，那么就没有取消的检测；

```kotlin
fun main() = runBlocking<Unit> {
    (1..5).asFlow().collect { value -> 
        if (value == 3) cancel()  
        println(value)
    } 
}
```

* 收集从 1 到 5 的所有数字，并且仅在从 `runBlocking` 返回之前检测到取消：

```
1
2
3
4
5
Exception in thread "main" kotlinx.coroutines.JobCancellationException: BlockingCoroutine was cancelled; job="coroutine#1":BlockingCoroutine{Cancelled}@3327bd23
```

#### 17.1 让繁忙的流可取消

* 在协程处于繁忙循环的情况下，必须明确检测是否取消。 可以添加 `.onEach { currentCoroutineContext().ensureActive() }`， 但是这里提供了一个现成的 [cancellable](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/cancellable.html) 操作符来执行此操作：

```kotlin
fun main() = runBlocking<Unit> {
    (1..5).asFlow().cancellable().collect { value -> 
        if (value == 3) cancel()  
        println(value)
    } 
}
```

* 使用 `cancellable` 操作符，仅收集从 1 到 3 的数字：

```
1
2
3
Exception in thread "main" kotlinx.coroutines.JobCancellationException: BlockingCoroutine was cancelled; job="coroutine#1":BlockingCoroutine{Cancelled}@5ec0a365
```

### 十八、流（Flow）与响应式流（Reactive Streams）

* 对于熟悉响应式流（[Reactive Streams](https://www.reactive-streams.org/)）或诸如 RxJava 与 Project Reactor 这样的响应式框架的人来说， Flow 的设计也许看起来会非常熟悉。

* 确实，其设计灵感来源于响应式流以及其各种实现。但是 Flow 的主要目标是拥有尽可能简单的设计， 对 Kotlin 以及挂起友好且遵从结构化并发。没有响应式的先驱及他们大量的工作，就不可能实现这一目标。你可以阅读 [Reactive Streams and Kotlin Flows](https://medium.com/@elizarov/reactive-streams-and-kotlin-flows-bfd12772cda4) 这篇文章来了解完成 Flow 的故事。

* 虽然有所不同，但从概念上讲，Flow *依然是*响应式流，并且可以将它转换为响应式（规范及符合 TCK）的发布者（Publisher），反之亦然。 这些开箱即用的转换器可以在 `kotlinx.coroutines` 提供的相关响应式模块（`kotlinx-coroutines-reactive` 用于 Reactive Streams，`kotlinx-coroutines-reactor` 用于 Project Reactor，以及 `kotlinx-coroutines-rx2`/`kotlinx-coroutines-rx3` 用于 RxJava2/RxJava3）中找到。 集成模块包含 `Flow` 与其他实现之间的转换，与 Reactor 的 `Context` 集成以及与一系列响应式实体配合使用的挂起友好的使用方式。