[TOC]

## final

#### 参考

* [程序员你真的理解final关键字吗？](https://blog.csdn.net/qq_44543508/article/details/102720206)

* [浅析Java中的final关键字](https://www.cnblogs.com/dolphin0520/p/3736238.html)

* [Java并发（十九）：final实现原理](https://www.cnblogs.com/hexinwei1/p/10025840.html)

#### 说明

* 基于JDK1.8

###  一、final的基本用法

#### 1.1 修饰类

##### 1.1.1 作用

* 当用final修饰一个类时，表明这个类不能被继承。
* final类中的成员变量可以根据需要设为final，但是要注意final类中的所有成员方法(类中声明的方法)都会被隐式地指定为final方法。
* 在使用final修饰类的时候，要注意谨慎选择，除非这个类真的在以后不会用来继承或者出于安全的考虑，尽量不要将类设计为final类。

##### 1.1.2 实例

```java
final class Person {
}

class Student extends Person{//这里编译报错
}
```

* 报错信息：Cannot inherit from final 'Person'，不能继承final的类。

#### 1.2 修饰方法

##### 1.2.1 作用

* 第一个原因是把方法锁定，以防任何继承类修改它的含义
* 第二个原因是效率。在早期的Java实现版本中，会将final方法转为内嵌调用。但是如果方法过于庞大，可能看不到内嵌调用带来的任何性能提升。在最近的Java版本中，不需要使用final方法进行这些优化了。
* 类的private方法会隐式地被指定为final方法。

##### 1.2.2 实例

```java
class Person {
    public final String getName() {
        return "person name";
    }
}

class Student extends Person {
    @Override
    public final String getName() {
        return "student name";//编译错误，无法被重写
    }

    public String test() {
        return getName(); //可以调用，因为是public方法
    }
}
```

* 报错信息：'getName()' cannot override 'getName()' in 'Person'; overridden method is final

#### 1.3 修饰变量

##### 1.3.1 作用

* 当final修饰一个基本数据类型时，表示该基本数据类型的值一旦在初始化后便不能发生变化；如果final修饰一个引用类型时，则在对其初始化之后便不能再让其指向其他对象了，但该引用所指向的对象的内容是可以发生变化的。本质上是一回事，因为引用的值是一个地址，final要求值，即地址的值不发生变化。

##### 1.3.2 实例

```java
public class Test {
    public final int res = 0;
    public final String str = new String();
    public final StringBuilder stringBuilder = new StringBuilder();

    public void test() {
        res = 1;//编译错误 Cannot assign a value to final variable 'res
        str = "11";//编译错误 Cannot assign a value to final variable 'str'
        stringBuilder.append("1111");
    }
}
```

* res是基本数据类型，不能更改值，所以编译错误
* str是引用类型，由于String本质是char数组，所以赋值地址发生了变化，所以编译失败
* stringBuilder是引用类型，但是由于其地址的值没有改变，所以是编译成功的

#### 1.4 修饰成员变量

##### 1.4.1 修饰变量基础上增加的效果

* 成员变量必须在定义时或者构造器中进行初始化赋值,否则编译不通过

##### 1.4.2 实例

```java
public class Test {
    public final int res = 1;//编译正确
    public final String str;//编译正确
    //编译错误 Variable 'integer' might not have been initialized
    public final Integer integer;
    
    public Test() {
        str = "abc";
    }
}
```

* res在定义时候初始化赋值的,编译通过
* str在构造器中初始化赋值的,编译通过
* integer没有进行赋值,编译错误

#### 1.5 修饰局部变量

* 修饰局部变量只有修饰变量的效果，没有新增的效果。在使用该final变量之前赋值了就行。赋值过之后不能修改(值/地址)

### 二、深入理解

#### 2.1 final变量和普通变量有什么区别？

* 通过实例取了解它们的区别

```java
public class Test {
    public static void main(String[] args) {
        String a = "HelloWorld";
        final String b = "Hello";
        final String c = "World";
        String d = "Hello";
        String e = "World";
        String f = b + c;
        String g = d + e;
        String h = b + e;
        System.out.println((a == f));
        System.out.println((a == g));
        System.out.println((a == h));
        System.out.println((a == i));
    }
}
```

* 输出结果

```text
true
false
false
true
```

* 这里面就是final变量和普通变量的区别了，当final变量是基本数据类型以及String类型时，如果在编译期间能知道它的确切值，则编译器会把它当做编译期常量使用。也就是说在用到该final变量的地方，相当于直接访问的这个常量，不需要在运行时确定。这种和C语言中的宏替换有点像。因此在上面的一段代码中，由于变量b和c被final修饰，因此会被当做编译器常量，所以在使用到b和c的地方会直接将变量b和c替换为它的 值。则a和g和i都指向的是常量池中相同的字符串(常量池中相同的字符串只有一份)。而g是字符串连接，内部实际是生成的StringBuilder对象去操作的，所以地址肯定不一样。详解可看[String](String.md)

* **注意**：只有在编译期间能确切知道final变量值的情况下，编译器才会进行这样的优化

```java
public class Test {
    public static void main(String[] args) {
        String a = "HelloWorld";
        final String b = getHello();
        final String c = "World";
        String d = b + c;
        System.out.println((a == d));
    }

    public static String getHello() {
        return "Hello";
    }
}
```

* 输出结果：false

#### 2.2 被final修饰的引用变量指向的对象内容可变吗？

* 被final修饰的引用变量一旦初始化赋值完成之后就不能指向其他对象了，但是对象的内容是可以变化的

```java
public class Test {
    public static void main(String[] args) {
        final StringBuilder stringBuilder = new StringBuilder("111");
        System.out.println(stringBuilder.toString());
        stringBuilder.append("222");
        System.out.println(stringBuilder.toString());
    }
}
```

* 编译成功，输出内容

```text
111
111222
```

#### 2.3 final和static

* static作用于成员变量用来表示只保存一份副本，而final的作用是用来保证变量初始化之后不可变。

```java
public class Test {
    public static void main(String[] args) {
        MyClass class1 = new MyClass();
        MyClass class2 = new MyClass();
        System.out.println(class1.i);
        System.out.println(class1.j);
        System.out.println(class2.i);
        System.out.println(class2.j);
    }
}

class MyClass {
    public final double i = Math.random();
    public static double j = Math.random();
}
```

* 输出结果：class1和class2的j的值相同，i的值不同。

#### 2.4 java为什么匿名内部类的参数引用必须final?

* [java为什么匿名内部类的参数引用时final](https://www.zhihu.com/question/21395848)

### 三、实现原理

* 



