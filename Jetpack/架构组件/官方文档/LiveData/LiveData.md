[TOC]

# LiveData 概览 

* [`LiveData`](https://developer.android.google.cn/reference/androidx/lifecycle/LiveData) 是一种可观察的数据存储器类。与常规的可观察类不同，LiveData 具有生命周期感知能力，意指它遵循其他应用组件（如 Activity、Fragment 或 Service）的生命周期。这种感知能力可确保 LiveData 仅更新处于活跃生命周期状态的应用组件观察者。

* **注意**：如需将 LiveData 组件导入您的 Android 项目，请参阅[向项目添加组件](https://developer.android.google.cn/topic/libraries/architecture/adding-components#lifecycle)。

* 如果观察者（由 [`Observer`](https://developer.android.google.cn/reference/androidx/lifecycle/Observer) 类表示）的生命周期处于 [`STARTED`](https://developer.android.google.cn/reference/androidx/lifecycle/Lifecycle.State#STARTED) 或 [`RESUMED`](https://developer.android.google.cn/reference/androidx/lifecycle/Lifecycle.State#RESUMED) 状态，则 LiveData 会认为该观察者处于活跃状态。LiveData 只会将更新通知给活跃的观察者。为观察 [`LiveData`](https://developer.android.google.cn/reference/androidx/lifecycle/LiveData) 对象而注册的非活跃观察者不会收到更改通知。

* 您可以注册与实现 [`LifecycleOwner`](https://developer.android.google.cn/reference/androidx/lifecycle/LifecycleOwner) 接口的对象配对的观察者。有了这种关系，当相应的 [`Lifecycle`](https://developer.android.google.cn/reference/androidx/lifecycle/Lifecycle) 对象的状态变为 [`DESTROYED`](https://developer.android.google.cn/reference/androidx/lifecycle/Lifecycle.State#DESTROYED) 时，便可移除此观察者。这对于 Activity 和 Fragment 特别有用，因为它们可以放心地观察 [`LiveData`](https://developer.android.google.cn/reference/androidx/lifecycle/LiveData) 对象，而不必担心泄露（当 Activity 和 Fragment 的生命周期被销毁时，系统会立即退订它们）。

* 如需详细了解如何使用 LiveData，请参阅[使用 LiveData 对象](https://developer.android.google.cn/topic/libraries/architecture/livedata#work_livedata)。

## 一、使用 LiveData 的优势

- **确保界面符合数据状态**

  LiveData 遵循观察者模式。当底层数据发生变化时，LiveData 会通知 [`Observer`](https://developer.android.google.cn/reference/androidx/lifecycle/Observer) 对象。您可以整合代码以在这些 `Observer` 对象中更新界面。这样一来，您无需在每次应用数据发生变化时更新界面，因为观察者会替您完成更新。

- **不会发生内存泄漏**

  观察者会绑定到 [`Lifecycle`](https://developer.android.google.cn/reference/androidx/lifecycle/Lifecycle) 对象，并在其关联的生命周期遭到销毁后进行自我清理。

- **不会因 Activity 停止而导致崩溃**

  如果观察者的生命周期处于非活跃状态（如返回栈中的 Activity），则它不会接收任何 LiveData 事件。

- **不再需要手动处理生命周期**

  界面组件只是观察相关数据，不会停止或恢复观察。LiveData 将自动管理所有这些操作，因为它在观察时可以感知相关的生命周期状态变化。

- **数据始终保持最新状态**

  如果生命周期变为非活跃状态，它会在再次变为活跃状态时接收最新的数据。例如，曾经在后台的 Activity 会在返回前台后立即接收最新的数据。

- **适当的配置更改**

  如果由于配置更改（如设备旋转）而重新创建了 Activity 或 Fragment，它会立即接收最新的可用数据。

- **共享资源**

  您可以使用单例模式扩展 [`LiveData`](https://developer.android.google.cn/reference/androidx/lifecycle/LiveData) 对象以封装系统服务，以便在应用中共享它们。`LiveData` 对象连接到系统服务一次，然后需要相应资源的任何观察者只需观察 `LiveData` 对象。如需了解详情，请参阅[扩展 LiveData](https://developer.android.google.cn/topic/libraries/architecture/livedata#extend_livedata)。

## 二、使用 LiveData 对象

* 请按照以下步骤使用 [`LiveData`](https://developer.android.google.cn/reference/androidx/lifecycle/LiveData) 对象：

1. 创建 `LiveData` 的实例以存储某种类型的数据。这通常在 [`ViewModel`](https://developer.android.google.cn/reference/androidx/lifecycle/ViewModel) 类中完成。

2. 创建可定义 [`onChanged()`](https://developer.android.google.cn/reference/androidx/lifecycle/Observer#onChanged(T)) 方法的 [`Observer`](https://developer.android.google.cn/reference/androidx/lifecycle/Observer) 对象，该方法可以控制当 `LiveData` 对象存储的数据更改时会发生什么。通常情况下，您可以在界面控制器（如 Activity 或 Fragment）中创建 `Observer` 对象。

3. 使用 [`observe()`](https://developer.android.google.cn/reference/androidx/lifecycle/LiveData#observe(android.arch.lifecycle.LifecycleOwner, android.arch.lifecycle.Observer)) 方法将 `Observer` 对象附加到 `LiveData` 对象。`observe()` 方法会采用 [`LifecycleOwner`](https://developer.android.google.cn/reference/androidx/lifecycle/LifecycleOwner) 对象。这样会使 `Observer` 对象订阅 `LiveData` 对象，以使其收到有关更改的通知。通常情况下，您可以在界面控制器（如 Activity 或 Fragment）中附加 `Observer` 对象。

* **注意**：您可以使用 [`observeForever(Observer)`](https://developer.android.google.cn/reference/androidx/lifecycle/LiveData#observeForever(android.arch.lifecycle.Observer)) 方法在没有关联的 [`LifecycleOwner`](https://developer.android.google.cn/reference/androidx/lifecycle/LifecycleOwner) 对象的情况下注册一个观察者。在这种情况下，观察者会被视为始终处于活跃状态，因此它始终会收到关于修改的通知。您可以通过调用 [`removeObserver(Observer)`](https://developer.android.google.cn/reference/androidx/lifecycle/LiveData#removeObserver(android.arch.lifecycle.Observer)) 方法来移除这些观察者。

* 当您更新存储在 `LiveData` 对象中的值时，它会触发所有已注册的观察者（只要附加的 `LifecycleOwner` 处于活跃状态）。

* LiveData 允许界面控制器观察者订阅更新。当 `LiveData` 对象存储的数据发生更改时，界面会自动更新以做出响应。

### 2.1 创建 LiveData 对象

* LiveData 是一种可用于任何数据的封装容器，其中包括可实现 `Collections` 的对象，如 `List`。[`LiveData`](https://developer.android.google.cn/reference/androidx/lifecycle/LiveData) 对象通常存储在 [`ViewModel`](https://developer.android.google.cn/reference/androidx/lifecycle/ViewModel) 对象中，并可通过 getter 方法进行访问，如以下示例中所示：

```kotlin
class NameViewModel : ViewModel() {

    // Create a LiveData with a String
    val currentName: MutableLiveData<String> by lazy {
        MutableLiveData<String>()
    }

    // Rest of the ViewModel...
}
```

* 最初，`LiveData` 对象中的数据并未经过设置。

> **注意**：请确保用于更新界面的 `LiveData` 对象存储在 `ViewModel` 对象中，而不是将其存储在 Activity 或 Fragment 中，原因如下：避免 Activity 和 Fragment 过于庞大。现在，这些界面控制器负责显示数据，但不负责存储数据状态。将 `LiveData` 实例与特定的 Activity 或 Fragment 实例分离开，并使 `LiveData` 对象在配置更改后继续存在。

* 您可以在 [ViewModel 指南](https://developer.android.google.cn/topic/libraries/architecture/viewmodel)中详细了解 `ViewModel` 类的好处和用法。

### 2.2 观察 LiveData 对象

* 在大多数情况下，应用组件的 `onCreate()` 方法是开始观察 [`LiveData`](https://developer.android.google.cn/reference/androidx/lifecycle/LiveData) 对象的正确着手点，原因如下：
  * 确保系统不会从 Activity 或 Fragment 的 `onResume()` 方法进行冗余调用。
  * 确保 Activity 或 Fragment 变为活跃状态后具有可以立即显示的数据。一旦应用组件处于 [`STARTED`](https://developer.android.google.cn/reference/androidx/lifecycle/Lifecycle.State#STARTED) 状态，就会从它正在观察的 `LiveData` 对象接收最新值。只有在设置了要观察的 `LiveData` 对象时，才会发生这种情况。

* 通常，LiveData 仅在数据发生更改时才发送更新，并且仅发送给活跃观察者。此行为的一种例外情况是，观察者从非活跃状态更改为活跃状态时也会收到更新。此外，如果观察者第二次从非活跃状态更改为活跃状态，则只有在自上次变为活跃状态以来值发生了更改时，它才会收到更新。

* 以下示例代码说明了如何开始观察 `LiveData` 对象：

```kotlin
class NameActivity : AppCompatActivity() {

    // Use the 'by viewModels()' Kotlin property delegate
    // from the activity-ktx artifact
    //implementation "androidx.activity:activity-ktx:1.2.4"
    private val model: NameViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // Other code to setup the activity...

        // Create the observer which updates the UI.
        val nameObserver = Observer<String> { newName ->
            // Update the UI, in this case, a TextView.
            nameTextView.text = newName
        }

        // Observe the LiveData, passing in this activity as the LifecycleOwner and the observer.
        model.currentName.observe(this, nameObserver)
    }
}
```

* 在传递 `nameObserver` 参数的情况下调用 [`observe()`](https://developer.android.google.cn/reference/androidx/lifecycle/LiveData#observe(android.arch.lifecycle.LifecycleOwner, android.arch.lifecycle.Observer)) 后，系统会立即调用 [`onChanged()`](https://developer.android.google.cn/reference/androidx/lifecycle/Observer#onChanged(T))，从而提供 `mCurrentName` 中存储的最新值。如果 `LiveData` 对象尚未在 `mCurrentName` 中设置值，则不会调用 `onChanged()`。

### 2.3 更新 LiveData 对象

* LiveData 没有公开可用的方法来更新存储的数据。[`MutableLiveData`](https://developer.android.google.cn/reference/androidx/lifecycle/MutableLiveData) 类将公开 [`setValue(T)`](https://developer.android.google.cn/reference/androidx/lifecycle/MutableLiveData#setValue(T)) 和 [`postValue(T)`](https://developer.android.google.cn/reference/androidx/lifecycle/MutableLiveData#postValue(T)) 方法，如果您需要修改存储在 [`LiveData`](https://developer.android.google.cn/reference/androidx/lifecycle/LiveData) 对象中的值，则必须使用这些方法。通常情况下会在 [`ViewModel`](https://developer.android.google.cn/reference/androidx/lifecycle/ViewModel) 中使用 `MutableLiveData`，然后 `ViewModel` 只会向观察者公开不可变的 `LiveData` 对象。

* 设置观察者关系后，您可以更新 `LiveData` 对象的值（如以下示例中所示），这样当用户点按某个按钮时会触发所有观察者：

```kotlin
button.setOnClickListener {
    val anotherName = "John Doe"
    model.currentName.setValue(anotherName)
}
```

* 在本示例中调用 `setValue(T)` 导致观察者使用值 `John Doe` 调用其 [`onChanged()`](https://developer.android.google.cn/reference/androidx/lifecycle/Observer#onChanged(T)) 方法。本示例中演示的是按下按钮的方法，但也可以出于各种各样的原因调用 `setValue()` 或 `postValue()` 来更新 `mName`，这些原因包括响应网络请求或数据库加载完成。在所有情况下，调用 `setValue()` 或 `postValue()` 都会触发观察者并更新界面。

* **注意**：您必须调用 [`setValue(T)`](https://developer.android.google.cn/reference/androidx/lifecycle/MutableLiveData#setValue(T)) 方法以从主线程更新 `LiveData` 对象。如果在工作器线程中执行代码，您可以改用 [`postValue(T)`](https://developer.android.google.cn/reference/androidx/lifecycle/MutableLiveData#postValue(T)) 方法来更新 `LiveData` 对象。

### 2.4 将 LiveData 与 Room 一起使用

* [Room](https://developer.android.google.cn/training/data-storage/room) 持久性库支持返回 [`LiveData`](https://developer.android.google.cn/reference/androidx/lifecycle/LiveData) 对象的可观察查询。可观察查询属于数据库访问对象 (DAO) 的一部分。

* 当数据库更新时，Room 会生成更新 `LiveData` 对象所需的所有代码。在需要时，生成的代码会在后台线程上异步运行查询。此模式有助于使界面中显示的数据与存储在数据库中的数据保持同步。您可以在 [Room 持久性库指南](https://developer.android.google.cn/topic/libraries/architecture/room)中详细了解 Room 和 DAO。

### 2.5 将协程与 LiveData 一起使用

* `LiveData` 支持 Kotlin 协程。如需了解详情，请参阅[将 Kotlin 协程与 Android 架构组件一起使用](https://developer.android.google.cn/topic/libraries/architecture/coroutines)。

## 三、扩展 LiveData

* 如果观察者的生命周期处于 [`STARTED`](https://developer.android.google.cn/reference/androidx/lifecycle/Lifecycle.State#STARTED) 或 [`RESUMED`](https://developer.android.google.cn/reference/androidx/lifecycle/Lifecycle.State#RESUMED) 状态，则 LiveData 会认为该观察者处于活跃状态。以下示例代码说明了如何扩展 [`LiveData`](https://developer.android.google.cn/reference/androidx/lifecycle/LiveData) 类：

```kotlin
class StockLiveData(symbol: String) : LiveData<BigDecimal>() {
    private val stockManager = StockManager(symbol)

    private val listener = { price: BigDecimal ->
        value = price
    }

    override fun onActive() {
        stockManager.requestPriceUpdates(listener)
    }

    override fun onInactive() {
        stockManager.removeUpdates(listener)
    }
}
```

* 本示例中的价格监听器实现包括以下重要方法：
  * 当 `LiveData` 对象具有活跃观察者时，会调用 [`onActive()`](https://developer.android.google.cn/reference/androidx/lifecycle/LiveData#onActive()) 方法。这意味着，您需要从此方法开始观察股价更新。
  * 当 `LiveData` 对象没有任何活跃观察者时，会调用 [`onInactive()`](https://developer.android.google.cn/reference/androidx/lifecycle/LiveData#onInactive()) 方法。由于没有观察者在监听，因此没有理由与 `StockManager` 服务保持连接。
  * [`setValue(T)`](https://developer.android.google.cn/reference/androidx/lifecycle/MutableLiveData#setValue(T)) 方法将更新 `LiveData` 实例的值，并将更改告知活跃观察者。

* 您可以使用 `StockLiveData` 类，如下所示：

```kotlin
public class MyFragment : Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        val myPriceListener: LiveData<BigDecimal> = ...
        myPriceListener.observe(viewLifecycleOwner, Observer<BigDecimal> { price: BigDecimal? ->
            // Update the UI.
        })
    }
}
```

* [`observe()`](https://developer.android.google.cn/reference/androidx/lifecycle/LiveData#observe(androidx.lifecycle.LifecycleOwner, androidx.lifecycle.Observer)) 方法将与 Fragment 视图关联的 [`LifecycleOwner`](https://developer.android.google.cn/reference/androidx/lifecycle/LifecycleOwner) 作为第一个参数传递。这样做表示此观察者已绑定到与所有者关联的 [`Lifecycle`](https://developer.android.google.cn/reference/androidx/lifecycle/Lifecycle) 对象，这意味着：
  * 如果 `Lifecycle` 对象未处于活跃状态，那么即使值发生更改，也不会调用观察者。
  * 销毁 `Lifecycle` 对象后，会自动移除观察者。

* `LiveData` 对象具有生命周期感知能力，这一事实意味着您可以在多个 Activity、Fragment 和 Service 之间共享这些对象。为使示例保持简单，您可以将 `LiveData` 类实现为一个单例，如下所示：

```kotlin
class StockLiveData(symbol: String) : LiveData<BigDecimal>() {
    private val stockManager: StockManager = StockManager(symbol)

    private val listener = { price: BigDecimal ->
        value = price
    }

    override fun onActive() {
        stockManager.requestPriceUpdates(listener)
    }

    override fun onInactive() {
        stockManager.removeUpdates(listener)
    }

    companion object {
        private lateinit var sInstance: StockLiveData

        @MainThread
        fun get(symbol: String): StockLiveData {
            sInstance = if (::sInstance.isInitialized) sInstance else StockLiveData(symbol)
            return sInstance
        }
    }
}
```

* 并且您可以在 Fragment 中使用它，如下所示：

```kotlin
class MyFragment : Fragment() {

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        StockLiveData.get(symbol).observe(viewLifecycleOwner, Observer<BigDecimal> { price: BigDecimal? ->
            // Update the UI.
        })

    }
```

* 多个 Fragment 和 Activity 可以观察 `MyPriceListener` 实例。仅当一个或多项系统服务可见且处于活跃状态时，LiveData 才会连接到该服务。

## 四、转换 LiveData

* 您可能希望在将 [`LiveData`](https://developer.android.google.cn/reference/androidx/lifecycle/LiveData) 对象分派给观察者之前对存储在其中的值进行更改，或者您可能需要根据另一个实例的值返回不同的 `LiveData` 实例。[`Lifecycle`](https://developer.android.google.cn/reference/android/arch/lifecycle/package-summary) 软件包会提供 [`Transformations`](https://developer.android.google.cn/reference/androidx/lifecycle/Transformations) 类，该类包括可应对这些情况的辅助程序方法。

- [`Transformations.map()`](https://developer.android.google.cn/reference/androidx/lifecycle/Transformations#map(android.arch.lifecycle.LiveData, android.arch.core.util.Function))

  对存储在 `LiveData` 对象中的值应用函数，并将结果传播到下游。

```kotlin
val userLiveData: LiveData<User> = UserLiveData()
val userName: LiveData<String> = Transformations.map(userLiveData) {
    user -> "${user.name} ${user.lastName}"
}
```

- [`Transformations.switchMap()`](https://developer.android.google.cn/reference/androidx/lifecycle/Transformations#switchMap(android.arch.lifecycle.LiveData, android.arch.core.util.Function>))

  与 `map()` 类似，对存储在 `LiveData` 对象中的值应用函数，并将结果解封和分派到下游。传递给 `switchMap()` 的函数必须返回 `LiveData` 对象，如以下示例中所示：

```kotlin
private fun getUser(id: String): LiveData<User> {
  ...
}
val userId: LiveData<String> = ...
val user = Transformations.switchMap(userId) { id -> getUser(id) }
```

* 您可以使用转换方法在观察者的生命周期内传送信息。除非观察者正在观察返回的 `LiveData` 对象，否则不会计算转换。因为转换是以延迟的方式计算，所以与生命周期相关的行为会隐式传递下去，而不需要额外的显式调用或依赖项。

* 如果您认为 [`ViewModel`](https://developer.android.google.cn/reference/androidx/lifecycle/ViewModel) 对象中需要有 `Lifecycle` 对象，那么进行转换或许是更好的解决方案。例如，假设您有一个界面组件，该组件接受地址并返回该地址的邮政编码。您可以为此组件实现简单的 `ViewModel`，如以下示例代码所示：

```kotlin
class MyViewModel(private val repository: PostalCodeRepository) : ViewModel() {

    private fun getPostalCode(address: String): LiveData<String> {
        // DON'T DO THIS
        return repository.getPostCode(address)
    }
}
```

* 然后，该界面组件需要取消注册先前的 `LiveData` 对象，并在每次调用 `getPostalCode()` 时注册到新的实例。此外，如果重新创建了该界面组件，它会再触发一次对 `repository.getPostCode()` 方法的调用，而不是使用先前调用所得的结果。

* 您也可以将邮政编码查询实现为地址输入的转换，如以下示例中所示：

```kotlin
class MyViewModel(private val repository: PostalCodeRepository) : ViewModel() {
    private val addressInput = MutableLiveData<String>()
    val postalCode: LiveData<String> = Transformations.switchMap(addressInput) {
            address -> repository.getPostCode(address) }

    private fun setInput(address: String) {
        addressInput.value = address
    }
}
```

* 在这种情况下，`postalCode` 字段定义为 `addressInput` 的转换。只要您的应用具有与 `postalCode` 字段关联的活跃观察者，就会在 `addressInput` 发生更改时重新计算并检索该字段的值。

* 此机制允许较低级别的应用创建以延迟的方式按需计算的 `LiveData` 对象。`ViewModel` 对象可以轻松获取对 `LiveData` 对象的引用，然后在其基础之上定义转换规则。

### 4.1 创建新的转换

* 有十几种不同的特定转换在您的应用中可能很有用，但默认情况下不提供它们。如需实现您自己的转换，您可以使用 [`MediatorLiveData`](https://developer.android.google.cn/reference/androidx/lifecycle/MediatorLiveData) 类，该类可以监听其他 [`LiveData`](https://developer.android.google.cn/reference/androidx/lifecycle/LiveData) 对象并处理它们发出的事件。`MediatorLiveData` 正确地将其状态传播到源 `LiveData` 对象。如需详细了解此模式，请参阅 [`Transformations`](https://developer.android.google.cn/reference/androidx/lifecycle/Transformations) 类的参考文档。

## 五、合并多个 LiveData 源

* [`MediatorLiveData`](https://developer.android.google.cn/reference/androidx/lifecycle/MediatorLiveData) 是 [`LiveData`](https://developer.android.google.cn/reference/androidx/lifecycle/LiveData) 的子类，允许您合并多个 LiveData 源。只要任何原始的 LiveData 源对象发生更改，就会触发 `MediatorLiveData` 对象的观察者。

* 例如，如果界面中有可以从本地数据库或网络更新的 `LiveData` 对象，则可以向 `MediatorLiveData` 对象添加以下源：
  * 与存储在数据库中的数据关联的 `LiveData` 对象。
  * 与从网络访问的数据关联的 `LiveData` 对象。

* 您的 Activity 只需观察 `MediatorLiveData` 对象即可从这两个源接收更新。有关详细示例，请参阅[应用架构指南](https://developer.android.google.cn/topic/libraries/architecture/guide)的[附录：公开网络状态](https://developer.android.google.cn/topic/libraries/architecture/guide#addendum)部分。