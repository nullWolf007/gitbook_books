### 函数式(SAM)接口

#### 1.定义

* 定义：只有一个抽象方法的接口称为*函数式接口*或 *SAM（单一抽象方法）*接口。函数式接口可以有多个非抽象成员，但只能有一个抽象成员。
* 解释：只能有一个抽象成员以及一个抽象方法的接口，叫做函数式接口。
* 可以用 `fun` 修饰符在 Kotlin 中声明一个函数式接口。

```kotlin
fun interface KRunnable {
   fun invoke()
}
```

#### 2.SAM转换

* 使用 lambda 表达式可以替代手动创建实现函数式接口的类。 通过 SAM 转换， Kotlin 可以将其签名与接口的单个抽象方法的签名匹配的任何 lambda 表达式转换为实现该接口的类的实例。

* 函数式接口

```kotlin
fun interface IntPredicate {
   fun accept(i: Int): Boolean
}
```

* 不使用SAM转换

```kotlin
// 创建一个类的实例
val isEven = object : IntPredicate {
   override fun accept(i: Int): Boolean {
       return i % 2 == 0
   }
}
```

* 使用SAM转换

```kotlin
// 通过 lambda 表达式创建一个实例
val isEven = IntPredicate { it % 2 == 0 }
```

* 使用

```kotlin
fun interface IntPredicate {
   fun accept(i: Int): Boolean
}

val isEven = IntPredicate { it % 2 == 0 }

fun main() {
   println("Is 7 even? - ${isEven.accept(7)}")
}
```
