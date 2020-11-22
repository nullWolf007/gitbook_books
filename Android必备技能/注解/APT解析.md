[TOC]

# APT解析

### 参考链接

* [**Java编译时注解处理器（APT）详解**](https://blog.csdn.net/qq_20521573/article/details/82321755)
* [**【Android】注解框架（三）-- 编译时注解，手写ButterKnife**](https://www.jianshu.com/p/57211e053d0c)
* [Java注解实战之APT构建模块化的第一步](https://blog.51cto.com/14332859/2394970)
* [**JavaPoet简介**](https://blog.csdn.net/Viiou/article/details/86388268)

## 一、APT简介

### 1.1 什么是APT

* APT即为Annotation Processing Tool，它是javac的一个工具，即编译时注解处理器。APT可以用来在编译时扫描和处理注解。通过APT可以获取到注解和被注解对象的相关信息，在拿到这些信息后我们可以根据需求来自动的生成一些代码，省去了手动编写。注意，获取注解及生成代码都是在代码编译时完成的，相比反射在运行时处理注解大大提高了程序性能。APT的核心时AbstractProcessor类

### 1.2 涉及到的第三方库

* AutoService：这个库主要作用是注册注解，并对其生成META-INF的配置信息
* JavaPoet：这个库的主要作用是帮助我们通过类调用的形式生成Java代码

### 1.3 哪里用到APT？

* APT技术被广范的运用在Java框架中，包括Android项目以及Java后台项目，对于Android项目像EventBus、ButterKnife、Dagger2等都是用到了APT技术

### 1.4 如何在AS中构建APT项目

#### 1.3.1 说明

* 需要两个Java Library(new一个Module就行)，一般就是Annotation模块和Compiler模块

* Annotation模块，用来存放自定义的注解

* Compiler模块，这个模块依赖Annotation模块

* 主项目模块，需要依赖Annotation模块，同时需要通过annotationProcessor依赖Compiler模块

* 为什么要强调上述两个模块一定要是Java Library？如果创建Android Library模块你会发现不能找到AbstractProcessor这个类，这是因为Android平台是基于OpenJDK的，而OpenJDK中不包含APT的相关代码。因此，在使用APT时，必须在Java Library中进行。

#### 1.3.2 APT流程图

![APT流程图](https://github.com/nullWolf007/images/raw/master/android/%E8%BF%9B%E9%98%B6/AOP%E5%92%8CIOC/APT%E6%B5%81%E7%A8%8B%E5%9B%BE.png)

  

## 二、AutoService和JavaPoet

### 2.1 JavaPoet

#### 2.1.1 常用类

|       类       |                    含义                     |
| :------------: | :-----------------------------------------: |
|   MethodSpec   |         代表一个构造函数或方法声明          |
|    TypeSpec    |       代表一个类、接口，或者枚举声明        |
|   FieldSpec    |       代表一个成员变量，一个字段声明        |
|    JavaFile    |          包含一个顶级类的Java文件           |
| ParameterSpec  |                用来创建参数                 |
| AnnotationSpec |                用来创建注解                 |
|    TypeName    | 类型，如在添加返回值类型是使用TypeName.VOID |
|   ClassName    |               用来包装一个类                |

#### 2.1.2 常用方法

|            方法             |      说明      |
| :-------------------------: | :------------: |
|       addModifiers()        | 设置修饰关键字 |
|       addAnnotation()       |  设置注解对象  |
|        addJavadoc()         |    设置注释    |
|     addSuperinterface()     |    实现接口    |
|        superclass()         |     继承类     |
|         addMethod()         |    添加方法    |
|         addField()          |     添加域     |
|  addCode()/addStatement()   |   添加方法体   |
|   TypeSpec.classBuilder()   |     创建类     |
| TypeSpec.interfaceBuilder() |    创建接口    |
|   TypeSpec.enumBuilder()    |    创建枚举    |

## 三、APT知识点

### 3.1 ProcessingEnvironment相关方法

|       方法        |                          说明                          |
| :---------------: | :----------------------------------------------------: |
| getElementUtils() |    返回实现Elements接口的对象，用于操作元素的工具类    |
|    getFiler()     |  返回实现Filer接口的对象，用于创建文件、类和辅助文件   |
|   getMessager()   | 返回实现Messager接口的对象，用于报告错误信息、警告信息 |
|   getOptions()    |                   返回指定的参数选项                   |
|  getTypeUtils()   |     返回实现Types接口的对象，用于操作类型的工具类      |

### 3.2 获取注解元素

|            方法            |        说明        |
| :------------------------: | :----------------: |
| getElementsAnnotatedWith() | 返回注解元素的集合 |

### 3.3 Element的方法

|         方法          |                             说明                             |
| :-------------------: | :----------------------------------------------------------: |
|    getAnnotation()    | 返回此元素针对指定里类型的注释(如果存在这样的注释)，否则返回null |
| getEnclosingElement() |           返回封装此元素(非严格意义上)的最里层元素           |
| getEnclosedElement()  |            返回此元素直接封装(非严格意义上)的元素            |
|       getKind()       |                       返回此元素的类型                       |
|    getModifiers()     |                返回此元素的修饰符，不包括注释                |
|    getSimpleName()    |                 返回此元素的简单(未限定)名称                 |
|       asType()        |                     返回此元素定义的类型                     |

## 四、实例

### 4.1 注意点

* 如果Gradle的版本号大于5.X，插件版本大于3.0。则需额外添加`annotationProcessor 'com.google.auto.service:auto-service:1.0-rc3'`。如果版本低于的话，无需额外添加的。

  ```java
  implementation 'com.google.auto.service:auto-service:1.0-rc3'
  annotationProcessor 'com.google.auto.service:auto-service:1.0-rc3'//额外添加的
  implementation 'com.squareup:javapoet:1.10.0'
  implementation project(':mubutterknife-annotations')
  ```

### 4.2 源码

* 查看源码请点击[MuButterKnifeDemo](https://github.com/nullWolf007/MuButterKnife/tree/master/MuButterKnifeDemo)


