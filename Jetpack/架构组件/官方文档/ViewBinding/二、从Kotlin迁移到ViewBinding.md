[TOC]

# Migrate from Kotlin synthetics to Jetpack view binding

* Kotlin Android Extensions is deprecated, which means that using Kotlin synthetics for view binding is no longer supported. If your app uses Kotlin synthetics for view binding, use this guide to migrate to Jetpack view binding.
* Kotlin Android Extensions 是不赞成的，意味着使用kotlin来进行视图绑定将不再进行支持与更新，所以如果你的app使用Kotlin的方式进行视图绑定，请根据此教程升级到ViewBinding。

* If your app does not already use Kotlin synthetics for view binding, see [View binding](https://developer.android.google.cn/topic/libraries/view-binding) for basic usage information.

## 一、Update the Gradle file

* Like Android Extensions, Jetpack view binding is enabled on a module by module basis. For each module that uses view binding, set the `viewBinding` build option to `true` in the module-level `build.gradle` file:
* 在build.gradle文件虾，添加如下所示代码

```groovy
android {
    ...
    buildFeatures {
        viewBinding true
    }
}
```

* If your app does not use Parcelize features, remove the line that enables Kotlin Android Extensions:
* 如果你的app没有使用Parcelize 的话，可以把如下代码删除掉

```groovy
apply plugin: `kotlin-android-extensions`
```

* **Note:** If your app uses Parcelize features, you should switch to using the standalone `kotlin-parcelize` Gradle plugin as described in [Parcelable implementation generator](https://developer.android.google.cn/kotlin/parcelize).
* 注意：如果你的app使用了Parcelize ，那么你应该使用kotlin-parcelize去进行替换kotlin-android-extensions

* To learn more about enabling view binding in a module, see [Setup instructions](https://developer.android.google.cn/topic/libraries/view-binding#setup).

## 二、Update activity and fragment classes

* With Jetpack view binding, a binding class is generated for each XML layout file that the module contains. The name of this binding class is the name of the XML file in Pascal case with the word *Binding* added to the end. For example, if the name of the layout file is `result_profile.xml`, the name of the generated binding class is `ResultProfileBinding`.
* 使用JetPack的ViewBinding，为某个模块启用视图绑定功能后，系统会为该模块中包含的每个 XML 布局文件生成一个绑定类。每个绑定类均包含对根视图以及具有 ID 的所有视图的引用。系统会通过以下方式生成绑定类的名称：将 XML 文件的名称转换为驼峰式大小写，并在末尾添加“Binding”一词。
* 例如：`result_profile.xml`生成对应的`ResultProfileBinding`

* Perform the following steps to change your activity and fragment classes to use the generated binding classes instead of synthetic properties to reference views:
* 请按照如下的步骤去设置activity和fragment俩进行升级

1.Remove all imports from `kotlinx.android.synthetic`.

删除所有的imports from `kotlinx.android.synthetic`.

2.Inflate an instance of the generated binding class for the activity or fragment to use.

绑定一个Binding class实例给Activity或者Fragment去使用，如何绑定可以参考上篇文章，概览

- For activities, follow the instructions in [Use view binding in activities](https://developer.android.google.cn/topic/libraries/view-binding#activities) to inflate an instance in your activity's [`onCreate()`](https://developer.android.google.cn/reference/kotlin/android/app/Activity#oncreate) method.
- For fragments, follow the instructions in [Use view binding in fragments](https://developer.android.google.cn/topic/libraries/view-binding#fragments) to inflate an instance in your fragment's [`onCreateView()`](https://developer.android.google.cn/reference/kotlin/androidx/fragment/app/Fragment#oncreateview) method.

3.Change all view references to use the binding class instance instead of synthetic properties:

按照如下的方式修改所有的kotlin视图绑定

```kotlin
// Reference to "name" TextView using synthetic properties.
name.text = viewModel.nameString

// Reference to "name" TextView using the binding class instance.
binding.name.text = viewModel.nameString
```

* To learn more, see the [Usage](https://developer.android.google.cn/topic/libraries/view-binding#usage) section in the view binding guide.