[TOC]

## 委托之by

##### 转载

* [一文彻底搞懂Kotlin中的委托](https://juejin.cn/post/6844904038589267982#heading-0)

### 1. 什么是委托？

* 委托，也就是委托模式，它是23种经典设计模式种的一种，又名`代理模式`，在委托模式中，有2个对象参与同一个请求的处理，接受请求的对象将请求委托给另一个对象来处理。委托模式是一项技巧，其他的几种设计模式如：策略模式、状态模式和访问者模式都是委托模式的具体场景应用。

* 委托模式中，有三个角色，约束、委托对象和被委托对象。

<img src="https://user-gold-cdn.xitu.io/2020/1/6/16f786688c5c0310?imageView2/0/w/1280/h/960/format/webp/ignore-error/1" alt="委托模式中的角色.png" style="zoom: 50%;" />

- 约束： 约束是接口或者抽象类，它定义了通用的业务类型，也就是需要被代理的业务
- 被委托对象： 具体的业务逻辑执行者
- 委托对象： 负责对真实角色的应用，将约束类定义的业务委托给具体的委托对象。

### 2. 委托的具体场景

* 上一节讲了委托的定义和它所包含的几个角色，那么具体该怎么运用呢？我们以一个实际的例子来看看。

* 现在很多年轻人都爱完游戏，不管是吃鸡、王者荣耀还是英雄联盟。它们都是有等级之分的：`青铜->白银->黄金->铂金->钻石->宗师->王者`，等级越高，代表你越厉害，就拿英雄联盟来说，我们多数混迹在白银黄金阶段，要上钻石宗师段位非常困难。比如你排位打了很久，就差几场就能上宗师了，老是打不上去，这个时候怎么办呢？好办，现在有很多游戏代练，`委托`游戏代练给你打上去就好了。这其实就是一个委托模式。代码该怎么写呢？一起来看看：

* 首先，我们定义约束类，定义我们需要委托的业务，就拿这个场景来说，我们的业务就是`打排位赛，升级`。因此，定义个约束类(接口)`IGamePlayer`:

```kotlin
// 约束类
interface IGamePlayer {
    // 打排位赛
    fun rank()
    // 升级
    fun upgrade()
}
```

* 约束类中，定义了我们要代理的业务`rank()`，`upgrade()`，然后，我们就定义`被委托对象`，也就是`游戏代练`：

```kotlin
// 被委托对象，本场景中的游戏代练
class RealGamePlayer(private val name: String): IGamePlayer{
    override fun rank() {
        println("$name 开始排位赛")
    }

    override fun upgrade() {
       println("$name 升级了")
    }
}
```

* 如上，我们定义了一个被委托对象`RealGamePlayer`, 它有一个属性`name`,它实现了我们约定的业务（实现了接口方法）。

* 接下来，就是`委托角色`：

```kotlin
// 委托对象
class DelegateGamePlayer(private val player: IGamePlayer): IGamePlayer by player
```

* 我们定义了一个委托类`DelegateGamePlayer`, 现在游戏代练有很多，水平有高有低，如果发现水平不行，我们可以随时换，因此，我们把被委托对象作为委托对象的属性，通过构造方法传进去。

> 注意：在kotlin 中，委托用关键字`by` 修饰，`by`后面就是你委托的对象，可以是一个`表达式`。因此在本例中，通过`by player` 委托给了具体的被委托对象。

* 最后，看一下场景测试类：

```kotlin
// Client 场景测试
fun main() {
    val realGamePlayer = RealGamePlayer("张三")
    val delegateGamePlayer = DelegateGamePlayer(realGamePlayer)
    delegateGamePlayer.rank()
    delegateGamePlayer.upgrade()
}
```

* 我们定义了一个游戏代练，叫张三，将它传递给委托类，然后就可以开始排位和升级的业务了，而最终谁完成了排位赛和升级了，当然是我们的被委托对象，也就是游戏代练--张三。

* 运行，结果如下：

```txt
张三 开始排位赛
张三 升级了
```

* 小结：以上就是委托的应用，再来回顾一下它的定义：2个对象参与处理同一请求，这个`请求`就是我们约束类的逻辑，因此委托类(`DelegateGamePlayer`)和被委托类(`RealGamePlayer`)都需要实现我们的约束接口`IGamePlayer`。

### 3. 属性委托

* 在Kotlin 中，有一些常见的属性类型，虽然我们可以在每次需要的时候手动实现它们，但是很麻烦，各种样板代码存在，我们知道，Kotlin可是宣称要实现零样板代码的。为了解决这些问题呢？Kotlin标准为我们提供了`委托属性`。

```kotlin
class Test {
    // 属性委托
    var prop: String by Delegate()
}
```

* `委托属性`的语法如下：

```kotlin
val/var <属性名>: <类型> by <表达式>
```

* 跟我们前面将的委托类似，只不过前面是`类委托`，这里`属性委托`。

#### 3.1 属性委托的原理

* 前面讲的委托中，我们有个`约束角色`，里面定义了代理的业务逻辑。而委托属性呢？其实就是上面的简化，被代理的逻辑就是这个属性的`get`/`set`方法。`get`/`set`会委托给`被委托对象`的`setValue`/`getValue`方法，因此被委托类需要提供`setValue`/`getValue`这两个方法。如果是`val` 属性，只需提供`getValue`。如果是`var` 属性，则`setValue`/`getValue`都需要提供。

* 比如上面的`Delegate`类：

```kotlin
class Delegate {
    operator fun getValue(thisRef: Any?, property: KProperty<*>): String {
        return "$thisRef, thank you for delegating '${property.name}' to me!"
    }

    operator fun setValue(thisRef: Any?, property: KProperty<*>, value: String) {
        println("$value has been assigned to '${property.name}' in $thisRef.")
    }
}
```

其中的参数解释如下：

- `thisRef` —— 必须与 属性所有者 类型（对于扩展属性——指被扩展的类型）相同或者是它的超类型；
- `property` —— 必须是类型 `KProperty<*>`或其超类型。
- `value` —— 必须与属性同类型或者是它的子类型。

* 测试如下：

```kotlin
fun main() {
    println(Test().prop)
    Test().prop = "Hello, Android技术杂货铺！"
}
```

* 打印结果如下：

```kotlin
kotlinpack.Test@117ae12, thank you for delegating 'prop' to me!
Hello, Android技术杂货铺！ has been assigned to 'prop' in kotlinpack.Test@1405ef7.
```

#### 3.2 另一种实现属性委托的方式

* 上面我们讲了，要实现属性委托，就必须要提供`getValue`/`setValue`方法，对于比较`懒`的同学可能就要说了，这么复杂的参数，还要每次都要手写，真是麻烦，一不小心就写错了。确实是这样，为了解决这个问题， Kotlin 标准库中声明了2个含所需 `operator`方法的 `ReadOnlyProperty / ReadWriteProperty` 接口。

```kotlin
interface ReadOnlyProperty<in R, out T> {
    operator fun getValue(thisRef: R, property: KProperty<*>): T
}

interface ReadWriteProperty<in R, T> {
    operator fun getValue(thisRef: R, property: KProperty<*>): T
    operator fun setValue(thisRef: R, property: KProperty<*>, value: T)
}
```

* `被委托类` 实现这两个接口其中之一就可以了，`val` 属性实现`ReadOnlyProperty`，`var`属性实现`ReadOnlyProperty`。 

```kotlin
// val 属性委托实现
class Delegate1: ReadOnlyProperty<Any,String>{
    override fun getValue(thisRef: Any, property: KProperty<*>): String {
        return "通过实现ReadOnlyProperty实现，name:${property.name}"
    }
}
// var 属性委托实现
class Delegate2: ReadWriteProperty<Any,Int>{
    override fun getValue(thisRef: Any, property: KProperty<*>): Int {
        return  20
    }

    override fun setValue(thisRef: Any, property: KProperty<*>, value: Int) {
       println("委托属性为： ${property.name} 委托值为： $value")
    }

}
// 测试
class Test {
    // 属性委托
    val d1: String by Delegate1()
    var d2: Int by Delegate2()
}
```

* 如上代码所示，定义了2个属性代理，都通过 `ReadOnlyProperty / ReadWriteProperty` 接口实现。

* 测试代码如下：

```kotlin
val test = Test()
println(test.d1)
println(test.d2)
test.d2 = 100
```

* 打印结果：

```
通过实现ReadOnlyProperty实现，name:d1
20
委托属性为： d2 委托值为： 100
```

* 可以看到，与手动实现setValue/getValue效果一样，但是这样写代码就方便了很多了。

### 4. Kotlin 标准库中提供几个委托

* Kotlin 标准库中提供了几种委托，例如：
  * 延迟属性（lazy properties）: 其值只在首次访问时计算；
  * 可观察属性（observable properties）: 监听器会收到有关此属性变更的通知；
  * 把多个属性储存在一个映射（map）中，而不是每个存在单独的字段中。

#### 4.1 延迟属性 lazy

* [`lazy()`](https://link.juejin.cn/?target=https%3A%2F%2Fkotlinlang.org%2Fapi%2Flatest%2Fjvm%2Fstdlib%2Fkotlin%2Flazy.html) 是接受一个 lambda 并返回一个 `Lazy <T>` 实例的函数，返回的实例可以作为实现延迟属性的委托： 第一次调用 `get()` 会执行已传递给 `lazy()` 的 lambda 表达式并记录结果， 后续调用 `get()` 只是返回记录的结果。

```kotlin
val lazyProp: String by lazy {
    println("Hello，第一次调用才会执行我！")
    "西哥！"
}

// 打印lazyProp 3次，查看结果
fun main() {
    println(lazyProp)
    println(lazyProp)
    println(lazyProp)
}
```

* 打印结果如下：

```
Hello，第一次调用才会执行我！
西哥！
西哥！
西哥！
```

* 可以看到，只有第一次调用，才会执行`lambda`表达式中的逻辑，后面调用只会返回`lambda`表达式的最终值。

##### 4.1.1 lazy 也可以接受参数

* `lazy`延迟初始化是可以接受参数的，提供了如下三个参数：

```kotlin
/**
 * Specifies how a [Lazy] instance synchronizes initialization among multiple threads.
 */
public enum class LazyThreadSafetyMode {

    /**
     * Locks are used to ensure that only a single thread can initialize the [Lazy] instance.
     */
    SYNCHRONIZED,

    /**
     * Initializer function can be called several times on concurrent access to uninitialized [Lazy] instance value,
     * but only the first returned value will be used as the value of [Lazy] instance.
     */
    PUBLICATION,

    /**
     * No locks are used to synchronize an access to the [Lazy] instance value; if the instance is accessed from multiple threads, its behavior is undefined.
     *
     * This mode should not be used unless the [Lazy] instance is guaranteed never to be initialized from more than one thread.
     */
    NONE,
}
```

* 三个参数解释如下：
  * `LazyThreadSafetyMode.SYNCHRONIZED`: 添加同步锁，使`lazy`延迟初始化线程安全
  * `LazyThreadSafetyMode. PUBLICATION`： 初始化的`lambda`表达式可以在同一时间被多次调用，但是只有第一个返回的值作为初始化的值。
  * `LazyThreadSafetyMode. NONE`：没有同步锁，多线程访问时候，初始化的值是未知的，非线程安全，一般情况下，不推荐使用这种方式，除非你能保证初始化和属性始终在同一个线程

* 使用如下：

```
val lazyProp: String by lazy(LazyThreadSafetyMode.SYNCHRONIZED) {
    println("Hello，第一次调用才会执行我！")
    "西哥！"
}
```

* 如果你指定的参数为`LazyThreadSafetyMode.SYNCHRONIZED`,则可以省略，因为`lazy`默认就是使用的`LazyThreadSafetyMode.SYNCHRONIZED`。

#### 4.2 可观察属性 Observable

* 如果你要观察一个属性的变化过程，那么可以将属性委托给`Delegates.observable`, `observable`函数原型如下：

```kotlin
public inline fun <T> observable(initialValue: T, crossinline onChange: (property: KProperty<*>, oldValue: T, newValue: T) -> Unit):
            ReadWriteProperty<Any?, T> =
        object : ObservableProperty<T>(initialValue) {
            override fun afterChange(property: KProperty<*>, oldValue: T, newValue: T) = onChange(property, oldValue, newValue)
        }
```

* 接受2个参数：
  * `initialValue`： 初始值
  * `onChange`： 属性值被修改时的回调处理器，回调有三个参数`property`,`oldValue`,`newValue`,分别为： 被赋值的属性、旧值与新值。

* 使用如下：

```kotlin
var observableProp: String by Delegates.observable("默认值：xxx"){
    property, oldValue, newValue ->
    println("property: $property: $oldValue -> $newValue ")
}
// 测试
fun main() {
    observableProp = "第一次修改值"
    observableProp = "第二次修改值"
}
```

* 打印如下：

```
property: var observableProp: kotlin.String: 默认值：xxx -> 第一次修改值 
property: var observableProp: kotlin.String: 第一次修改值 -> 第二次修改值 
```

* 可以看到，每一次赋值，都能观察到值的变化过程。

##### 4.2.1 vetoable 函数

* `vetoable` 与 `observable`一样，可以观察属性值的变化，不同的是，`vetoable`可以通过`处理器函数来决定属性值是否生效`。

* 来看这样一个例子：声明一个Int类型的属性`vetoableProp`，如果新的值比旧值大，则生效，否则不生效。

* 代码如下：

```kotlin
var vetoableProp: Int by Delegates.vetoable(0){
    _, oldValue, newValue ->
    // 如果新的值大于旧值，则生效
    newValue > oldValue
}
```

* 测试代码：

```kotlin
fun main() {
    println("vetoableProp=$vetoableProp")
    vetoableProp = 10
    println("vetoableProp=$vetoableProp")
    vetoableProp = 5
    println("vetoableProp=$vetoableProp")
    vetoableProp = 100
    println("vetoableProp=$vetoableProp")
}
```

* 打印如下：

```kotlin
vetoableProp=0
vetoableProp=10
vetoableProp=10
vetoableProp=100
```

* 可以看到`10 -> 5` 的赋值没有生效。

#### 4.3 属性存储在映射中

* 还有一种情况，在一个映射（map）里存储属性的值，使用映射实例自身作为委托来实现委托属性，如：

```kotlin
class User(val map: Map<String, Any?>) {
    val name: String by map
    val age: Int     by map
}
```

* 测试如下：

```kotlin
fun main() {
    val user = User(mapOf(
        "name" to "西哥",
        "age"  to 25
    ))
   println("name=${user.name} age=${user.age}") 
}
```

* 打印如下：

```
name=西哥 age=25
```

* 使用映射实例自身作为委托来实现委托属性，可以使用在json解析中，因为json本身就可以解析成一个map。**不过，说实话，我暂时还没有发现这种使用场景的好处或者优势，如果有知道的同学，评论区告知，谢谢！**

### 5. 总结

* 委托在kotlin中占有举足轻重的地位，特别是属性委托，lazy延迟初始化使用非常多，还有其他一些场景，比如在我们安卓开发中，使用属性委托来封装`SharePreference`，大大简化了`SharePreference`的存储和访问。在我们软件开发中，始终提倡的是高内聚，低耦合。而委托，就是内聚，可以降低耦合。另一方面，委托的使用，也能减少很多重复的样板代码。

