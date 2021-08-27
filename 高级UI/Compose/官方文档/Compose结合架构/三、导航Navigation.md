[TOC]

# 使用 Compose 进行导航

* [Navigation 组件](https://developer.android.com/guide/navigation)支持 [Jetpack Compose](https://developer.android.com/jetpack/compose) 应用。您可以在利用 Navigation 组件的基础架构和功能的同时，在可组合项之间导航。

* **注意**：如果您不熟悉 Compose，请先查看 [Jetpack Compose](https://developer.android.com/jetpack/compose) 资源，然后再继续。

## 一、设置

* 如需支持 Compose，请在应用模块的 `build.gradle` 文件中使用以下依赖项：

```groovy
dependencies {
    implementation "androidx.navigation:navigation-compose:2.4.0-alpha05"
}
```

## 二、使用入门

* `NavController` 是 Navigation 组件的中心 API。此 API 是有状态的，可以跟踪组成应用屏幕的可组合项的返回堆栈以及每个屏幕的状态。

* 您可以通过在可组合项中使用 `rememberNavController()` 方法来创建 `NavController`：

```kotlin
val navController = rememberNavController()
```

* 您应该在可组合项层次结构中的适当位置创建 `NavController`，使所有需要引用它的可组合项都可以访问它。这遵循[状态提升](https://developer.android.com/jetpack/compose/state#state-hoisting)的原则，并且允许您使用 `NavController` 及其通过 `currentBackStackEntryAsState()` 提供的状态作为更新屏幕外的可组合项的可信来源。有关此功能的示例，请参阅[与底部导航栏集成](https://developer.android.com/jetpack/compose/navigation#bottom-nav)。

* **注意**：如果您要使用基于 fragment 的 Navigation 组件，就不必在 Compose 中定义新的导航图，也不必使用 `NavHost` 可组合项。如需了解详情，请参阅[互操作性](https://developer.android.com/jetpack/compose/navigation#interoperability)。

## 三、创建 NavHost

* 每个 `NavController` 都必须与一个 `NavHost` 可组合项相关联。`NavHost` 将 `NavController` 与导航图相关联，后者用于指定您应能够在其间进行导航的可组合项目的地。当您在可组合项之间进行导航时，`NavHost` 的内容会自动进行[重组](https://developer.android.com/jetpack/compose/mental-model#recomposition)。导航图中的每个可组合项目的地都与一个路线相关联。

* **关键术语**：**路线**是一个 `String`，用于定义指向可组合项的路径。您可以将其视为指向特定目的地的隐式深层链接。每个目的地都应该有一条唯一的路线。

* 如需创建 `NavHost`，您需要使用之前通过 `rememberNavController()` 创建的 `NavController`，以及导航图的起始目的地的路线。`NavHost` 创建使用 [Navigation Kotlin DSL](https://developer.android.com/guide/navigation/navigation-kotlin-dsl#navgraphbuilder) 中的 lambda 语法来构建导航图。您可以使用 `composable()` 方法向导航结构添加内容。此方法需要您提供一个路线以及应关联到相应目的地的可组合项：

```kotlin
NavHost(navController = navController, startDestination = "profile") {
    composable("profile") { Profile(/*...*/) }
    composable("friendslist") { FriendsList(/*...*/) }
    /*...*/
}
```

* **注意**：Navigation 组件要求您遵循[导航原则](https://developer.android.com/guide/navigation/navigation-principles#fixed_start_destination)并使用固定的起始目的地。您不应为 `startDestination` 路线使用可组合项值。

## 四、导航到可组合项

* 如需导航到导航图中的可组合项目的地，您必须使用 `navigate()` 方法。`navigate()` 接受代表目的地路线的单个 `String` 参数。如需从导航图中的某个可组合项进行导航，请调用 `navigate()`：

```kotlin
@Composable
fun Profile(navController: NavController) {
    /*...*/
    Button(onClick = { navController.navigate("friends") }) {
        Text(text = "Navigate next")
    }
    /*...*/
}
```

* 您应仅在回调中调用 `navigate()`，而不能在可组合项本身中调用它，以避免每次重组时都调用 `navigate()`。

* 默认情况下，`navigate()` 会将您的新目的地添加到返回堆栈中。您可以通过向我们的 `navigate()` 调用附加其他导航选项来修改 `navigate` 的行为：

```kotlin
// Pop everything up to the "home" destination off the back stack before
// navigating to the "friends" destination
navController.navigate(“friends”) {
    popUpTo("home")
}

// Pop everything up to and including the "home" destination off
// the back stack before navigating to the "friends" destination
navController.navigate("friends") {
    popUpTo("home") { inclusive = true }
}

// Navigate to the "search” destination only if we’re not already on
// the "search" destination, avoiding multiple copies on the top of the
// back stack
navController.navigate("search") {
    launchSingleTop = true
}
```

* 如需查看更多用例，请参阅 [popUpTo 指南](https://developer.android.com/guide/navigation/navigation-navigate#pop)。

* **注意**：[`anim` 块](https://developer.android.com/reference/kotlin/androidx/navigation/NavOptionsBuilder#anim(kotlin.Function1))不能与 Navigation Compose 一起使用。系统会在[此功能请求](https://issuetracker.google.com/172112072)中跟踪 Navigation Compose 中的转换动画。

## 五、使用参数进行导航

* Navigation Compose 还支持在可组合项目的地之间传递参数。为此，您需要向路线中添加参数占位符，就像在使用基础导航库时[向深层链接中添加参数](https://developer.android.com/guide/navigation/navigation-deep-link#implicit)一样。

```kotlin
NavHost(startDestination = "profile/{userId}") {
    ...
    composable("profile/{userId}") {...}
}
```

* 默认情况下，所有参数都会被解析为字符串。您可以使用 `arguments` 参数来设置 `type`，以指定其他类型：

```kotlin
NavHost(startDestination = "profile/{userId}") {
    ...
    composable(
        "profile/{userId}",
        arguments = listOf(navArgument("userId") { type = NavType.StringType })
    ) {...}
}
```

* 您应该从 `composable()` 函数的 lambda 中提供的 `NavBackStackEntry` 中提取 `NavArguments`。

```kotlin
composable("profile/{userId}") { backStackEntry ->
    Profile(navController, backStackEntry.arguments?.getString("userId"))
}
```

* 若要将参数传递到目的地，您需要在 `navigate` 调用中添加路线值而不是占位符：

```kotlin
navController.navigate("profile/user1234")
```

* 如需查看支持的类型的列表，请参阅[在目的地之间传递数据](https://developer.android.com/guide/navigation/navigation-pass-data#supported_argument_types)。

### 5.1 添加可选参数

* Navigation Compose 还支持可选的导航参数。可选参数与必需参数有以下两点不同：
  * 可选参数必须使用查询参数语法 (`"?argName={argName}"`) 来添加
  * 可选参数必须具有 `defaultValue` 集或 `nullability = true`（将默认值隐式设置为 `null`）

* 这意味着，所有可选参数都必须以列表的形式显式添加到 `composable()` 函数：

```kotlin
composable(
    "profile?userId={userId}",
    arguments = listOf(navArgument("userId") { defaultValue = "me" })
) { backStackEntry ->
    Profile(navController, backStackEntry.arguments?.getString("userId"))
}
```

* 现在，即使没有向目的地传递任何参数，系统也会使用“me”的 `defaultValue`。

* 通过路线处理参数的结构意味着可组合项将完全独立于 Navigation，并且更易于测试。

## 六、深层链接

* Navigation Compose 支持隐式深层链接，此类链接也可定义为 `composable()` 函数的一部分。使用 `navDeepLink()` 以列表的形式添加深层链接：

```kotlin
val uri = "https://example.com"

composable(
    "profile?id={id}",
    deepLinks = listOf(navDeepLink { uriPattern = "$uri/{id}" })
) { backStackEntry ->
    Profile(navController, backStackEntry.arguments?.getString("id"))
}
```

* 借助这些深层链接，您可以将特定的网址、操作和/或 MIME 类型与可组合项关联起来。默认情况下，这些深层链接不会向外部应用公开。如需向外部提供这些深层链接，您必须向应用的 `manifest.xml` 文件添加相应的 `<intent-filter>` 元素。如需启用上述深层链接，您应该在清单的 `<activity>` 元素中添加以下内容：

```xml
<activity …>
  <intent-filter>
    ...
    <data android:scheme="https" android:host="www.example.com" />
  </intent-filter>
</activity>
```

* 当其他应用触发该深层链接时，Navigation 会自动深层链接到相应的可组合项。

* 这些深层链接还可用于构建包含可组合项中的相关深层链接的 `PendingIntent`：

```kotlin
val id = "exampleId"
val context = LocalContext.current
val deepLinkIntent = Intent(
    Intent.ACTION_VIEW,
    "https://example.com/$id".toUri(),
    context,
    MyActivity::class.java
)

val deepLinkPendingIntent: PendingIntent? = TaskStackBuilder.create(context).run {
    addNextIntentWithParentStack(deepLinkIntent)
    getPendingIntent(0, PendingIntent.FLAG_UPDATE_CURRENT)
}
```

* 然后，您可以像使用任何其他 `PendingIntent` 一样，使用此 `deepLinkPendingIntent` 在相应深层链接目的地打开您的应用。

## 七、嵌套导航结构

* 您可以将目的地归入一个[嵌套图](https://developer.android.com/guide/navigation/navigation-design-graph#nested_graphs)，以便在应用界面中对特定流程进行模块化。例如，您可以对独立的登录流程进行模块化。

* 嵌套图可以封装其目的地。与根图一样，嵌套图必须具有被其路径标识为起始目的地的目的地。此目的地是当您导航到与嵌套图相关联的路径时所导航到的目的地。

* 如需向 `NavHost` 添加嵌套图，您可以使用 `navigation` 扩展函数：

```kotlin
NavHost(navController, startDestination = startRoute) {
    ...
    navigation(startDestination = nestedStartRoute, route = nested) {
        composable(nestedStartRoute) { ... }
    }
    ...
}
```

## 八、与底部导航栏集成

* 通过在可组合项层次结构中的更高层级定义 `NavController`，您可以将 Navigation 与其他组件（例如 `BottomNavBar`）相关联。这样，您就可以通过选择底部栏中的图标来进行导航。

* 如需将底部导航栏中的项与您的导航图中的路线相关联，建议您定义密封的类（例如此处所示的 `Screen`），其中包含相应目的地的路线和字符串资源 ID。

```kotlin
sealed class Screen(val route: String, @StringRes val resourceId: Int) {
    object Profile : Screen("profile", R.string.profile)
    object FriendsList : Screen("friendslist", R.string.friends_list)
}
```

* 然后，将这些项放置在 `BottomNavigationItem` 可以使用的列表中：

```kotlin
val items = listOf(
   Screen.Profile,
   Screen.FriendsList,
)
```

* 在 `BottomNavigation` 可组合项中，使用 `currentBackStackEntryAsState()` 函数获取当前的 `NavBackStackEntry`。此条目允许您访问当前的 `NavDestination`。然后，可通过 `hierarchy` 辅助方法将该项的路由与当前目的地及其父目的地的路由进行比较来确定每个 `BottomNavigationItem` 的选定状态（以处理使用[嵌套导航](https://developer.android.com/jetpack/compose/navigation#nested-nav)的情况）。

* 该项目的路由还用于将 `onClick` lambda 连接到对 `navigate` 的调用，以便在点按该项时会转到该项。通过使用 `saveState` 和 `restoreState` 标志，当您在底部导航项之间切换时，系统会正确保存并恢复该项的状态和返回堆栈。

```kotlin
val navController = rememberNavController()
Scaffold(
  bottomBar = {
    BottomNavigation {
      val navBackStackEntry by navController.currentBackStackEntryAsState()
      val currentDestination = navBackStackEntry?.destination
      items.forEach { screen ->
        BottomNavigationItem(
          icon = { Icon(Icons.Filled.Favorite, contentDescription = null) },
          label = { Text(stringResource(screen.resourceId)) },
          selected = currentDestination?.hierarchy?.any { it.route == screen.route } == true,
          onClick = {
            navController.navigate(screen.route) {
              // Pop up to the start destination of the graph to
              // avoid building up a large stack of destinations
              // on the back stack as users select items
              popUpTo(navController.graph.findStartDestination().id) {
                saveState = true
              }
              // Avoid multiple copies of the same destination when
              // reselecting the same item
              launchSingleTop = true
              // Restore state when reselecting a previously selected item
              restoreState = true
            }
          }
        )
      }
    }
  }
) { innerPadding ->
  NavHost(navController, startDestination = Screen.Profile.route, Modifier.padding(innerPadding)) {
    composable(Screen.Profile.route) { Profile(navController) }
    composable(Screen.FriendsList.route) { FriendsList(navController) }
  }
}
```

* 在这里，您可以利用 `NavController.currentBackStackEntryAsState()` 方法从 NavHost 函数中获取 navController 状态，并与 `BottomNavigation` 组件共享此状态。这意味着 `BottomNavigation` 会自动拥有最新状态。

## 九、互操作性

* 如果您想将 Navigation 组件与 Compose 配合使用，有以下两种选择：
  * 使用基于 fragment 的 Navigation 组件定义导航图。
  * 使用 Compose 目的地在 Compose 中通过 `NavHost` 定义导航图。只有在导航图中的所有屏幕都是可组合项的情况下，才可以这么做。

* 因此，若要构建混合应用，我们建议使用基于 fragment 的 Navigation 组件，并使用 fragment 存储基于 view 的屏幕、Compose 屏幕和同时使用 view 和 Compose 的屏幕。应用中的每个屏幕 fragment 都是一个可组合项的封装容器，下一步是将所有这些屏幕与 Navigation Compose 结合在一起，并移除所有 fragment。

### 9.1 使用基于 fragment 的 Navigation 从 Compose 导航

* 要在 Compose 代码内更改目的地，您可以公开可传递到并由层次结构中的任何可组合项触发的事件：

```kotlin
@Composable
fun MyScreen(onNavigate: (Int) -> ()) {
    Button(onClick = { onNavigate(R.id.nav_profile) } { /* ... */ }
}
```

* 在您的 fragment 中，您可以通过找到 `NavController` 并导航到目的地，在 Compose 和基于 fragment 的 Navigation 组件之间架起桥梁：

```kotlin
override fun onCreateView( /* ... */ ) {
    setContent {
        MyScreen(onNavigate = { dest -> findNavController().navigate(dest) })
    }
}
```

* 或者，您可以将 `NavController` 传递到 Compose 层次结构下方。不过，公开简单的函数的可重用性和可测试性更高。

## 十、测试

* 强烈建议您将 Navigation 代码与可组合项目的地分离开，以便独立于 `NavHost` 可组合项单独测试每个可组合项。

* 借助 `composable` lambda 提供的间接层，您可以将 Navigation 代码与可组合项本身分离开。这在以下两个方向上可行：
  * 仅将解析后的参数传递到可组合项
  * 传递应由要导航的可组合项触发的 lambda，而不是 `NavController` 本身。

* 例如，如果某个 `Profile` 可组合项接受 `userId` 作为输入，并允许用户导航到好友的个人资料页面，则可以采用以下签名：

```kotlin
@Composable
fun Profile(
    userId: String,
    navigateToFriendProfile: (friendUserId: String) -> Unit
) {
 …
}
```

* 在这里，我们看到 `Profile` 可组合项可独立于 Navigation 运行，从而可单独进行测试。`composable` lambda 会封装弥合 Navigation API 与您的可组合项之间的差距所需的基本逻辑：

```kotlin
composable(
    "profile?userId={userId}",
    arguments = listOf(navArgument("userId") { defaultValue = "me" })
) { backStackEntry ->
    Profile(backStackEntry.arguments?.getString("userId")) { friendUserId ->
        navController.navigate("profile?userId=$friendUserId")
    }
}
```

## 十一、了解详情

* 如需详细了解 Jetpack Navigation，请参阅 [Navigation 组件使用入门](https://developer.android.com/guide/navigation/navigation-getting-started)或参加 [Jetpack Compose Navigation Codelab](https://developer.android.com/codelabs/jetpack-compose-navigation)。