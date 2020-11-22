[TOC]

## static

#### 参考

* [深入理解static关键字](https://blog.csdn.net/qq_44543508/article/details/102736466)
* [一个 static 还能难得住我？](https://juejin.im/post/6844904175545876488)

### 一、简介

#### 1.1 作用

* 创建独立于具体对象的域变量或者方法。**以致于即使没有创建对象，也能使用属性和调用方法**！
*  **用来形成静态代码块以优化程序性能**。static块可以置于类中的任何地方，类中可以有多个static块。在类初次被加载的时候，会按照static块的顺序来执行每个static块，并且只会执行一次。为什么说static块可以用来优化程序性能，是因为它的特性:只会在类加载的时候执行一次。因此，很多时候会将一些只需要进行一次的初始化操作都放在static代码块中进行。

#### 1.2 限制

* static 关键字只能定义在类的 `{}` 中，而不能定义在任何方法中。
* **static是不允许用来修饰局部变量**

#### 1.3 特点

* 被static修饰的变量或者方法是独立于该类的任何对象，也就是说，这些变量和方法**不属于任何一个实例对象，而是被类的实例对象所共享**。
* 在该类被第一次加载的时候，就会去加载被static修饰的部分，而且只在类第一次使用时加载并进行初始化，注意这是第一次用就要初始化，后面根据需要是可以再次赋值的。
* static变量值在类加载的时候分配空间，以后创建类对象的时候不会重新分配。赋值的话，是可以任意赋值的！
* 被static修饰的变量或者方法是优先于对象存在的，也就是说当一个类加载完毕之后，即便没有创建对象，也可以去访问。

#### 1.4 应用场景

* 修饰成员变量
* 修饰成员方法
* 静态代码块
* 修饰类【只能修饰内部类也就是静态内部类】
* 静态导包

### 二、基本使用

#### 2.1 修饰成员变量

##### 2.1.1 概述

* 静态变量可以通过`类.静态变量`的方式去调用，也可以通过`对象.静态变量`的方式去调用
* 静态变量被所有的对象所共享，在内存中只有一个副本，它当且仅当在类初次加载时会被初始化。而非静态变量是对象所拥有的，在创建对象的时候被初始化，存在多个副本，各个对象拥有的副本互不影响。
* static成员变量的初始化顺序按照定义的顺序进行初始化
* 在方法中可以通过this关键字去指向静态变量。

##### 2.1.2 实例

```java
public class Test {
    public static void main(String[] args) {
        System.out.println(MyClass.num);
        MyClass myClass = new MyClass();
        System.out.println(myClass.num);
        myClass.getNum();
    }
}

class MyClass {
    public static int num = 233;

    public void getNum() {
        int num = 666;
        System.out.println(num);
        System.out.println(this.num);
    }
}
```

* 输出结果

```text
233
233
666
233
```

* 第一行和第二行分别是上述的两种方式去获取静态变量
* 第三行获取的是成员变量，就近原则
* 第四行都过this关键字指向了同名的成员静态变量

#### 2.2 修饰方法

##### 2.2.1 概述

* static可以修饰方法，被static修饰的方法被称为静态方法

* **static方法就是没有this的方法，在static内部不能调用非静态方法，反过来是可以的。**this指的是当前对象，因为static静态方法不属于任何对象，所以就谈不上this了。

#### 2.3 修饰代码块

##### 2.3.1 概述

* static 修饰的代码块被称为静态代码块。静态代码块可以置于类中的任何地方，类中可以有多个 static 块，在类初次被加载的时候，会按照 static 代码块的顺序来执行，每个 static 修饰的代码块只能执行一次。

* 多个类的继承中初始化块、静态初始化块、构造器的执行顺序为：**父类静态块——>子类静态块——>父类代码块——>父类构造器——>子类代码块——>子类构造器**

##### 2.3.2 实例

```java
public class Test {
    public static void main(String[] args) {
        Son son1 = new Son();
        System.out.println("-------------");
        Son son2 = new Son();
    }
}

class Father {
    static {
        System.out.println("Static Father");
    }

    {
        System.out.println("Normal Father");
    }

    public Father() {
        System.out.println("Constructor Father");
    }
}

class Son extends Father {
    static {
        System.out.println("Static Son");
    }

    {
        System.out.println("Normal Son");
    }

    public Son() {
        System.out.println("Constructor Son");
    }
}
```

* 输出结果

```text
Static Father
Static Son
Normal Father
Constructor Father
Normal Son
Constructor Son
-------------
Normal Father
Constructor Father
Normal Son
Constructor Son
```

#### 2.4 静态内部类

##### 2.4.1 概述

* `静态内部类`就是用 static 修饰的内部类，静态内部类可以包含静态成员，也可以包含非静态成员，但是在非静态内部类中不可以声明静态成员。
* 非静态内部类的实例创建需要有外部类对象的引用，所以非静态内部类对象的创建必须依托于外部类的实例；而静态内部类的实例创建只需依托外部类
* 由于非静态内部类对象持有了外部类对象的引用，因此非静态内部类可以访问外部类的静态和非静态成员；而静态内部类只能访问外部类的静态成员；

##### 2.4.2 实例

* 静态内部类实现单例模式

```java
public class Singleton {
    
   // 声明为 private 避免调用默认构造方法创建对象
    private Singleton() {
    }
    
   // 声明为 private 表明静态内部该类只能在该 Singleton 类中被访问
    private static class SingletonHolder {
        private static final Singleton INSTANCE = new Singleton();
    }

    public static Singleton getUniqueInstance() {
        return SingletonHolder.INSTANCE;
    }
}
```

* 当 `Singleton` 类加载时，静态内部类 `SingletonHolder` 没有被加载进内存。只有当调用 `getUniqueInstance()`方法从而触发 `SingletonHolder.INSTANCE` 时 `SingletonHolder` 才会被加载，此时初始化 `INSTANCE` 实例，并且 JVM 能确保 `INSTANCE` 只被实例化一次。
* 这种方式不仅具有延迟初始化的好处，而且由 JVM 提供了对线程安全的支持。

#### 2.5 静态导包

##### 2.5.1 概述

* 静态导入就是使用 `import static` 用来导入某个类或者某个包中的静态方法或者静态变量。并且不需要使用类名调用类中静态成员，可以直接使用类中静态成员变量和成员方法

##### 2.5.2 实例

```java
//  Math. --- 将Math中的所有静态资源导入，这时候可以直接使用里面的静态方法，而不用通过类名进行调用
//  如果只想导入单一某个静态方法，只需要将换成对应的方法名即可
 
import static java.lang.Math.;
//  换成import static java.lang.Math.max;具有一样的效果
 
public class Demo {
	public static void main(String[] args) {
 
		int max = max(1,2);
		System.out.println(max);
	}
}
```

* 静态导包在书写代码的时候确实能省一点代码，可以直接调用里面的静态成员，但是会影响代码可读性，所以开发中一般情况下不建议这么使用。

### 三、深入理解

#### 3.1 final

* 先了解一下[final](final.md)关键字

#### 3.2 JVM内存区域

* 查看[JVM内存区域](必备Java知识/JVM/GC/Java内存区域.md)

* **static 修饰的变量存储在方法区(元空间？)中**

#### 3.3 static变量的生命周期

* static 变量的生命周期与类的生命周期相同，随类的加载而创建，随类的销毁而销毁；普通成员变量和其所属的生命周期相同。

#### 3.4 static序列化

* 可先查看序列化[Serializable和Parcelable比较](Android必备技能/数据传输及序列化/Serializable和Parcelable比较.md)
* 序列化的目的就是为了 **把 Java 对象转换为字节序列**。对象转换为有序字节流，以便其能够在网络上传输或者保存在本地文件中。
* 声明为 static 和 transient 类型的变量不能被序列化，因为 static 修饰的变量保存在方法区中，只有堆内存才会被序列化。而 `transient` 关键字的作用就是防止对象进行序列化操作。

#### 3.5 类加载顺序

* 类的初始化顺序依次是
* 加载父类的静态字段 -> 父类的静态代码块 -> 子类静态字段 -> 子类静态代码块 -> 父类成员变量（非静态字段）-> 父类非静态代码块 -> 父类构造器 -> 子类成员变量 -> 子类非静态代码块 -> 子类构造器

