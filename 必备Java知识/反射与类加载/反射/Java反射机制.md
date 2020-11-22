[TOC]

## Java反射机制

#### 参考链接

* [**Java高级特性——反射**](https://www.jianshu.com/p/9be58ee20dee)

### 一、前言

#### 1.1 定义

* Java反射机制是在运行状态中，对于任意一个类，都能知道这个类的所有属性和方法；对于任意一个对象，都能够调用它的任意方法和属性；这种动态获取信息以及动态调用对象方法的功能成为Java语言的反射机制。

#### 1.2 用途

* 在日常的第三方应用开发过程中，经常会遇到某个类的成员变量/方法/属性是私有的或是只对系统应用开放，这时候就可以利用Java的反射机制通过反射来获取所需的私有成员或是方法。
* 反射可以使你在写代码的时候可以更加灵活，降低耦合，提高代码的自适应能力。

### 二、反射机制的相关类

|     类名      |                       用途                       |
| :-----------: | :----------------------------------------------: |
|    Class类    | 代表类的实体，在运行的Java应用程序中表示类和接口 |
|    Field类    |    代表类的成员变量（成员变量也称为类的属性）    |
|   Method类    |                   代表类的方法                   |
| Constructor类 |                 代表类的构造方法                 |

#### 2.1 Class类

**获得类的相关方法**

|            方法             |                          用途                          |
| :-------------------------: | :----------------------------------------------------: |
| asSubclass(Class<U> claszz) |         把传递的类的对象转换成代表其子类的对象         |
|            Cast             |            把对象转换成代表类或是接口的对象            |
|      getClassLoader()       |                     获得类的加载器                     |
|        getClasses()         | 返回一个数组，数组中包含该类中所有公共类和接口类的对象 |
|    getDeclaredClasses()     |   返回一个数组，数组中包含该类中所有类和接口类的对象   |
|  forName(String className)  |                **根据类名返回类的对象**                |
|          getName()          |                  获得类的完整路径名字                  |
|        newInstance()        |                      创建类的实例                      |
|        getPackage()         |                      获得类的包名                      |
|       getSimpleName()       |                      获得类的名字                      |
|       getSuperclass()       |               获得当前类继承的父类的名字               |
|       getInterfaces()       |               获得当前类实现的类或是接口               |

**获得类中属性相关的方法**

|             方法              |          用途          |
| :---------------------------: | :--------------------: |
|     getField(String name)     | 获得某个公有的属性对象 |
|          getFields()          | 获得所有公有的属性对象 |
| getDeclaredField(String name) |    获得某个属性对象    |
|      getDeclaredFields()      |    获得所有属性对象    |

**获得类中注解相关的方法**

|                      方法                       |                  用途                  |
| :---------------------------------------------: | :------------------------------------: |
|     getAnnotation(Class<A> annotationClass)     | 返回该类中与参数类型匹配的公有注解对象 |
|                getAnnotations()                 |       返回该类所有的公有注解对象       |
| getDeclaredAnnotation(Class<A> annotationClass) | 返回该类中与参数类型匹配的所有注解对象 |
|            getDeclaredAnnotations()             |         返回该类所有的注解对象         |

**获得类中构造器相关的方法**

|                        方法                        |                 用途                 |
| :------------------------------------------------: | :----------------------------------: |
|     getConstructor(Class...<?> parameterTypes)     | 获得该类中与参数类型匹配公有构造方法 |
|                 getConstructors()                  |     获得该类中的所有公有构造方法     |
| getDeclaredConstructor(Class...<?> parameterTypes) |  获得该类中与参数类型匹配的构造方法  |
|             getDeclaredConstructors()              |         获得该类所有构造方法         |

**返回类中方法相关的方法**

|                            方法                            |          用途          |
| :--------------------------------------------------------: | :--------------------: |
|     getMethod(String name, Class...<?> parameterTypes)     | 获得该类某个公有的方法 |
|                        getMethods()                        | 获得该类所有公有的方法 |
| getDeclaredMethod(String name, Class...<?> parameterTypes) |    获得该类某个方法    |
|                    getDeclaredMethods()                    |    获得该类所有方法    |

**类中其他重要的方法**

|                             方法                             |               用途               |
| :----------------------------------------------------------: | :------------------------------: |
|                        isAnnotation()                        |     如果是注解类型则返回true     |
| isAnnotationPresent(Class<? extends Annotation> annotationClass) | 如果是指定类型注解类型则返回true |
|                      isAnonymousClass()                      |      如果是匿名类则返回true      |
|                          isArray()                           |    如果是一个数组类则返回true    |
|                           isEnum()                           |      如果是枚举类则返回true      |
|                    isInstance(Object obj)                    |  如果obj是该类的实例则返回true   |
|                        isInterface()                         |      如果是接口类则返回true      |
|                        isLocalClass()                        |      如果是局部类则返回true      |
|                       isMemberClass()                        |      如果是内部类则返回true      |

#### 2.2 Field类

|             方法              |          用途           |
| :---------------------------: | :---------------------: |
|      equals(Object obj)       | 属性与obj相等则返回true |
|        get(Object obj)        |  获得obj中对应的属性值  |
| set(Object obj, Object value) |   设置obj中对应属性值   |

#### 2.3 Method类

|                方法                |                   用途                   |
| :--------------------------------: | :--------------------------------------: |
| invoke(Object obj, Object... args) | 传递object对象及参数调用该对象对应的方法 |

#### 2.4 Constructor类

|              方法               |            用途            |
| :-----------------------------: | :------------------------: |
| newInstance(Object... initargs) | 根据传递的参数创建类的对象 |

#### 2.5 其他重要的方法

|            方法             |                             用途                             |
| :-------------------------: | :----------------------------------------------------------: |
| setAccessible(boolean flag) | 值为true表示反射的对象在使用时应该取消 Java 语言访问检查。值为false表示反射的对象应该实施 Java 语言访问检查。 |

### 三、常用函数实例

#### 3.1 获取Class对象三种方法

##### 3.1.1 Class.forName

```java
Class<?> cls = Class.forName("com.example.student");//forName（包名.类名）
Student s1 = (Student)cls.newInstance();
```

* 通过forName返回类，通过newInstance返回类的实例

##### 3.1.2 对象.getClass

```java
Student s = new Student();    
Class<?> cls = s.getClass();
Student s2 = (Student)cls.newInstance();
```

* 对象s调用getClass()方法返回对象s所对应的Class对象
* 调用newInstance()方法让Class对象在内存中创建对象的实例

##### 3.1.3 类.Class

```java
Class<?> cls = Student.Class();
Student s3 = (Student)cls.newInstance();
```

* 获取指定类型的Class对象，这里是Student
* 调用newInstance()方法让Class对象在内存中创建对应实例，

#### 3.2 注意事项

* 被反射机制加载的类必须有无参数构造方法，否者运行会抛出异常

### 四、实例

#### 4.1 说明

* 创建一个Book类，通过**反射**创建对象、调用私有构造方法、调用私有方法

#### 4.2 Book-实体类

```java
public class Book {
    private String name;
    private Double price;

    public Book() {
    }

    private Book(String name, Double price) {
        this.name = name;
        this.price = price;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Double getPrice() {
        return price;
    }

    public void setPrice(Double price) {
        this.price = price;
    }

    @Override
    public String toString() {
        return "Book{" +
                "name='" + name + '\'' +
                ", price=" + price +
                '}';
    }

    private String printTest(int index) {
        String string = null;
        switch (index) {
            case 1:
                string = "1";
                break;
            case 2:
                string = "2";
                break;
            default:
                string = "3";
        }
        return string;
    }
}
```

#### 4.3 ReflectBook-反射类

```java
public class ReflectBook {
    public static final String TAG = "ReflectBook:";

    public static void main(String[] args) {
        System.out.println("反射公共的构造方法");
        ReflectBook.reflcetNewInstance();
        System.out.println("反射私有的构造方法");
        ReflectBook.reflectPrivateConstructor();
        System.out.println("反射私有方法printTest");
        ReflectBook.reflectPrivateMethod();
    }

    //反射公共的构造方法
    public static void reflcetNewInstance() {
        try {
            Class<?> classBook = Class.forName("Book");//获取类的对象
            Object objectBook = classBook.newInstance();//获取类的实例
            Book book = (Book) objectBook;
            book.setName("少年博览");
            book.setPrice(23.33);
            System.out.println("reflcetNewInstance:" + book.toString());
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InstantiationException e) {
            e.printStackTrace();
        }
    }

    //反射私有的构造方法
    public static void reflectPrivateConstructor() {
        try {
            Class<?> classBook = Class.forName("Book");
            //获得该类中与参数类型匹配的构造方法
            Constructor<?> declaredConstructorBook = classBook.getDeclaredConstructor(String.class, Double.class);
            //抑制Java语言访问检查
            declaredConstructorBook.setAccessible(true);
            Object objectBook = declaredConstructorBook.newInstance("故事会", 66.66);
            Book book = (Book) objectBook;
            System.out.println("reflectPrivateConstructor:" + book.toString());
        } catch (Exception ex) {
            ex.printStackTrace();
        }
    }

    //反射私有方法
    public static void reflectPrivateMethod() {
        try {
            Class<?> classBook = Class.forName("Book");
            //获得该类某个方法
            Method methodBook = classBook.getDeclaredMethod("printTest", int.class);
            //抑制Java语言访问检查
            methodBook.setAccessible(true);
            Object objectBook = classBook.newInstance();
            //传递object对象及参数调用该对象对应的方法
            String string = (String) methodBook.invoke(objectBook, 1);
            System.out.println("reflectPrivateMethod:" + string);

        } catch (Exception ex) {
            ex.printStackTrace();
        }
    }
}
```

 * 输出

```java
反射公共的构造方法
reflcetNewInstance:Book{name='少年博览', price=23.33}
反射私有的构造方法
reflectPrivateConstructor:Book{name='故事会', price=66.66}
反射私有方法printTest
reflectPrivateMethod:1
```



 

   

