[TOC]

## Java类加载

#### 参考

* [Android类加载器与Java类加载器的对比](https://juejin.im/post/6844903940094427150#heading-1)
* [Android进阶解密](https://book.douban.com/subject/30358046/)

### 一、基础知识

#### 1.1 概述

* 类加载器的任务是根据一个类的全限定名来读取此类的二进制字节流到JVM中，然后转换为一个与目标类对应的java.lang.Class对象实例

#### 1.2 四种类加载器

##### 1.2.1 Bootstrap ClassLoader(引导类加载器)

* 引导类加载器主要加载的是JVM自身需要的类，这个类加载使用 **C++语言实现** （这里仅限于Hotspot，也就是JDK1.5之后默认的虚拟机，有很多其他的虚拟机是用Java语言实现的）的，是虚拟机自身的一部分，
* 它负责**将 /lib路径下的核心类库或-Xbootclasspath参数指定的路径下的jar包加载到内存中**，
* 注意：由于虚拟机是按照文件名识别加载jar包的，如rt.jar，如果文件名不被虚拟机识别，即使把jar包丢到lib目录下也是没有作用的(出于安全考虑，Bootstrap启动类加载器只加载包名为**java、javax、sun**等开头的类)。

##### 1.2.2 Extension ClassLoader(扩展类加载器)

* 扩展类加载器是指Sun公司(已被Oracle收购)实现的**sun.misc.Launcher$ExtClassLoader类**，**由Java语言实现**的，是Launcher的静态内部类，
* 它**负责加载/lib/ext目录下或者由系统变量-Djava.ext.dir指定位路径中的类**，开发者可以直接使用标准扩展类加载器。

##### 1.2.3 Application ClassLoader(应用程序类加载器)

* 也称应用程序加载器，是指 Sun公司实现的**sun.misc.Launcher$AppClassLoader**。
* 它**负责加载系统类路径java -classpath或-D java.class.path 指定路径下的类库，也就是我们经常用到的classpath路径**
* 开发者可以直接使用系统类加载器，一般情况下该类加载是程序中默认的类加载器，通过**ClassLoader#getSystemClassLoader()** 方法可以获取到该类加载器。

##### 1.2.4 Custom ClassLoader(自定义类加载器)

* 除了系统提供的类加载器， 还可以自定义类加载器，自定义类加载器通过继承 java. lang. ClassLoader 类的方式来实现自己的类加载器， Extensions ClassLoader 和 AppClassLoader 也继承了 java.lang.ClassLoader 类。

#### 1.3 ClassLoader继承关系

![](..\..\..\images\必备Java知识\反射和类加载\类加载\ClassLoader的继承关系.jpg)

* ClassLoader是一个抽象类，其中定义了 ClassLoader 的主要功能。 
* SecureClassLoader 继承了抽象类 ClassLoader ，但 SecureClassLoader 并不是 ClassLoader 的实现类，而是拓展了 ClassLoader 类加入了权限方面的功能，加强了 ClassLoader 的安全性。
* URLClassLoader 继承自 SecureClassLoader ，可以通过 URL 路径从 jar 文件和文件夹中加载类和资源。
*  ExtClassLoader AppClassLoader 都继承自 URLClassLoader ，它们都是 Launcher 内部类， Launcher是Java 虚拟机的入口应用， ExtClassLoader AppClassLoader 都是在 Launcher 中进行初始化的

### 二、双亲委托模式

#### 2.1 概述

* Java虚拟机对class文件采用的是**按需加载**的方式，也就是说当需要使用该类时才会将它的class文件加载到内存生成class对象，而且加载某个类的class文件时，Java虚拟机采用的是**双亲委派模式**即把请求交由父类处理，它一种任务委派模式。

#### 2.2 含义

* 类加载器查找 Class 所采用的是双亲委托模式，所谓双亲委托模式就是首先判断该Class 是否已经加载，如果没有则不是自身去查找而是委托给父加载器进行查找，这样依次进行递归，直到委托到最顶层的 Bootstrap ClassLoader ，如果 Bootstrap ClassLoader找到了Class，就会直接返回，如果没找到，则继续依次向下查找，如果还没找到则最后会交由自身去查找

#### 2.3 流程图

![双亲委托模式](..\..\..\images\必备Java知识\反射和类加载\类加载\双亲委托模式.jpg)

* 双亲委派模式中的父子关系并非通常所说的类继承关系，而是采用**组合关系**来复用父类加载器的相关代码
* “父类加载器”不能理解为“父类，加载器”，而应该理解为“父，类加载器”
* 其中由于Bootstrap ClassLoader类加载器是由C/C++编写的，所以对Java不可见，所以对ExtClassLoader去获取父加载器的话，返回的是null

#### 2.4 源码解析

##### 2.4.1 loadClass(String)

* 该方法加载指定名称（包括包名）的二进制类型，该方法在JDK1.2之后不再建议用户重写但用户可以直接调用该方法，loadClass()方法是ClassLoader类自己实现的，该方法中的逻辑就是双亲委派模式的实现，其源码如下，loadClass(String name, boolean resolve)是一个重载方法，resolve参数代表是否生成class对象的同时进行解析相关操作。

```java
protected Class<?> loadClass(String name, boolean resolve)
	throws ClassNotFoundException
{
	synchronized (getClassLoadingLock(name)) {
        //先从缓存中查找此类是否已经加载过了 加载过了就不用执行下面操作了
        Class<?> c = findLoadedClass(name);
        if (c == null) {
        	long t0 = System.nanoTime();
            try {
            	if (parent != null) {
                    //如果没找到 则交给父类加载器去加载
                	c = parent.loadClass(name, false);
				} else {
                    //如果返回的是null 则说明此时父加载器是BootstrapClassLoader
                    //所以交给BootstrapClassLoader去加载
                	c = findBootstrapClassOrNull(name);
				}
			} catch (ClassNotFoundException e) {
			}

            if (c == null) {
                long t1 = System.nanoTime();
                // 如果都没有找到，则通过自定义实现的findClass去查找并加载
                c = findClass(name);

                sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                sun.misc.PerfCounter.getFindClasses().increment();
			}
		}
        //是否需要在加载时进行解析
        if (resolve) {
        	resolveClass(c);
		}
        return c;
	}
}
```

* `this.getClass().getClassLoder.loadClass("className")`，这样就可以直接调用`ClassLoader`的`loadClass`方法获取到class对象

##### 2.4.2 findClass(String)

* 在JDK1.2之前，在自定义类加载时，总会去继承ClassLoader类并重写loadClass方法，从而实现自定义的类加载类，但是在JDK1.2之后已不再建议用户去覆盖loadClass()方法，而是建议把自定义的类加载逻辑写在findClass()方法中，从前面的源码可知，findClass()方法是在loadClass()方法中被调用的，当loadClass()方法中父加载器加载失败后，则会调用自己的findClass()方法来完成类加载，这样就可以保证自定义的类加载器也符合双亲委托模式。需要注意的是ClassLoader类中并没有实现findClass()方法的具体代码逻辑，取而代之的是抛出ClassNotFoundException异常，同时应该知道的是findClass方法通常是和defineClass方法一起使用的，ClassLoader类中findClass()方法源码如下：

```java
protected Class<?> findClass(String name) throws ClassNotFoundException {
	throw new ClassNotFoundException(name);
}
```

##### 2.4.3 defineClass(String name,byte[] b, int off, int len)

* name:classname
* b:字节码的byte字节流
* off：开始解析的索引
* len：解析的字符长度

* defineClass()方法是用来将byte字节流解析成JVM能够识别的Class对象(ClassLoader中已实现该方法逻辑)，通过这个方法不仅能够通过class文件实例化class对象，也可以通过其他方式实例化class对象，如通过网络接收一个类的字节码，然后转换为byte字节流创建对应的Class对象，defineClass()方法通常与findClass()方法一起使用，一般情况下，在自定义类加载器时，会直接覆盖ClassLoader的findClass()方法并编写加载规则，取得要加载类的字节码后转换成流，然后调用defineClass()方法生成类的Class对象，

**实例**

```java
protected Class<?> findClass(String name) throws ClassNotFoundException {
	  // 获取类的字节数组
      byte[] classData = getClassData(name);  
      if (classData == null) {
          throw new ClassNotFoundException();
      } else {
	      //使用defineClass生成class对象
          return defineClass(name, classData, 0, classData.length);
      }
  }
```

* **需要注意的是，如果直接调用defineClass()方法生成类的Class对象，这个类的Class对象并没有解析(也可以理解为链接阶段，毕竟解析是链接的最后一步)，其解析操作需要等待初始化阶段进行。**

#### 2.5 好处

##### 2.5.1 共享

* 避免重复加载，如加载过一次 Class ，就不需要再次加载，而是直接读取已经加载的 Class

##### 2.5.2 隔离

* 更加安全，如果不使用双亲委托模式，就可以自定义一个String 类来替代系统的 String 类，这显然会造成安全隐患，采用双亲委托模式会使得系统的 String 类在 Java 虚拟机启动时就被加载，也就无法自定义 String 类来替代系统的 String 类，除非我们修改 加载器搜索类的默认算法。还有一点， 只有两个类名一致并且被同一个类加载器加载的类， Java 虚拟机才会认为它们是同一个类，想要骗过 Java 虚拟机显然不会那么容易。

### 三、自定义类加载器

#### 3.1 步骤

* 定义一个自定义ClassLoader并继承抽象类ClassLoader 
* 重写findClass方法，并在findClass方法中调用defineClass方法。

#### 3.2 使用场景

* 当class文件不在ClassPath路径下，默认系统类加载器无法找到该class文件，在这种情况下我们需要实现一个自定义的ClassLoader来加载特定路径下的class文件生成class对象。
* 当一个class文件是通过网络传输并且可能会进行相应的加密操作时，需要先对class文件进行相应的解密后再加载到JVM内存中，这种情况下也需要编写自定义的ClassLoader并实现相应的逻辑。
* 当需要实现热部署功能时(一个class文件通过不同的类加载器产生不同class对象从而实现热部署功能)，需要实现自定义ClassLoader的逻辑。

#### 3.3 实例

##### 3.3.1 Test.java

* 使用javac工具编译成.class文件，放入D:\lib目录下

```java
public class Test {
    public void test(String words) {
        System.out.println("Test" + words);
    }
}
```

##### 3.3.2 MyClassLoder.java

* 主要就是从文件中获取byte数组
* 然后使用defineClass方法获取Class对象实例

```java
public class MyClassLoader extends ClassLoader {
    private String path;

    public MyClassLoader(String path) {
        this.path = path;
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        Class clazz = null;
        byte[] classData = loadClassData(name);
        if (classData == null) {
            throw new ClassNotFoundException();
        } else {
            clazz = defineClass(name, classData, 0, classData.length);
        }
        return clazz;
    }

    //从文件中获取byte数组
    private byte[] loadClassData(String name) {
        String fileName = getFileName(name);
        File file = new File(path, fileName);
        InputStream in = null;
        ByteArrayOutputStream out = null;

        try {
            in = new FileInputStream(file);
            out = new ByteArrayOutputStream();
            byte[] buffer = new byte[1024];
            int length = 0;
            while ((length = in.read(buffer)) != -1) {
                out.write(buffer, 0, length);
            }
            return out.toByteArray();
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                if (in != null) {
                    in.close();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }

            try {
                if (out != null) {
                    out.close();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        return null;
    }

    private String getFileName(String name) {
        int index = name.lastIndexOf(".");
        //如果没有找到'.'则这届在末尾添加.class
        if (index == -1) {
            return name + ".class";
        } else {
            return name.substring(index + 1) + ".class";
        }
    }
}
```

##### 3.3.3 MyClassLoaderTest.java

```java
public class MyClassLoaderTest {
    public static void main(String[] args) {
        MyClassLoader myClassLoader = new MyClassLoader("D:\\lib");

        try {
            Class clazz = myClassLoader.loadClass("Test");
            if (clazz != null) {
                Object obj = clazz.newInstance();
                System.out.println(obj.getClass().getClassLoader());
                Method method = clazz.getMethod("test", String.class);
                method.invoke(obj, "testWords");
            }
        } catch (ClassNotFoundException | IllegalAccessException | InstantiationException | NoSuchMethodException | InvocationTargetException e) {
            e.printStackTrace();
        }
    }
}
```

* 输出结果

```java
MyClassLoader@154617c
TesttestWords
```

