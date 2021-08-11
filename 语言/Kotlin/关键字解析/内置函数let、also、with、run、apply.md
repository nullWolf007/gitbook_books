[TOC]

## 内置函数let、also、with、run、apply

#### 转载

* [巧用Kotlin：内置函数let、also、with、run、apply大大提高你的开发效率！](https://blog.csdn.net/carson_ho/article/details/104471757)
* [【Kotlin篇】差异化分析，let，run，with，apply及also](https://juejin.cn/post/6975384870675546126)

### 一 前言

* 在Kotlin中，有一些用于扩展 & 方便开发者编码的内置函数，能大大提高开发者的开发效率。今天，我将主要讲解的是：
  * let函数
  * also函数
  * with函数
  * run函数
  * apply函数

### 二 基础知识：接口回调中Lambda使用

* 在Kotlin中可使用Lambda函数简化一些不必要的嵌套接口回调方法

* 注：仅支持单个抽象方法回调，多个回调方法不支持。

```java
// Java接口回调
mVar.setEventListener(new ExamEventListener(){
 
    public void onSuccess(Data data){
      // ...
    }
 
});
```
```kotlin
// 同等效果的Kotlin接口回调（无使用lambda表达式）
mVar.setEventListener(object: ExamEventListener{
     
    public void onSuccess(Data data){
      // ...
    } 
});

// Kotlin接口回调（使用lambda表达式，仅留下参数）
mVar.setEventListener({
   data: Data ->
   // ... 
})

// 继续简化
// 简化1：借助kotlin的智能类型推导，忽略数据类型
mVar.setEventListener({
   data ->
   // ... 
})

// 简化2：若参数无使用，可忽略
mVar.setEventListener({
   // ... 
})

// 简化3：若setEventListener函数最后一个参数是一个函数，可把括号的实现提到圆括号外
mVar.setEventListener(){
   // ... 
}

// 简化3：若setEventListener函数只有一个参数 & 无使用到，可省略圆括号
mVar.setEventListener{
   // ... 
}
```

* 下面，我将讲解Kotlin里提供用于扩展 & 方便开发者编码的几个有用内置函数：let函数、also函数、with函数、 run函数、apply函数。

### 三 let函数

#### 1.let块中的最后一条语句如果是非赋值语句，则默认情况下它是返回语句，反之，则返回的是一个 Unit类型

```kotlin
public inline fun <T, R> T.let(block: (T) -> R): R 
```

* let函数是参数化类型 T 的扩展函数。在let块内可以通过 it 指代该对象。返回值为let块的最后一行或指定return表达式。

* 我们以一个Book对象为例，类中包含Book的name和price，如下：

```kotlin
class Book() {
    var name = "《数据结构》"
    var price = 60
    fun displayInfo() = print("Book name : $name and price : $price")
}

fun main(args: Array<String>) {
    val book = Book().let {
        it.name = "《计算机网络》"
        "This book is ${it.name}"
    }
    print(book)
}

控制台输出：
This book is 《计算机网络》
```

* 在上面案例中，我们对Book对象使用let作用域函数，在函数块的最后一句添加了一行字符串代码，并且对Book对象进行打印，我们可以看到最后控制台输出的结果为字符串“This book is 《计算机网络》”。

* 按照我们的编程思想，打印一个对象，输出必定是对象，但是使用let函数后，输出为最后一句字符串。这是由于let函数的特性导致。因为在Kotlin中，如果**let块中的最后一条语句是非赋值语句，则默认情况下它是返回语句。**

* 那如果我们将let块中最后一条语句修改为赋值语句，会发生什么变化？

```kotlin
fun main(args: Array<String>) {
    val book = Book().let {
        it.name = "《计算机网络》"
    }
    print(book)
}

控制台输出：
kotlin.Unit
```

* 可以看到我们将Book对象的name值进行了赋值操作，同样对Book对象进行打印，但是最后控制台的输出结果为“kotlin.Unit”，这是因为在let函数块的最后一句是赋值语句，print则将其当做是一个函数来看待。

* 这是**let**角色设定的**第一点**：
  * **let块中的最后一条语句如果是非赋值语句，则默认情况下它是返回语句，反之，则返回的是一个 Unit类型**

#### 2.let可用于空安全检查。

* 如需对非空对象执行操作，可对其使用安全调用操作符 ?. 并调用 let 在 lambda 表达式中执行操作。如下案例：


```java
// 使用Java
if( mVar != null ){
    mVar.function1();
    mVar.function2();
    mVar.function3();
}
```

```kotlin
// 使用kotlin（无使用let函数）
mVar?.function1()
mVar?.function2()
mVar?.function3()

// 使用kotlin（使用let函数）
// 方便了统一判空的处理 & 确定了mVar变量的作用域
mVar?.let {
	it.function1()
    it.function2()
    it.function3()
}
```

#### 3.let可对调用链的结果进行操作。

* 关于这一点，官方教程给出了一个案例，在这里就直接使用：

```kotlin
fun main(args: Array<String>) { 
    val numbers = mutableListOf("One","Two","Three","Four","Five")
    val resultsList = numbers.map { it.length }.filter { it > 3 }
    print(resultsList)
}

[5, 4, 4]
```

* 我们的目的是获取数组列表中长度大于3的值。因为我们必须打印结果，所以我们将结果存储在一个单独的变量中，然后打印它。但是使用“let”操作符，我们可以将代码修改为:

```kotlin
fun main(args: Array<String>) {
    val numbers = mutableListOf("One","Two","Three","Four","Five")
    numbers.map { it.length }.filter { it > 3 }.let {
        print(it)
    }
}

[5, 4, 4]
```

* 使用let后可以直接对数组列表中长度大于3的值进行打印，去掉了变量赋值这一步。

#### 4.let可以将“It”替代object对象去访问其公有的属性 & 方法

* let是通过使用“It”关键字来引用对象的上下文，因此，这个“It”可以被重命名为一个可读的lambda参数，如下将**it**重命名为**book**：

```kotlin
fun main(args: Array<String>) {
    val book = Book().let {book ->
        book.name = "《计算机网络》"
    }
    print(book)
}
```

### 四 also函数

#### 1 作用 & 应用场景

* 类似let函数，但区别在于返回值：
  * let函数：返回值 = 最后一行 / return的表达式
  * also函数：返回值 = 传入的对象的本身

#### 2 使用示例

```kotlin
// let函数
var result = mVar.let {
	it.function1()
    it.function2()
    it.function3()
    999
}
// 最终结果 = 返回999给变量result

// also函数
var result = mVar.also {
	it.function1()
    it.function2()
    it.function3()
    999
}
// 最终结果 = 返回一个mVar对象给变量result
```

### 五 with函数

#### 1 作用

* 调用同一个对象的多个方法 / 属性时，可以省去对象名重复，直接调用方法名 / 属性即可

#### 2 应用场景

* 需要调用同一个对象的多个方法 / 属性

#### 3 使用方法

```kotlin
with(object){
	// ... 
}
// 返回值 = 函数块的最后一行 / return表达式
```

#### 4 使用示例

```kotlin
// 此处要调用people的name 和 age属性
// kotlin
val people = People("carson", 25)
with(people) {
	println("my name is $name, I am $age years old")
}
```

```java
// Java
User peole = new People("carson", 25);
String var1 = "my name is " + peole.name + ", I am " + peole.age + " years old";
System.out.println(var1);
```

### 六 run函数

#### 1 作用 & 应用场景

* 结合了let、with两个函数的作用，即：
  * 调用同一个对象的多个方法 / 属性时，可以省去对象名重复，直接调用方法名 / 属性即可
  * 定义一个变量在特定作用域内
  * 统一做判空处理

#### 2 使用方法

```kotlin
object.run{
	// ... 
}
// 返回值 = 函数块的最后一行 / return表达式
```

#### 3 使用示例

```kotlin
// 此处要调用people的name 和 age属性，且要判空
// kotlin
val people = People("carson", 25)
people?.run{
    println("my name is $name, I am $age years old")
}
```

```java
// Java
User peole = new People("carson", 25);
String var1 = "my name is " + peole.name + ", I am " + peole.age + " years old";
System.out.println(var1);
```

### 七 apply函数

#### 1 作用 & 应用场景

* 与run函数类似，但区别在于返回值：
  * run函数返回最后一行的值 / 表达式
  * apply函数返回传入的对象的本身

#### 2 应用场景

* 对象实例初始化时需要对对象中的属性进行赋值 & 返回该对象

#### 3 使用示例
```kotlin
// run函数
val people = People("carson", 25)
val result = people?.run{
    println("my name is $name, I am $age years old")
    999
}
// 最终结果 = 返回999给变量result
```

```kotlin
val people = People("carson", 25)
val result = people?.run{
    println("my name is $name, I am $age years old")
    999
}
// 最终结果 = 返回一个people对象给变量result
```

* 至此，关于Kotlin里提供用于扩展 & 方便开发者编码的几个有用内置函数讲解完毕。

### 八 小结

- **用于初始化对象或更改对象属性，可使用`apply`**
- **如果将数据指派给接收对象的属性之前验证对象，可使用`also`**
- **如果将对象进行空检查并访问或修改其属性，可使用`let`**
- **如果是非null的对象并且当函数块中不需要返回值时，可使用`with`**
- **如果想要计算某个值，或者限制多个本地变量的范围，则使用`run`**
