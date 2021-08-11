[TOC]

# 使用 ViewPager 创建包含标签页的滑动视图

* 滑动视图允许您通过水平手指手势或滑动在同级屏幕（例如标签页）之间进行导航。此导航模式也称为“水平分页”。本主题介绍了如何创建具有滑动视图（以便在标签页之间切换）的标签页布局，以及如何显示标题条而不是标签页。

* **注意**：如需实现滑动视图，我们建议采用经过改进的 [`ViewPager2`](https://developer.android.google.cn/reference/kotlin/androidx/viewpager2/widget/ViewPager2) 库。如需了解详情，请参阅[使用 ViewPager2 创建包含标签页的滑动视图](https://developer.android.google.cn/guide/navigation/navigation-swipe-view-2)和 [ViewPager2 迁移指南](https://developer.android.google.cn/training/animation/vp2-migration)。

## 一、实现滑动视图

* 您可以使用 AndroidX 的 [`ViewPager`](https://developer.android.google.cn/reference/kotlin/androidx/viewpager/widget/ViewPager) 控件创建滑动视图。如需使用 ViewPager 和标签页，您需要将 [ViewPager](https://developer.android.google.cn/jetpack/androidx/releases/viewpager#androidx-deps) 和[材料组件](https://material.io/develop/android/docs/getting-started/)的依赖项添加到项目中。

* 如需使用 `ViewPager` 设置布局，请将 `<ViewPager>` 元素添加到 XML 布局中。例如，如果滑动视图中的每个页面都应使用整个布局，布局应大致如下所示：

```xml
<androidx.viewpager.widget.ViewPager
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/pager"
    android:layout_width="match_parent"
    android:layout_height="match_parent" />
```

* 若要插入代表各个页面的子视图，您需要将此布局挂接到 [`PagerAdapter`](https://developer.android.google.cn/reference/androidx/viewpager/widget/PagerAdapter)。有以下两种内置适配器可供选择：
  * [`FragmentPagerAdapter`](https://developer.android.google.cn/reference/androidx/fragment/app/FragmentPagerAdapter) - 适用于在固定的少量同级屏幕之间进行导航。
  * [`FragmentStatePagerAdapter`](https://developer.android.google.cn/reference/androidx/fragment/app/FragmentStatePagerAdapter) - 适用于对未知数量的页面进行分页。`FragmentStatePagerAdapter` 会在用户导航至其他位置时销毁 Fragment，从而优化内存使用情况。

* 以下示例展示了如何使用 `FragmentStatePagerAdapter` 在一系列 `Fragment` 对象之间滑动：

```kotlin
class CollectionDemoFragment : Fragment() {
    // When requested, this adapter returns a DemoObjectFragment,
    // representing an object in the collection.
    private lateinit var demoCollectionPagerAdapter: DemoCollectionPagerAdapter
    private lateinit var viewPager: ViewPager

    override fun onCreateView(inflater: LayoutInflater,
            container: ViewGroup?,
            savedInstanceState: Bundle?): View? {
       return inflater.inflate(R.layout.collection_demo, container, false)
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        demoCollectionPagerAdapter = DemoCollectionPagerAdapter(childFragmentManager)
        viewPager = view.findViewById(R.id.pager)
        viewPager.adapter = demoCollectionPagerAdapter
    }
}

// Since this is an object collection, use a FragmentStatePagerAdapter,
// and NOT a FragmentPagerAdapter.
class DemoCollectionPagerAdapter(fm: FragmentManager) : FragmentStatePagerAdapter(fm) {

    override fun getCount(): Int  = 100

    override fun getItem(i: Int): Fragment {
        val fragment = DemoObjectFragment()
        fragment.arguments = Bundle().apply {
            // Our object is just an integer :-P
            putInt(ARG_OBJECT, i + 1)
        }
        return fragment
    }

    override fun getPageTitle(position: Int): CharSequence {
        return "OBJECT ${(position + 1)}"
    }
}

private const val ARG_OBJECT = "object"

// Instances of this class are fragments representing a single
// object in our collection.
class DemoObjectFragment : Fragment() {

   override fun onCreateView(inflater: LayoutInflater,
           container: ViewGroup?,
           savedInstanceState: Bundle?): View {
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

* [`TabLayout`](https://developer.android.google.cn/reference/com/google/android/material/tabs/TabLayout) 提供了一种横向显示标签页的方式。当与 `ViewPager` 结合使用时，`TabLayout` 可以提供一种熟悉的界面，让用户在滑动视图中浏览各个页面。

<img src="https://developer.android.google.cn/images/topic/libraries/architecture/navigation-tab-layout.png" alt="img" style="zoom: 67%;" />

* **图 1**：具有四个标签页的 `TabLayout`。

* 如需在 `ViewPager` 中包含 `TabLayout`，请在 `<ViewPager>` 元素内添加 `<TabLayout>` 元素，如下所示：

```xml
<androidx.viewpager.widget.ViewPager
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/pager"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <com.google.android.material.tabs.TabLayout
        android:id="@+id/tab_layout"
        android:layout_width="match_parent"
        android:layout_height="wrap_content" />

</androidx.viewpager.widget.ViewPager>
```

* 接下来，使用 [`setupWithViewPager()`](https://developer.android.google.cn/reference/com/google/android/material/tabs/TabLayout#setupWithViewPager(android.support.v4.view.ViewPager)) 将 `TabLayout` 与 `ViewPager` 相关联。`TabLayout` 中的各个标签页会自动填充 `PagerAdapter` 中的页面标题：

```kotlin
class CollectionDemoFragment : Fragment() {
    ...
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        val tabLayout = view.findViewById(R.id.tab_layout)
        tabLayout.setupWithViewPager(viewPager)
    }
    ...
}

class DemoCollectionPagerAdapter(fm: FragmentManager) : FragmentStatePagerAdapter(fm) {

    override fun getCount(): Int  = 4

    override fun getPageTitle(position: Int): CharSequence {
        return "OBJECT ${(position + 1)}"
    }
    ...
}
```

* **注意**：如果您具有大量或可能无限多个页面，请将 `TabLayout` 上的 `android:tabMode` 属性设置为“可滚动”。这样可以防止 `TabLayout` 尝试同时在屏幕上放下所有标签页，且允许用户滚动浏览标签页列表。

* 如需了解标签页布局的其他设计准则，请参阅[适用于标签页的 Material Design 文档](https://material.io/design/components/tabs.html)。