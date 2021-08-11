[TOC]

# 使用 ViewPager2 创建包含标签的滑动视图

* 滑动视图允许您通过水平手指手势或滑动在同级屏幕（例如标签页）之间进行导航。此导航模式也称为“水平分页”。本主题介绍了如何创建具有滑动视图（以便在标签页之间切换）的标签页布局，以及如何显示标题条而不是标签页。

* **注意**：如果您的应用已使用 [`ViewPager`](https://developer.android.google.cn/reference/kotlin/androidx/viewpager/widget/ViewPager)，请参阅[将 ViewPager 迁移到 ViewPager2](https://developer.android.google.cn/training/animation/vp2-migration)。

## 一、实现滑动视图

* 您可以使用 AndroidX 的 [`ViewPager2`](https://developer.android.google.cn/reference/kotlin/androidx/viewpager2/widget/ViewPager2) 微件创建滑动视图。如需使用 ViewPager2 和标签页，您需要将 [ViewPager2](https://developer.android.google.cn/jetpack/androidx/releases/viewpager2#androidx-deps) 和[材料组件](https://material.io/develop/android/docs/getting-started/)的依赖项添加到项目中。

* 如需使用 `ViewPager2` 设置布局，请将 `<ViewPager2>` 元素添加到 XML 布局中。例如，如果滑动视图中的每个页面都应使用整个布局，布局应大致如下所示：

```xml
<androidx.viewpager2.widget.ViewPager2
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/pager"
    android:layout_width="match_parent"
    android:layout_height="match_parent" />
```

* 如需插入代表各个页面的子视图，您需要将此布局挂接到 [`FragmentStateAdapter`](https://developer.android.google.cn/reference/kotlin/androidx/viewpager2/adapter/FragmentStateAdapter)。下面演示了如何用它在一系列 `Fragment` 对象集合中滑动浏览对象：

```kotlin
class CollectionDemoFragment : Fragment() {
    // When requested, this adapter returns a DemoObjectFragment,
    // representing an object in the collection.
    private lateinit var demoCollectionAdapter: DemoCollectionAdapter
    private lateinit var viewPager: ViewPager2

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View? {
        return inflater.inflate(R.layout.collection_demo, container, false)
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        demoCollectionAdapter = DemoCollectionAdapter(this)
        viewPager = view.findViewById(R.id.pager)
        viewPager.adapter = demoCollectionAdapter
    }
}

class DemoCollectionAdapter(fragment: Fragment) : FragmentStateAdapter(fragment) {

    override fun getItemCount(): Int = 100

    override fun createFragment(position: Int): Fragment {
        // Return a NEW fragment instance in createFragment(int)
        val fragment = DemoObjectFragment()
        fragment.arguments = Bundle().apply {
            // Our object is just an integer :-P
            putInt(ARG_OBJECT, position + 1)
        }
        return fragment
    }
}

private const val ARG_OBJECT = "object"

// Instances of this class are fragments representing a single
// object in our collection.
class DemoObjectFragment : Fragment() {

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View {
        return inflater.inflate(R.layout.fragment_collection_object, container, false)
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        arguments?.takeIf { it.containsKey(ARG_OBJECT) }?.apply {
            val textView: TextView = view.findViewById(android.R.id.text1)
            textView.text = getInt(ARG_OBJECT).toString()
        }
    }
}
```

* 下面几部分将介绍如何添加标签页，以帮助简化页面之间的导航。

## 二、使用 TabLayout 添加标签页

* [`TabLayout`](https://developer.android.google.cn/reference/com/google/android/material/tabs/TabLayout) 提供了一种横向显示标签页的方式。当与 `ViewPager2` 结合使用时，`TabLayout` 可以提供一种熟悉的界面，让用户在滑动视图中浏览各个页面。

<img src="https://developer.android.google.cn/images/topic/libraries/architecture/navigation-tab-layout.png" alt="img" style="zoom:33%;" />

* **图 1**：具有四个标签页的 `TabLayout`。

* 如需在 `ViewPager2` 中包含 `TabLayout`，请在 `<ViewPager2>` 元素上方添加 `<TabLayout>` 元素，如下所示：

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <com.google.android.material.tabs.TabLayout
        android:id="@+id/tab_layout"
        android:layout_width="match_parent"
        android:layout_height="wrap_content" />

    <androidx.viewpager2.widget.ViewPager2
        android:id="@+id/pager"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1" />

</LinearLayout>
```

* 接下来，创建 [`TabLayoutMediator`](https://developer.android.google.cn/reference/com/google/android/material/tabs/TabLayoutMediator) 以将 `TabLayout` 与 `ViewPager2` 关联，并按如下所示将它附加到其中：

```kotlin
class CollectionDemoFragment : Fragment() {
    ...
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        val tabLayout = view.findViewById(R.id.tab_layout)
        TabLayoutMediator(tabLayout, viewPager) { tab, position ->
            tab.text = "OBJECT ${(position + 1)}"
        }.attach()
    }
    ...
}
```

* **注意**：如果您具有大量或可能无限多个页面，请将 `TabLayout` 上的 `android:tabMode` 属性设置为“可滚动”。这样可以防止 `TabLayout` 尝试同时在屏幕上放下所有标签页，且允许用户滚动浏览标签页列表。

* 如需了解标签页布局的其他设计准则，请参阅[适用于标签页的 Material Design 文档](https://material.io/design/components/tabs.html)。