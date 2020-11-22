[TOC]

## Java注解

#### 参考

* [java注解-最通俗易懂的讲解](https://blog.csdn.net/qq1404510094/article/details/80577555)
* [Java注解-元数据、注解分类、内置注解和自定义注解](https://juejin.cn/post/6844903897908133902#heading-0)

### 一、基础知识

#### 1.1 什么是注解

* 注解是Java 1.5引入的，目前已被广泛应用于各种Java框架，如Hibernate，Jersey，Spring。注解相当于是一种嵌入在程序中的元数据，可以使用注解解析工具或编译器对其进行解析，也可以指定注解在编译期或运行期有效。
* 通俗的来说，是对一个类或者方法或者属性等，贴上来一个标签，我们后续通过反射的技术拿到对应标签的内容，然后对他进行额外的处理

#### 1.2 元数据

##### 1.2.1 概述

* 元数据从metadata一词译来，就是“关于数据的数据”的意思，即描述数据的结构信息。元数据的功能作用有很多，比如：你可能用过Javadoc的注释自动生成文档。这就是元数据功能的一种。总的来说，元数据可以用来创建文档，跟踪代码的依赖性，执行编译时格式检查，代替已有的配置文件。

* 在Java中元数据以标签的形式存在于Java代码中，元数据标签的存在并不影响程序代码的编译和执行，被用来生成其它的文件或只在运行时知道被运行代码的描述信息。

##### 1.2.2 作用

* 生成文档：这是最常见的，也是java 最早提供的注解。常用的有@param @return 等;

* 跟踪代码依赖性，实现替代配置文件功能。常见的是spring 2.5 开始的基于注解配置。作用就是减少配置。现在的框架基本都使用了这种配置来减少配置文件的数量。;

* 在编译时进行格式检查。如@override 放在方法前，如果你这个方法并不是覆盖了超类方法，则编译时就能检查出

#### 1.3 注解的分类

##### 1.3.1 根据注解参数个数

* 标记注解:一个没有成员定义的Annotation类型被称为标记注解。
* 单值注解:只有一个值
* 完整注解:拥有多个值

##### 1.3.2 根据注解使用方法和用途

* JDK内置系统注解
* 元注解
* 自定义注解

### 二、内置注解

#### 2.1 @Override

* 限定重写父类方法，若想要重写父类的一个方法时，需要使用该注解告知编译器我们正在重写一个方法。如此一来，当父类的方法被删除或修改了，编译器会提示错误信息；或者该方法不是重写也会提示错误。
* 主要用于编译时检查。

#### 2.2 @Deprecated

* 标记已过时，当我们想要让编译器知道一个方法已经被弃用(deprecate)时，应该使用这个注解。Java推荐在javadoc中提供信息，告知用户为什么这个方法被弃用了，以及替代方法是什么。当使用了注解为`@Deprecated`的元素时，编译器会报出警告。

#### 2.3 @SuppressWarnings

* 抑制编译器警告，该注解仅仅告知编译器，忽略它们产生了特殊警告。

### 三、元注解

#### 3.1 概述

* 元注解是可以注解到注解上的注解，或者说元注解是一种基本注解，但是它能够应用到其它的注解上面。
* Java5.0定义了4个标准的meta-annotation类型，它们被用来提供对其它 annotation类型作说明。Java5.0定义的元注解有四个

#### 3.2 注解的语法

##### 3.2.1 注解的定义

```java
public @interface TestAnnotation {
}
```

* 它的形式跟接口很类似，不过前面多了一个 @ 符号。上面的代码就创建了一个名字为 TestAnnotaion 的注解。

##### 3.2.2 注解的使用

```java
@TestAnnotation
public class Test {
}
```

* 创建一个类 Test,然后在类定义的地方加上 @TestAnnotation 就可以用 TestAnnotation 注解这个类了。
* 不过，要想注解能够正常工作，还需要了解元注解。

#### 3.3 @Target

* **用于描述注解的使用范围（即：被描述的注解可以用在什么地方）。**表示支持注解的程序元素的种类，一些可能的值有TYPE, METHOD, CONSTRUCTOR, FIELD等等。如果Target元注解不存在，那么该注解就可以使用在任何程序元素之上。

* 取值(ElementType)有：

```java
public enum ElementType {
    //可以给一个类型进行注解，比如类、接口、枚举
    TYPE,

    //可以给属性进行注解
    FIELD,

    //可以给方法进行注解
    METHOD,

    //可以给一个方法内的参数进行注解
    PARAMETER,
	
    //可以给构造方法进行注解
    CONSTRUCTOR,

    //可以给局部变量进行注解
    LOCAL_VARIABLE,

    //可以给一个注解进行注解
    ANNOTATION_TYPE,

    //可以给一个包进行注解
    PACKAGE,

    //从1.8有的，类型参数声明，可以应用于类的泛型声明之处
    TYPE_PARAMETER,

    //从1.8有的，此类型包括类型声明和类型参数声明，是为了方便设计者进行类型检查
    TYPE_USE
}
```

#### 3.4 @Retention

* **表示需要在什么级别保存该注释信息，用于描述注解的生命周期（即：被描述的注**解**在什么范围内有效）**表示注解类型保留时间的长短。

```java
public enum RetentionPolicy {
    //源码级别 源文件保留
    //注解将被编译器丢弃，只存在源码中，其功能是与编译器交互，用于代码检测，
    //如@Override,@SuppressWarings，
    //许多框架如Dragger就是使用这个级别的注解，这个级别的框架额外效率损耗发生在编译时。
    SOURCE,

    //字节码级别 class保留
    //注解存在源码与字节码文件中，主要用于编译时生成而外的文件，如XML，Java文件等，
    //这个级别需要添加JVM加载时候的代理（javaagent），使用代理来动态修改字节码文件。
    CLASS,

 	//运行时级别 运行时保留
    //注解存在源码，字节码与Java虚拟机中，主要用于运行时反射获取相关信息，
    //许多框架如OrmLite就是使用这个级别的注解，这个级别的框架额外的效率损耗发生在程序运行时。
    RUNTIME
}
```

#### 3.5 @Documented

* 表示使用该注解的元素应被javadoc或类似工具文档化，它应用于类型声明，类型声明的注解会影响客户端对注解元素的使用。如果一个类型声明添加了Documented注解，那么它的注解会成为被注解元素的公共API的一部分，@Documented是一个标记注解。

#### 3.6 @Inherited

* 表示一个注解类型会被自动继承，如果用户在类声明的时候查询注解类型，同时类声明中也没有这个类型的注解，那么注解类型会自动查询该类的父类，这个过程将会不停地重复，直到该类型的注解被找到为止，或是到达类结构的顶层（Object）。

##### 3.6.1 实例

```java
@Inherited
@Retention(RetentionPolicy.RUNTIME)
@interface Test {}

@Test
public class A {}

public class B extends A {}
```

* 注解 Test 被 @Inherited 修饰，之后类 A 被 Test 注解，类 B 继承 A,类 B 也拥有 Test 这个注解。

#### 3.7 @Repeatable

* @Repeatable 自然是可重复的意思。@Repeatable 是 Java 1.8 才加进来的，所以算是一个新的特性。

##### 3.7.1 实例

* 什么样的注解会多次应用呢？通常是注解的值可以同时取多个。
* 举个例子，一个人他既是程序员又是产品经理,同时他还是个画家。

```java
@interface Persons {
    Person[] value();
}

@Repeatable(Persons.class)
@interface Person{
    String role default "";
}

@Person(role="artist")
@Person(role="coder")
@Person(role="PM")
public class SuperMan{
}
```

* 注意上面的代码，@Repeatable 注解了 Person。而 @Repeatable 后面括号中的类相当于一个容器注解。

* 什么是容器注解呢？就是用来存放其它注解的地方。它本身也是一个注解。

* 我们再看看代码中的相关容器注解。

```java
@interface Persons {
    Person[] value();
}
```

* 按照规定，它里面必须要有一个 value 的属性，属性类型是一个被 @Repeatable 注解过的注解数组，注意它是数组。

* 如果不好理解的话，可以这样理解。Persons 是一张总的标签，上面贴满了 Person 这种同类型但内容不一样的标签。把 Persons 给一个 SuperMan 贴上，相当于同时给他贴了程序员、产品经理、画家的标签

### 四、注解参数与默认值

#### 4.1 注解参数

##### 4.1.1 概述

* 注解的属性也叫做成员变量。注解只有成员变量，没有方法。注解的成员变量在注解的定义中以“无形参的方法”形式来声明，其方法名定义了该成员变量的名字，其返回值定义了该成员变量的类型。

##### 4.1.2 实例

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface TestAnnotation {
    int id();
    String msg();
}
```

* 上面代码定义了 TestAnnotation 这个注解中拥有 id 和 msg 两个参数。在使用的时候，我们应该给它们进行赋值。
* 赋值的方式是在注解的括号内以 value=”” 形式，多个参数之前用 ，隔开。

```java
@TestAnnotation(id=3,msg="hello annotation")
public class Test {
}
```

##### 4.1.3 注解类型

* 基本数据类型
* String
* 枚举类型
* 注解类型
* Class类型
* 以上类型的一维数组类型

#### 4.2 默认值

##### 4.2.1 概述

* 注解中参数可以有默认值，默认值需要用 default 关键值指定。

##### 4.2.2 实例1

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface TestAnnotation {
    public int id() default -1;
    public String msg() default "Hi";
}
```

* TestAnnotation 中 id 属性默认值为 -1，msg 属性默认值为 Hi。

```java
@TestAnnotation()
public class Test {}
```

* 因为有默认值，所以无需要再在 @TestAnnotation 后面的括号里面进行赋值了，这一步可以省略。

##### 4.2.3 实例2

* 另外，还有一种情况。如果一个注解内仅仅只有一个名字为 value 的参数时，应用这个注解时可以直接将参数值填写到括号内。

```java
public @interface Check {
    String value();
}
```

* 上面代码中，Check 这个注解只有 value 这个参数。所以可以这样应用。

```java
@Check("hi")
int a;
```

* 等同于

```java
@Check(value="hi")
int a;
```

##### 4.2.4 实例3

* 最后，还需要注意的一种情况是一个注解没有任何参数。比如

```java
public @interface Perform {}
```

* 那么在应用这个注解的时候，括号都可以省略。

```java
@Perform
public void testMethod(){}
```

### 五、注解的提取

#### 5.1 概述

* 注解一般都是通过反射的方式去操作的
* 反射相关详情请查看[Java反射机制](必备Java知识/反射与类加载/反射/Java反射机制.md)

#### 5.2 步骤

* 首先可以通过 Class 对象的 isAnnotationPresent() 方法判断它是否应用了某个注解

```java
public boolean isAnnotationPresent(Class<? extends Annotation> annotationClass) {}
```

* 然后通过 getAnnotation() 方法来获取 Annotation 对象。

```java
 public <A extends Annotation> A getAnnotation(Class<A> annotationClass) {}
```

* 或者是 getAnnotations() 方法。

```java
public Annotation[] getAnnotations() {}
```

* 前一种方法返回指定类型的注解，后一种方法返回注解到这个元素上的所有注解。
* 如果获取到的 Annotation 如果不为 null，则就可以调用它们的属性方法了。

#### 5.3 实例1

* 检阅出在类上的注解
* TestAnnotation.java

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface TestAnnotation {
    public int id() default -1;

    public String msg() default "Hi";
}
```

* Test.java

```java
@TestAnnotation(id = 10)
public class Test {
    public static void main(String[] args) {
        boolean hasAnnotation = Test.class.isAnnotationPresent(TestAnnotation.class);
        if (hasAnnotation) {
            TestAnnotation testAnnotation = Test.class.getAnnotation(TestAnnotation.class);
            System.out.println("id:" + testAnnotation.id());
            System.out.println("msg:" + testAnnotation.msg());
        }
    }
}
```

* 输出结果

```java
id:10
msg:Hi
```

#### 5.4 实例2

* 检阅出在属性、方法上的注解

* check.java

```java
@Retention(RetentionPolicy.RUNTIME)
public @interface Check {
    String value();
}
```

* Perform.java

```java
@Retention(RetentionPolicy.RUNTIME)
public @interface Perform {}
```

* Test.java

```java
@TestAnnotation(msg = "hello")
public class Test {
    @Check(value = "hi")
    int a;

    @Perform
    public void testMethod() {
    }

    public static void main(String[] args) {
        boolean hasAnnotation = Test.class.isAnnotationPresent(TestAnnotation.class);
        if (hasAnnotation) {
            TestAnnotation testAnnotation = Test.class.getAnnotation(TestAnnotation.class);
            //获取类的注解
            System.out.println("id:" + testAnnotation.id());
            System.out.println("msg:" + testAnnotation.msg());
        }
        try {
            Field a = Test.class.getDeclaredField("a");
            a.setAccessible(true);
            //获取一个成员变量上的注解
            Check check = a.getAnnotation(Check.class);
            if (check != null) {
                System.out.println("check value:" + check.value());
            }
            Method testMethod = Test.class.getDeclaredMethod("testMethod");
            // 获取方法中的注解
            Annotation[] ans = testMethod.getAnnotations();
            for (Annotation an : ans) {
                System.out.println("method testMethod annotation:" + an.annotationType().getSimpleName());
            }
        } catch (NoSuchFieldException | NoSuchMethodException | SecurityException e) {
            e.printStackTrace();
            System.out.println(e.getMessage());
        }
    }
}
```

### 六、自定义注解

#### 6.1 使用

* 使用@interface自定义注解时，自动继承了java.lang.annotation.**Annotation**接口，由编译程序自动完成其他细节。在定义注解时，不能继承其他的注解或接口。

##### 6.1.1 定义注解格式

* @interface用来声明一个注解，其中的每一个方法实际上是声明了一个配置参数。方法的名称就是参数的名称，返回值类型就是参数的类型（返回值类型只能是**基本类型、Class、String、enum**）。可以通过default来声明参数的默认值。

```java
public @interface 注解名 {
    定义体S
}
```

##### 6.1.2 注解参数

* 注解里面的每一个方法实际上就是声明了一个配置参数，其规则如下:

**①修饰符**

* 只能用public或默认(default)这两个访问权修饰 ，默认为default

**②类型**

* 注解参数只支持以下数据类型：

* 基本数据类型（int,float,boolean,byte,double,char,long,short)；

* String类型；

* Class类型；

* enum类型；

* Annotation类型;

* 以上所有类型的数组

**③命名**

* 对取名没有要求，如果只有一个参数成员,最好把参数名称设为"**value**",后加小括号。

**④参数**

* 注解中的方法不能存在参数

**⑤默认值**

* 可以包含默认值，使用default来声明默认值

#### 6.2 实例1

* 编写测试注解（模拟的，都是没有参数的方法，运行一下）
* 需要测试的类 NoBug.java

```java
public class NoBug {
    @Jiecha
    public void suanShu(){
        System.out.println("1234567890");
    }
    @Jiecha
    public void jiafa(){
        System.out.println("1+1="+1+1);
    }
    @Jiecha
    public void jiefa(){
        System.out.println("1-1="+(1-1));
    }
    @Jiecha
    public void chengfa(){
        System.out.println("3 x 5="+ 3*5);
    }
    @Jiecha
    public void chufa(){
        System.out.println("6 / 0="+ 6 / 0);
    }
    public void ziwojieshao(){
        System.out.println("我写的程序没有 bug!");
    }
}
```

* 注解类 Jiecha

```java
@Retention(RetentionPolicy.RUNTIME)
public @interface Jiecha {
}
```

* 测试类TestTool

```java
public class TestTool {
    public static void main(String[] args) {
        NoBug testobj = new NoBug();
        Class clazz = testobj.getClass();
        Method[] method = clazz.getDeclaredMethods();
        //用来记录测试产生的 log 信息
        StringBuilder log = new StringBuilder();
        // 记录异常的次数
        int errornum = 0;
        for ( Method m: method ) {
            // 只有被 @Jiecha 标注过的方法才进行测试
            if ( m.isAnnotationPresent( Jiecha.class )) {
                try {
                    m.setAccessible(true);
                    m.invoke(testobj, null);
                } catch (Exception e) {
                    errornum++;
                    log.append(m.getName());
                    log.append(" ");
                    log.append("has error:");
                    log.append("\n\r  caused by ");
                    //记录测试过程中，发生的异常的名称
                    log.append(e.getCause().getClass().getSimpleName());
                    log.append("\n\r");
                    //记录测试过程中，发生的异常的具体信息
                    log.append(e.getCause().getMessage());
                    log.append("\n\r");
                } 
            }
        }
        log.append(clazz.getSimpleName());
        log.append(" has  ");
        log.append(errornum);
        log.append(" error.");
        // 生成测试报告
        System.out.println(log.toString());
    }
}
```

* 输出结果

```java
1234567890
1+1=11
1-1=0
3 x 5=15
chufa has error:
  caused by ArithmeticException
/ by zero
NoBug has  1 error.
```

* 提示 NoBug 类中的 chufa() 这个方法有异常，这个异常名称叫做 ArithmeticException，原因是运算过程中进行了除 0 的操作。