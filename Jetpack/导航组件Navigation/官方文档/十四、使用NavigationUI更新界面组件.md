[TOC]

# 使用 NavigationUI 更新界面组件

* Navigation 组件包含 [`NavigationUI`](https://developer.android.google.cn/reference/androidx/navigation/ui/NavigationUI) 类。此类包含多种静态方法，可帮助您使用顶部应用栏、抽屉式导航栏和底部导航栏来管理导航。

## 一、顶部应用栏

* [顶部应用栏](https://material.io/design/components/app-bars-top.html)在应用顶部提供了一个固定位置，用于显示当前屏幕的信息和操作。

![显示顶部应用栏的屏幕](https://developer.android.google.cn/topic/libraries/architecture/images/top-app-bar.png)


* **图 1.** 显示顶部应用栏的屏幕。

* 利用 `NavigationUI` 包含的方法，您可以在用户浏览应用的过程中自动更新顶部应用栏中的内容。例如，`NavigationUI` 可使用导航图中的目的地标签及时更新顶部应用栏的标题。

```xml
<navigation>
    <fragment ...
              android:label="Page title">
      ...
    </fragment>
</navigation>
```

* 如果您按下面介绍的顶部应用栏实现方法使用 `NavigationUI`，您可以在标签中使用 `{argName}` 格式，从而根据提供给相应目的地的参数来自动填充您附加到目的地的标签。

* `NavigationUI` 支持以下顶部应用栏类型：
  * [`Toolbar`](https://developer.android.google.cn/reference/android/widget/Toolbar)
  * [`CollapsingToolbarLayout`](https://developer.android.google.cn/reference/com/google/android/material/appbar/CollapsingToolbarLayout)
  * [`ActionBar`](https://developer.android.google.cn/reference/androidx/appcompat/app/ActionBar)

* 如需详细了解应用栏，请参阅[设置应用栏](https://developer.android.google.cn/training/appbar/setting-up)。

### 1.1 AppBarConfiguration

* `NavigationUI` 使用 [`AppBarConfiguration`](https://developer.android.google.cn/reference/androidx/navigation/ui/AppBarConfiguration) 对象管理在应用显示区域左上角的导航按钮的行为。导航按钮的行为会根据用户是否位于顶层目的地而变化。
* 顶层目的地是一组存在层次关系的目的地中的根级或最高级目的地。顶层目的地不会在顶部应用栏中显示“向上”按钮，因为不存在更高等级的目的地。默认情况下，应用的起始目的地是唯一的顶层目的地。

* 当用户位于顶层目的地时，如果目的地使用了 `DrawerLayout`，导航按钮会变为抽屉式导航栏图标 ![img](https://developer.android.google.cn/images/guide/navigation/drawer-icon.png)。如果目的地没有使用 `DrawerLayout`，导航按钮处于隐藏状态。当用户位于任何其他目的地时，导航按钮会显示为向上按钮 ![img](https://developer.android.google.cn/images/guide/navigation/up-button.png)。在配置导航按钮时，如需将起始目的地用作唯一顶层目的地，请创建 `AppBarConfiguration` 对象并传入相应的导航图，如下所示：

```kotlin
val appBarConfiguration = AppBarConfiguration(navController.graph)
```

* 在某些情况下，您可能需要定义多个顶层目的地，而不是使用默认的起始目的地。这种情况的一种常见用例是 `BottomNavigationView`，在此场景中，同级屏幕可能彼此之间并不存在层次关系，并且可能各自有一组相关的目的地。对于这样的情况，您可以改为将一组目的地 ID 传递给构造函数，如下所示：

```kotlin
val appBarConfiguration = AppBarConfiguration(setOf(R.id.main, R.id.profile))
```

### 1.2 创建工具栏

* 如需使用 `NavigationUI` 创建工具栏，请先在主 activity 中定义该工具栏，如下所示：

```xml
<LinearLayout>
    <androidx.appcompat.widget.Toolbar
        android:id="@+id/toolbar" />
    <androidx.fragment.app.FragmentContainerView
        android:id="@+id/nav_host_fragment"
        ... />
    ...
</LinearLayout>
```

* 接下来，从主 activity 的 `onCreate()` 方法中调用 [`setupWithNavController()`](https://developer.android.google.cn/reference/androidx/navigation/ui/NavigationUI#setupWithNavController(android.support.v7.widget.Toolbar, androidx.navigation.NavController, androidx.navigation.ui.AppBarConfiguration))，如以下示例所示：

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    setContentView(R.layout.activity_main)

    ...

    val navController = findNavController(R.id.nav_host_fragment)
    val appBarConfiguration = AppBarConfiguration(navController.graph)
    findViewById<Toolbar>(R.id.toolbar)
        .setupWithNavController(navController, appBarConfiguration)
}
```

* **注意**：使用 `Toolbar` 时，Navigation 组件会自动处理导航按钮的点击事件，因此您无需替换 [`onSupportNavigateUp()`](https://developer.android.google.cn/reference/androidx/appcompat/app/AppCompatActivity#onSupportNavigateUp())。

* 如需将导航按钮配置为在所有目的地都显示为向上按钮，那么在构建您的 `AppBarConfiguration` 时，请为顶层目的地传递一组空白目的地 ID。例如，如果您有第二个 activity，其应在所有目的地的 `Toolbar` 中显示向上按钮，这样做可能就很有用。这样一来，当返回堆栈上没有其他目的地时，用户便可导航回父 activity。您可以使用 [`setFallbackOnNavigateUpListener()`](https://developer.android.google.cn/reference/androidx/navigation/ui/AppBarConfiguration.Builder#setFallbackOnNavigateUpListener(androidx.navigation.ui.AppBarConfiguration.OnNavigateUpListener)) 控制在 `navigateUp()` 不另行执行任何操作时的回退行为，如以下示例所示：

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    ...

    val navHostFragment =
        supportFragmentManager.findFragmentById(R.id.nav_host_fragment) as NavHostFragment
    val navController = navHostFragment.navController
    val appBarConfiguration = AppBarConfiguration(
        topLevelDestinationIds = setOf(),
        fallbackOnNavigateUpListener = ::onSupportNavigateUp
    )
    findViewById<Toolbar>(R.id.toolbar)
        .setupWithNavController(navController, appBarConfiguration)
}
```

### 1.3 包含 CollapsingToolbarLayout

* 如需在工具栏中添加 `CollapsingToolbarLayout`，请先在 activity 中定义工具栏和周围布局，如下所示：

```xml
<LinearLayout>
    <com.google.android.material.appbar.AppBarLayout
        android:layout_width="match_parent"
        android:layout_height="@dimen/tall_toolbar_height">

        <com.google.android.material.appbar.CollapsingToolbarLayout
            android:id="@+id/collapsing_toolbar_layout"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            app:contentScrim="?attr/colorPrimary"
            app:expandedTitleGravity="top"
            app:layout_scrollFlags="scroll|exitUntilCollapsed|snap">

            <androidx.appcompat.widget.Toolbar
                android:id="@+id/toolbar"
                android:layout_width="match_parent"
                android:layout_height="?attr/actionBarSize"
                app:layout_collapseMode="pin"/>
        </com.google.android.material.appbar.CollapsingToolbarLayout>
    </com.google.android.material.appbar.AppBarLayout>

    <androidx.fragment.app.FragmentContainerView
        android:id="@+id/nav_host_fragment"
        ... />
    ...
</LinearLayout>
```

* 接着，通过主 activity 的 `onCreate` 方法调用 [`setupWithNavController()`](https://developer.android.google.cn/reference/androidx/navigation/ui/NavigationUI#setupWithNavController(android.support.v7.widget.Toolbar, androidx.navigation.NavController, androidx.navigation.ui.AppBarConfiguration))，如下所示：

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    setContentView(R.layout.activity_main)

    ...

    val layout = findViewById<CollapsingToolbarLayout>(R.id.collapsing_toolbar_layout)
    val toolbar = findViewById<Toolbar>(R.id.toolbar)
    val navHostFragment =
        supportFragmentManager.findFragmentById(R.id.nav_host_fragment) as NavHostFragment
    val navController = navHostFragment.navController
    val appBarConfiguration = AppBarConfiguration(navController.graph)
    layout.setupWithNavController(toolbar, navController, appBarConfiguration)
}
```

### 1.4 操作栏

* 如需向默认操作栏添加导航支持，请通过主 activity 的 `onCreate()` 方法调用 [`setupActionBarWithNavController()`](https://developer.android.google.cn/reference/androidx/navigation/ui/NavigationUI#setupActionBarWithNavController(android.support.v7.app.AppCompatActivity, androidx.navigation.NavController, androidx.navigation.ui.AppBarConfiguration))，如下所示。请注意，您需要在 `onCreate()` 之外声明 `AppBarConfiguration`，因为您在替换 `onSupportNavigateUp()` 时也使用该方法：

```kotlin
private lateinit var appBarConfiguration: AppBarConfiguration

...

override fun onCreate(savedInstanceState: Bundle?) {
    ...

    val navHostFragment =
        supportFragmentManager.findFragmentById(R.id.nav_host_fragment) as NavHostFragment
    val navController = navHostFragment.navController
    appBarConfiguration = AppBarConfiguration(navController.graph)
    setupActionBarWithNavController(navController, appBarConfiguration)
}
```

* 接着，替换 `onSupportNavigateUp()` 以处理向上导航：

```kotlin
override fun onSupportNavigateUp(): Boolean {
    val navController = findNavController(R.id.nav_host_fragment)
    return navController.navigateUp(appBarConfiguration)
            || super.onSupportNavigateUp()
}
```

### 1.5 支持应用栏变体

* 如果对于应用中的每个目的地，应用栏的布局都类似，那么向 activity 添加顶部应用栏的效果很好。但是，如果顶部应用栏在不同目的地之间有很大变化，请考虑从 activity 中移除顶部应用栏，并改为在每个目的地 fragment 中进行定义。

* 例如，一个目的地可能使用标准 `Toolbar`，而另一个目的地则使用 `AppBarLayout` 创建带有标签页的更复杂的应用栏，如图 2 所示。

![两个顶部应用栏变体；左侧为标准工具栏，右侧为带有工具栏和标签页的应用栏布局](https://developer.android.google.cn/images/guide/navigation/app-bar-variations.png)

* **图 2.** 两个应用栏变体。左侧为标准 `Toolbar`。右侧为带有 `Toolbar` 和标签页的 `AppBarLayout`。

* 如需使用 `NavigationUI` 在目的地 fragment 中实现此示例，首先请在每个 fragment 布局中定义应用栏，从使用标准工具栏的目的地 fragment 开始：

```xml
<LinearLayout>
    <androidx.appcompat.widget.Toolbar
        android:id="@+id/toolbar"
        ... />
    ...
</LinearLayout>
```

* 接下来，定义使用带有标签页的应用栏的目的地 fragment：

```xml
<LinearLayout>
    <com.google.android.material.appbar.AppBarLayout
        ... />

        <androidx.appcompat.widget.Toolbar
            android:id="@+id/toolbar"
            ... />

        <com.google.android.material.tabs.TabLayout
            ... />

    </com.google.android.material.appbar.AppBarLayout>
    ...
</LinearLayout>
```

* 这两个 fragment 的导航配置逻辑相同，不过您应该在每个 fragment 的 `onViewCreated()` 方法中调用 [`setupWithNavController()`](https://developer.android.google.cn/reference/androidx/navigation/ui/NavigationUI#setupWithNavController(androidx.appcompat.widget.Toolbar, androidx.navigation.NavController))，而不是通过 activity 对它们进行初始化：

```kotlin
override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
    val navController = findNavController()
    val appBarConfiguration = AppBarConfiguration(navController.graph)

    view.findViewById<Toolbar>(R.id.toolbar)
            .setupWithNavController(navController, appBarConfiguration)
}
```

* **注意**：设置 fragment 过渡时，如果将顶部应用栏放入目的地 fragment 布局中，会导致在 fragment 过渡期间应用栏与布局的其余部分一起生成动画。

## 二、将目的地关联到菜单项

* `NavigationUI` 还提供帮助程序，用于将目的地关联到菜单驱动的界面组件。`NavigationUI` 包含一个帮助程序方法 [`onNavDestinationSelected()`](https://developer.android.google.cn/reference/androidx/navigation/ui/NavigationUI#onnavdestinationselected)，该方法使用 [`MenuItem`](https://developer.android.google.cn/reference/android/view/MenuItem) 以及托管关联目的地的 [`NavController`](https://developer.android.google.cn/reference/androidx/navigation/NavController)。如果 `MenuItem` 的 `id` 与目的地的 `id` 匹配，`NavController` 可以导航至该目的地。

* 例如，下面的 XML 代码段使用常见的 `id` 和 `details_page_fragment` 定义菜单项及目的地：

```xml
<?xml version="1.0" encoding="utf-8"?>
<navigation xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    xmlns:android="http://schemas.android.com/apk/res/android"
    ... >

    ...

    <fragment android:id="@+id/details_page_fragment"
         android:label="@string/details"
         android:name="com.example.android.myapp.DetailsFragment" />
</navigation>

<menu xmlns:android="http://schemas.android.com/apk/res/android">

    ...

    <item
        android:id="@+id/details_page_fragment"
        android:icon="@drawable/ic_details"
        android:title="@string/details" />
</menu>
```

* 例如，如果通过 activity 的 `onCreateOptionsMenu()` 添加菜单，则可以通过替换 activity 的 `onOptionsItemSelected()` 以调用 `onNavDestinationSelected()`，从而将菜单项与目的地相关联，如以下示例所示：

```kotlin
override fun onOptionsItemSelected(item: MenuItem): Boolean {
    val navController = findNavController(R.id.nav_host_fragment)
    return item.onNavDestinationSelected(navController) || super.onOptionsItemSelected(item)
}
```

* 现在，当用户点击 `details_page_fragment` 菜单项时，应用会自动使用相同的 `id` 导航到相应目的地。

## 三、添加抽屉式导航栏

* 抽屉式导航栏是显示应用主导航菜单的界面面板。当用户触摸应用栏中的抽屉式导航栏图标 ![img](https://developer.android.google.cn/images/guide/navigation/drawer-icon.png) 或用户从屏幕的左边缘滑动手指时，就会显示抽屉式导航栏。

![显示导航菜单的打开的抽屉式导航栏](https://developer.android.google.cn/images/topic/libraries/architecture/navigation-drawer.png)

* **图 3.** 显示导航菜单的打开的抽屉式导航栏。

* 抽屉式导航栏图标会显示在使用 `DrawerLayout` 的所有[顶层目的地](https://developer.android.google.cn/guide/navigation/navigation-ui#appbarconfiguration)上。

* 如需添加抽屉式导航栏，请先声明 [`DrawerLayout`](https://developer.android.google.cn/reference/androidx/drawerlayout/widget/DrawerLayout) 为根视图。在 `DrawerLayout` 内，为主界面内容以及包含抽屉式导航栏内容的其他视图添加布局。

* 例如，以下布局使用含有两个子视图的 `DrawerLayout`：包含主内容的 [`NavHostFragment`](https://developer.android.google.cn/reference/androidx/navigation/fragment/NavHostFragment) 和适用于抽屉式导航栏内容的 [`NavigationView`](https://developer.android.google.cn/reference/com/google/android/material/navigation/NavigationView)。

```xml
<?xml version="1.0" encoding="utf-8"?>
<!-- Use DrawerLayout as root container for activity -->
<androidx.drawerlayout.widget.DrawerLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:id="@+id/drawer_layout"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true">

    <!-- Layout to contain contents of main body of screen (drawer will slide over this) -->
    <androidx.fragment.app.FragmentContainerView
        android:name="androidx.navigation.fragment.NavHostFragment"
        android:id="@+id/nav_host_fragment"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:defaultNavHost="true"
        app:navGraph="@navigation/nav_graph" />

    <!-- Container for contents of drawer - use NavigationView to make configuration easier -->
    <com.google.android.material.navigation.NavigationView
        android:id="@+id/nav_view"
        android:layout_width="wrap_content"
        android:layout_height="match_parent"
        android:layout_gravity="start"
        android:fitsSystemWindows="true" />

</androidx.drawerlayout.widget.DrawerLayout>
```

* 接下来，将 [`DrawerLayout`](https://developer.android.google.cn/reference/androidx/drawerlayout/widget/DrawerLayout) 传递给 `AppBarConfiguration`，以将其连接到导航图，如以下示例所示：

```kotlin
val appBarConfiguration = AppBarConfiguration(navController.graph, drawerLayout)
```

* **注意**：当使用 `NavigationUI` 时，[顶部应用栏](https://developer.android.google.cn/guide/navigation/navigation-ui#top-app-bar)帮助程序会随着当前目的地的更改在抽屉式导航栏图标和向上图标之间自动转换。您无需使用 [`ActionBarDrawerToggle`](https://developer.android.google.cn/reference/androidx/appcompat/app/ActionBarDrawerToggle)。

* 接着，在您的主 activity 类中，通过主 activity 的 `onCreate()` 方法调用 [`setupWithNavController()`](https://developer.android.google.cn/reference/androidx/navigation/ui/NavigationUI#setupWithNavController(android.support.design.widget.NavigationView, androidx.navigation.NavController))，如下所示：

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    setContentView(R.layout.activity_main)

    ...

    val navHostFragment =
        supportFragmentManager.findFragmentById(R.id.nav_host_fragment) as NavHostFragment
    val navController = navHostFragment.navController
    findViewById<NavigationView>(R.id.nav_view)
        .setupWithNavController(navController)
}
```

* **注意**：如需设置抽屉式导航栏，还需要设置导航图和菜单 XML，如[将目的地关联到菜单项](https://developer.android.google.cn/guide/navigation/navigation-ui#Tie-navdrawer)中所述。

* 从 [Navigation 2.4.0-alpha01](https://developer.android.google.cn/jetpack/androidx/releases/navigation#2.4.0-alpha01) 开始，系统会在您使用 `setupWithNavController` 时保存并恢复每个菜单项的状态。

## 四、底部导航栏

* `NavigationUI` 也可以处理底部导航。当用户选择某个菜单项时，`NavController` 会调用 [`onNavDestinationSelected()`](https://developer.android.google.cn/reference/androidx/navigation/ui/NavigationUI#onNavDestinationSelected(android.view.MenuItem, androidx.navigation.NavController)) 并自动更新底部导航栏中的所选项目。

![底部导航栏](https://developer.android.google.cn/topic/libraries/architecture/images/bottom-navigation.png)

* **图 4.** 底部导航栏。

* 如需在应用中创建底部导航栏，请先在主 activity 中定义底部导航栏，如下所示：

```xml
<LinearLayout>
    ...
    <androidx.fragment.app.FragmentContainerView
        android:id="@+id/nav_host_fragment"
        ... />
    <com.google.android.material.bottomnavigation.BottomNavigationView
        android:id="@+id/bottom_nav"
        app:menu="@menu/menu_bottom_nav" />
</LinearLayout>
```

* 接着，在您的主 activity 类中，通过主 activity 的 `onCreate()` 方法调用 [`setupWithNavController()`](https://developer.android.google.cn/reference/androidx/navigation/ui/NavigationUI#setupWithNavController(android.support.design.widget.NavigationView, androidx.navigation.NavController))，如下所示：

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    setContentView(R.layout.activity_main)

    ...

    val navHostFragment =
        supportFragmentManager.findFragmentById(R.id.nav_host_fragment) as NavHostFragment
    val navController = navHostFragment.navController
    findViewById<BottomNavigationView>(R.id.bottom_nav)
        .setupWithNavController(navController)
}
```

* **注意**：如需设置底部导航，还需要设置导航图和菜单 XML，如[将目的地关联到菜单项](https://developer.android.google.cn/guide/navigation/navigation-ui#Tie-navdrawer)中所述。

* 从 [Navigation 2.4.0-alpha01](https://developer.android.google.cn/jetpack/androidx/releases/navigation#2.4.0-alpha01) 开始，系统会在您使用 `setupWithNavController` 时保存并恢复每个菜单项的状态。

* 有关包含底部导航栏的综合示例，请参阅 GitHub 上的 [Android 架构组件高级导航示例](https://github.com/android/architecture-components-samples/tree/main/NavigationAdvancedSample)。

## 五、监听导航事件

* 与 [`NavController`](https://developer.android.google.cn/reference/androidx/navigation/NavController) 进行交互是在不同目的地之间导航的主要方法。`NavController` 负责将 [`NavHost`](https://developer.android.google.cn/reference/androidx/navigation/NavHost) 的内容替换为新目的地。在大多数情况下，界面元素（如顶部应用栏或 `BottomNavigationBar` 等其他持续性导航控件）位于 `NavHost` 之外，并且随您在各个目的地之间导航进行更新。

* `NavController` 提供 `OnDestinationChangedListener` 接口，该接口在 `NavController` 的[当前目的地](https://developer.android.google.cn/reference/androidx/navigation/NavController#getCurrentDestination())或其参数发生更改时调用。可以通过 [`addOnDestinationChangedListener()`](https://developer.android.google.cn/reference/androidx/navigation/NavController#addOnDestinationChangedListener(androidx.navigation.NavController.OnDestinationChangedListener)) 方法注册新监听器。请注意，调用 `addOnDestinationChangedListener()` 时，如果当前目的地存在，则会立即被发送到您的监听器。

* `NavigationUI` 使用 `OnDestinationChangedListener` 让这些常见界面组件具备导航感知功能。不过请注意，您也可以单独使用 `OnDestinationChangedListener`，使任何自定义界面或业务逻辑感知导航事件。

* 举例来说，您可能会在应用的一些区域显示常见界面元素，而在另外一些区域隐藏这些元素。使用您自己的 `OnDestinationChangedListener`，您可以根据目标目的地选择性地显示或隐藏这些界面元素，如下例所示：

```kotlin
navController.addOnDestinationChangedListener { _, destination, _ ->
   if(destination.id == R.id.full_screen_destination) {
       toolbar.visibility = View.GONE
       bottomNavigationView.visibility = View.GONE
   } else {
       toolbar.visibility = View.VISIBLE
       bottomNavigationView.visibility = View.VISIBLE
   }
}
```

### 5.1 基于参数的监听器

* 作为替代方案，您还可以将参数与导航图中的默认值结合使用，相应的界面控制器可以使用这些值来更新其状态。例如，我们可以在 `NavGraph` 中创建参数，而不是根据上一个示例，使 `OnDestinationChangedListener` 中的逻辑基于目的地 ID：

```xml
<?xml version="1.0" encoding="utf-8"?>
<navigation xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:id="@+id/navigation\_graph"
    app:startDestination="@id/fragmentOne">
    <fragment
        android:id="@+id/fragmentOne"
        android:name="com.example.android.navigation.FragmentOne"
        android:label="FragmentOne">
        <action
            android:id="@+id/action\_fragmentOne\_to\_fragmentTwo"
            app:destination="@id/fragmentTwo" />
    </fragment>
    <fragment
        android:id="@+id/fragmentTwo"
        android:name="com.example.android.navigation.FragmentTwo"
        android:label="FragmentTwo">
        <argument
            android:name="ShowAppBar"
            android:defaultValue="true" />
    </fragment>
</navigation>
```

* 此参数在[导航到目的地](https://developer.android.google.cn/guide/navigation/navigation-navigate)时不使用，而是用作使用 `defaultValue` 向目的地附加其他信息的一种方法。在本例中，该值指示在此目的地上是否应该显示应用栏。

* 现在，我们可以在 `Activity` 中添加 `OnDestinationChangedListener`：

```kotlin
navController.addOnDestinationChangedListener { _, _, arguments ->
    appBar.isVisible = arguments?.getBoolean("ShowAppBar", false) == true
}
```

* 每当导航目的地发生更改时，[`NavController`](https://developer.android.google.cn/reference/kotlin/androidx/navigation/NavController) 就会调用此回调。`Activity` 现在可以根据回调中收到的参数，更新其拥有的界面组件的状态或可见性。

* 此方法的一个优点在于 `Activity` 只能查看导航图中的参数，而不知道各个 `Fragment` 角色和职责。同样，各个 fragment 也不知道包含的 `Activity` 及其拥有的界面组件。