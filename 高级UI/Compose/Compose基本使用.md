[TOC]

## Compose基本使用

#### 转载参考

* [原创|Android Jetpack Compose 最全上手指南](https://juejin.cn/post/6844903999347359751)
* [Jetpack Compose 基础知识](https://developer.android.com/jetpack/compose/tutorial)

### 一、前言

#### 1.1 Jetpack Compose的介绍

* Jetpack Compose 是一个用于构建原生Android UI 的现代化工具包，它基于声明式的编程模型，因此你可以简单地描述UI的外观，而Compose则负责其余的工作-当状态发生改变时，你的UI将自动更新。由于Compose基于Kotlin构建，因此可以与Java编程语言完全互操作，并且可以直接访问所有Android和Jetpack API。它与现有的UI工具包也是完全兼容的，因此你可以混合原来的View和现在新的View，并且从一开始就使用Material和动画进行设计。

#### 1.2 安装AS预览版

* 为了获得最佳 Jetpack Compose 开发体验，您应下载[最新 Canary 版的 Android Studio 预览版](https://developer.android.com/studio/preview)。这是因为，当您搭配使用 Android Studio 和 Jetpack Compose 开发应用时，可以从智能编辑器功能中受益，这些功能包括“新建项目”模板和即时预览 Compose 界面等。

#### 1.3 Jetpack Compose特征

##### 1.3.1 可组合函数

* Jetpack Compose 是围绕可组合函数构建的。这些函数可让您以编程方式定义应用界面，只需描述应用界面的形状和数据依赖关系，而不必关注界面的构建过程。如需创建可组合函数，只需将 `@Composable` 注释添加到函数名称中即可。

### 二、使用配置

#### 2.1 将Jetpack Compose添加到现有项目中

* 如果你想在现有的项目中使用Jetpack Compose，你需要配置一些必须的设置和依赖：

##### 2.1.1 gradle

* 在app目录下的`build.gradle` 中将app支持的最低API 版本设置为21或更高，同时开启Jetpack Compose `enable`开关，代码如下：

```groovy
android {
    defaultConfig {
        ...
        minSdkVersion 21
    }

    buildFeatures {
        // Enables Jetpack Compose for this module
        compose true
    }
    ...

    // Set both the Java and Kotlin compilers to target Java 8.

    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }

    kotlinOptions {
        jvmTarget = "1.8"
    }
    
     composeOptions {
        kotlinCompilerExtensionVersion compose_version
        kotlinCompilerVersion '1.4.32'
    }
}
```

##### 2.1.2 使用Kotlin-Gradle插件

```groovy
// Top-level build file where you can add configuration options common to all sub-projects/modules.
buildscript {
    ext {
        kotlin_version = '1.4.32'
        compose_version = '1.0.0-beta07'
    }
    repositories {
        google()
        jcenter()
        //配置
        maven { url 'https://dl.bintray.com/kotlin/kotlin-eap' }
    }
    dependencies {
        classpath "com.android.tools.build:gradle:4.1.3"
        classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlin_version"
    }
}

allprojects {
    repositories {
        google()
        jcenter()
        //配置
        maven { url 'https://dl.bintray.com/kotlin/kotlin-eap' }
    }
}

task clean(type: Delete) {
    delete rootProject.buildDir
}
```

##### 2.1.3 添加Jetpack Compose工具包依赖项

* 在app目录下的`build.gradle`添加Jetpack Compose 工具包依赖项，代码如下：

```groovy
dependencies {
    // You also need to include the following Compose toolkit dependencies.
    implementation 'androidx.ui:ui-tooling:0.1.0-dev10'
    implementation 'androidx.ui:ui-layout:0.1.0-dev10'
    implementation 'androidx.ui:ui-material:0.1.0-dev10'
    ...
}
```

* 完整的 

```groovy
dependencies {
    implementation 'androidx.compose.ui:ui:1.0.0-beta01'
    // Tooling support (Previews, etc.)
    implementation 'androidx.compose.ui:ui-tooling:1.0.0-beta01'
    // Foundation (Border, Background, Box, Image, Scroll, shapes, animations, etc.)
    implementation 'androidx.compose.foundation:foundation:1.0.0-beta01'
    // Material Design
    implementation 'androidx.compose.material:material:1.0.0-beta01'
    // Material design icons
    implementation 'androidx.compose.material:material-icons-core:1.0.0-beta01'
    implementation 'androidx.compose.material:material-icons-extended:1.0.0-beta01'
    // Integration with activities
    implementation 'androidx.activity:activity-compose:1.3.0-alpha03'
    // Integration with ViewModels
    implementation 'androidx.lifecycle:lifecycle-viewmodel-compose:1.0.0-alpha02'
    // Integration with observables
    implementation 'androidx.compose.runtime:runtime-livedata:1.0.0-beta01'
    implementation 'androidx.compose.runtime:runtime-rxjava2:1.0.0-beta01'

    // UI Tests
    androidTestImplementation 'androidx.compose.ui:ui-test-junit4:1.0.0-beta01'
}
```

#### 2.2 使用AS创建Compose

如果您想要启动一个默认包含对 Jetpack Compose 的支持的新项目，Android Studio 提供了新项目模板来帮助您入门。如需创建包含 Jetpack Compose 的新项目，请按以下步骤操作：

1. 如果您位于 **Welcome to Android Studio** 窗口中，请点击 **Start a new Android Studio project**。如果您已打开 Android Studio 项目，请从菜单栏中依次选择 **File > New > New Project**。
2. 在 **Select a Project Template** 窗口中，选择 **Empty Compose Activity**，然后点击 **Next**。
3. 在Configure your project窗口中，执行以下操作：
   1. 按照常规方法设置 **Name**、**Package name** 和 **Save location**。
   2. 请注意，在 **Language** 下拉菜单中，**Kotlin** 是唯一可用的选项，因为 Jetpack Compose 仅适用于使用 Kotlin 编写的类。
   3. 在 **Minimum API level dropdown** 菜单中，选择 API 级别 21 或更高级别。
4. 点击 **Finish**。
5. 根据[配置 Gradle](https://developer.android.com/jetpack/compose/setup#configure_gradle) 中所述的方法，验证项目的 `build.gradle` 文件配置是否正确。

### 三、基本使用-可组合函数

#### 3.1 添加文本元素

* 可以通过定义内容块并调用 `Text()` 函数来实现此目的。

* setContent 块定义了 Activity 的布局。我们不使用 XML 文件来定义布局内容，而是调用可组合函数。Jetpack Compose 使用自定义 Kotlin 编译器插件将这些可组合函数转换为应用的界面元素。例如，Compose 界面库定义了 `Text()` 函数；您可以调用该函数在应用中声明文本元素。

* MainActivity.kt

  ```kotlin
  class MainActivity : ComponentActivity() {
      override fun onCreate(savedInstanceState: Bundle?) {
          super.onCreate(savedInstanceState)
          setContent {
              Text(text = "Hello World!!!")
          }
      }
  }
  ```

#### 3.2 定义可组合函数

* 可组合函数只能在其他可组合函数的范围内调用。要使函数成为可组合函数，请添加 `@Composable` 注释。
* 如需尝试此操作，请定义一个 `Greeting()` 函数并向其传递一个名称，然后该函数就会使用该名称配置文本元素。

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            Greeting(name = "nullWolf")
        }
    }

    @Composable
    fun Greeting(name: String) {
        Text(text = "Hello $name")
    }
}
```

#### 3.3. 在AS中预览可组合函数

* [当前的 Canary 版 Android Studio](https://developer.android.com/studio/preview) 允许您在 IDE 中预览可组合函数，而无需将应用下载到 Android 设备或模拟器中。主要限制在于，可组合函数不能接受任何参数。因此，您无法直接预览 `Greeting()` 函数，而是需要创建另一个名为 `PreviewGreeting()` 的函数，由该函数使用适当的参数调用 `Greeting()`。请在 `@Composable` 上方添加 `@Preview` 注释。

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            Greeting(name = "nullWolf")
        }
    }

    @Composable
    fun Greeting(name: String) {
        Text(text = "Hello $name")
    }

    @Preview
    @Composable
    fun Preview(){
        Greeting(name = "nullWolf")
    }
}
```

* 重新构建您的项目。由于新的 `previewGreeting()` 函数未在任何位置受到调用，因此应用本身不会更改，但 Android Studio 会添加一个预览窗口。此窗口会显示由标有 `@Preview` 注释的可组合函数创建的界面元素的预览。任何时候，如需更新预览，请点击预览窗口顶部的**刷新**按钮。

![AS预览Compose](..\..\images\高级UI\Compose\AS预览Compose.PNG)

### 四、基本使用-布局

#### 4.1 前言

* 界面元素采用多层次结构，元素中又包含其他元素。在 Compose 中，您可以通过从可组合函数中调用其他可组合函数来构建界面层次结构。
* 在Android的`xml`布局中，如果要显示一个垂直结构的布局，最长用的就是`LinearLayout`, 设置`android:orientation` 值为`vertical`, 子元素就会垂直排列，那么，在Jetpack Compose 中，如何来实现垂直布局呢？先添加几个`Text`来看一下。

#### 4.2 添加多个Text

* 返回到您的 Activity，用新的 `NewsStory()` 函数替换 `Greeting()` 函数。在本教程的其余部分，您将修改该 `NewsStory()` 函数，并且不会再更改 `Activity` 代码。
* 最佳做法是单独创建不会被应用调用的预览函数；专门的预览函数可以提高性能，并且有利于以后更轻松地设置多个预览。因此，请创建一个默认预览函数，该函数的唯一用途就是调用 `NewsStory()` 函数。随着您按照本教程对 `NewsStory()` 进行更改，预览内容会反映您所做的更改。

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            NewStory()
        }
    }

    @Composable
    fun NewStory() {
        Text("A day in Shark Fin Cove")
        Text("Davenport, California")
        Text("December 2018")
    }

    @Preview
    @Composable
    fun DefaultPreview() {
        NewStory()
    }
}
```

<img src="..\..\images\高级UI\Compose\多个text展示.png" alt="多个text展示" style="zoom: 25%;" />

* 这段代码会在内容视图中创建三个文本元素。但是，由于我们未提供有关如何排列这三个文本元素的信息，因此它们会相互重叠，使文本无法阅读。

#### 4.3 使用Column

* `Column` 函数可让您垂直堆叠元素。向 `NewsStory()` 函数中添加一个 `Column`。

```kotlin
    @Composable
    fun NewStory() {
        Column {
            Text("A day in Shark Fin Cove")
            Text("Davenport, California")
            Text("December 2018")
        }
    }
```

<img src="..\..\images\高级UI\Compose\column展示.png" alt="column展示" style="zoom:25%;" />

* 默认设置会直接将所有子项逐个堆叠起来，中间不留间距。Column 本身位于内容视图的左上角。

#### 4.4 向Column中添加样式设置

* 通过将参数传递给 `Column` 调用，可以配置 Column 的尺寸和位置，以及 Column 的子项的排列方式。
* 该设置具有以下含义：
  - `modifier`：可供您配置布局。本例中使用了一个 `Modifier.padding` 修饰符，将 Column 内嵌在周围的视图中。

```kotlin
    @Composable
    fun NewStory() {
        Column(
            modifier = Modifier.padding(16.dp),
        ) {
            Text("A day in Shark Fin Cove")
            Text("Davenport, California")
            Text("December 2018")
        }
    }
```

<img src="..\..\images\高级UI\Compose\column样式展示.png" alt="column样式展示" style="zoom:25%;" />

#### 4.5 添加图片

* 现在，修改您的 `NewsStory()` 函数。您将添加对 `Image()` 的调用，以将图片放入 `Column`。“foundation”软件包中提供了这些可组合项，您可能需要添加该软件包。请参阅 [Jetpack Compose 设置说明](https://developer.android.com/jetpack/compose/setup)。图片的比例会有问题，但没关系，您可以在下一步中纠正此问题。
* 您还需要为图片提供 `contentDescription`。说明用于确保可访问性。但在类似情况下，如果图片只起装饰效果，可以像我们一样将说明设置为 `null`。

```kotlin
    @Composable
    fun NewStory() {
        Column(
            modifier = Modifier.padding(16.dp),
        ) {
            Image(
                //随便设置一张图片即可 
                painter = painterResource(id = R.drawable.ic_launcher_background),
                contentDescription = null
            )
            Text("A day in Shark Fin Cove")
            Text("Davenport, California")
            Text("December 2018")
        }
    }
```

<img src="..\..\images\高级UI\Compose\image展示.png" alt="image展示" style="zoom:25%;" />

* 图片已添加到布局中，但其尺寸尚未调整为适当大小。如需设置图片样式，请将尺寸 `Modifier` 传递给对 `Image()` 的调用。

  - **`height(180.dp)`**：指定图片的高度。
  - **`fillMaxWidth()`**：指定图片的宽度应足以填充所属布局。
* 您还需要向 `Image()` 传递一个 `contentScale` 参数：

  - `contentScale = ContentScale.Crop`：指定图片应填充 Column 的整个宽度，并根据需要剪裁为适当的高度。

```kotlin
    @Composable
    fun NewStory() {
        Column(
            modifier = Modifier.padding(16.dp),
        ) {
            Image(
                painter = painterResource(id = R.drawable.ic_launcher_background),
                contentDescription = null,
                modifier = Modifier
                    .height(180.dp)
                    .fillMaxWidth(),
                contentScale = ContentScale.Crop
            )
            Text("A day in Shark Fin Cove")
            Text("Davenport, California")
            Text("December 2018")
        }
    }
```



<img src="..\..\images\高级UI\Compose\image样式展示.png" alt="image样式展示" style="zoom:25%;" />

* 添加 `Spacer`，将图片与标题分开。

```kotlin
    @Composable
    fun NewStory() {
        Column(
            modifier = Modifier.padding(16.dp),
        ) {
            Image(
                painter = painterResource(id = R.drawable.ic_launcher_background),
                contentDescription = null,
                modifier = Modifier
                    .height(180.dp)
                    .fillMaxWidth(),
                contentScale = ContentScale.Crop
            )
            Spacer(modifier = Modifier.height(16.dp))
            Text("A day in Shark Fin Cove")
            Text("Davenport, California")
            Text("December 2018")
        }
    }
```

<img src="..\..\images\高级UI\Compose\Image添加Space.png" alt="Image添加Space" style="zoom:25%;" />

### 五、基本使用-Material Design

#### 5.1 前言

* Compose 旨在支持 Material Design 原则。它的许多界面元素都原生支持 Material Design。在本课中，您将使用 Material 控件来设置应用的样式。

#### 5.2 采用形状

* Material Design 系统的关键要素之一就是 `Shape`。使用 `clip()` 函数对图片的四角进行圆角化处理。
* `Shape` 不可见，但图片已被剪裁以匹配 `Shape`，因此现在呈现轻微的圆角

```kotlin
    @Composable
    fun NewStory() {
        Column(
            modifier = Modifier.padding(16.dp),
        ) {
            Image(
                painter = painterResource(id = R.drawable.ic_launcher_background),
                contentDescription = null,
                modifier = Modifier
                    .height(180.dp)
                    .fillMaxWidth()
                    .clip(shape = RoundedCornerShape(4.dp)),
                contentScale = ContentScale.Crop
            )
            Spacer(modifier = Modifier.height(16.dp))
            Text("A day in Shark Fin Cove")
            Text("Davenport, California")
            Text("December 2018")
        }
    }
```

<img src="..\..\images\高级UI\Compose\clip预览.png" alt="clip预览" style="zoom:25%;" />

#### 5.3 设置文本样式

* 借助 Compose，您可以轻松遵循 Material Design 原则。将 `MaterialTheme` 应用到您创建的组件。
* 差别可能不太明显，但文本现在采用了 `MaterialTheme` 的默认文本样式。接下来，对每个文本元素应用特定的段落样式。

```kotlin
    @Composable
    fun NewStory() {
        MaterialTheme {
            val typography = MaterialTheme.typography

            Column(
                modifier = Modifier.padding(16.dp),
            ) {
                Image(
                    painter = painterResource(
                        id = R.drawable.ic_launcher_background
                    ),
                    contentDescription = null,
                    modifier = Modifier
                        .height(180.dp)
                        .fillMaxWidth()
                        .clip(shape = RoundedCornerShape(4.dp)),
                    contentScale = ContentScale.Crop
                )
                Spacer(modifier = Modifier.height(16.dp))
                Text(
                    "A day in Shark Fin Cove",
                    style = typography.h6
                )
                Text(
                    "Davenport, California",
                    style = typography.body2
                )
                Text(
                    "December 2018",
                    style = typography.body2
                )
            }
        }
    }
```

<img src="..\..\images\高级UI\Compose\文本样式标题.png" alt="文本样式标题" style="zoom:25%;" />

* 在本例中，文章的标题很短。但有时，一篇文章的标题很长，我们不希望过长的标题影响应用的外观。尝试更改第一个文本元素。
* 配置文本元素，将长度上限设置为 2 行。如果文本很短，不超过此限制，则此设置没有影响；但如果文本过长，显示的文本就会被自动截短。

```kotlin
                Text(
                    "A day wandering through the sandhills " +
                            "in Shark Fin Cove, and a few of the " +
                            "sights I saw",
                    style = typography.h6,
                    maxLines = 2,
                    overflow = TextOverflow.Ellipsis
                )
```

<img src="..\..\images\高级UI\Compose\设置文本行数和结束方式.png" alt="设置文本行数和结束方式" style="zoom:25%;" />

