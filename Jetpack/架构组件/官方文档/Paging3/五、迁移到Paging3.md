[TOC]

# 迁移到 Paging 3

* Paging 3 与旧版 Paging 库存在很大区别。此版本提供了增强型功能，还解决了使用 Paging 2 时常见的问题。如果您的应用已在使用旧版 Paging 库，请阅读本页面，详细了解如何迁移到 Paging 3。

* 如果 Paging 3 是您在应用中使用的第一个 Paging 库版本，请参阅[加载并显示分页数据](https://developer.android.google.cn/topic/libraries/architecture/paging/v3-paged-data)，了解基本用法信息。

## 一、迁移到 Paging 3 的优势

* Paging 3 包含旧版库中没有的下列功能：
  * 为 Kotlin 协程和流程提供一流的支持。
  * 支持使用 RxJava `Single` 或 Guava `ListenableFuture` 基元进行异步加载。
  * 针对自适应界面设计的内置加载状态和错误信号，包括重试和刷新功能。
  * 改进了代码库层，包括取消支持和简化的数据源界面。
  * 改进了演示层、列表分隔符、自定义页面转换和加载状态页眉和页脚。

## 二、将应用迁移到 Paging 3

* 为了完全迁移到 Paging 3，您必须从 Paging 2 迁移所有三个主要组件：
  * `DataSource` 类
  * `PagedList`
  * `PagedListAdapter`

* 不过，某些 Paging 3 组件向后兼容旧版 Paging。具体来说，Paging 3 中的 [`PagingSource`](https://developer.android.google.cn/reference/kotlin/androidx/paging/PagingSource) API 可以是旧版中 [`LivePagedListBuilder`](https://developer.android.google.cn/reference/kotlin/androidx/paging/LivePagedListBuilder) 和 [`RxPagedListBuilder`](https://developer.android.google.cn/reference/kotlin/androidx/paging/RxPagedListBuilder) 的数据源。同样，`Pager` API 可将旧版 [`DataSource`](https://developer.android.google.cn/reference/kotlin/androidx/paging/DataSource) 对象用于 `toPagingSourceFactory()` 方法。这意味着，您有以下迁移选项：
  * 您可以将 `DataSource` 迁移到 `PagingSource`，但让 Paging 实现的其余部分保持不变。
  * 您可以迁移 `PagedList` 和 `PagedListAdapter`，但仍使用旧版 `DataSource` API。
  * 您可以迁移整个 Paging 实现以将应用完全迁移到 Paging 3。

* 本页中的各个部分介绍了如何迁移应用每个层上的 Paging 组件。

### 2.1 DataSource 类

* 本部分介绍迁移旧版 Paging 实现以使用 `PagingSource` 所需的所有更改。

* Paging 2 中的 `PageKeyedDataSource`、`PositionalDataSource` 和 `ItemKeyedDataSource` 都合并到了 Paging 3 中的 `PagingSource` API 中。所有旧版 API 类中的加载方法都合并到了 `PagingSource` 中的单个 `load()` 方法中。这样可以减少代码重复，因为在旧版 API 类的实现中，各种加载方法之间的很多逻辑通常是相同的。

* 在 Paging 3 中，所有加载方法参数被一个 `LoadParams` 密封类替代，该类中包含了每个加载类型所对应的子类。如果需要区分 `load()` 方法中的加载类型，请检查传入了 `LoadParams` 的哪个子类：`LoadParams.Refresh`、`LoadParams.Prepend` 还是 `LoadParams.Append`。

* 如需详细了解如何实现 `PagingSource`，请参阅[定义数据源](https://developer.android.google.cn/topic/libraries/architecture/paging/v3-paged-data#data-source)。

* **注意**：如果您当前的 Paging 实现包含自定义错误处理，请考虑改用 `PagingSource` 中内置的错误处理支持。如需了解详情，请参阅[处理错误](https://developer.android.google.cn/topic/libraries/architecture/paging/v3-paged-data#handle-errors)。

#### 2.1.1 刷新键

* `PagingSource` 实现必须定义如何从已加载分页数据的中间恢复刷新。为此，您可以实现 [`getRefreshKey()`](https://developer.android.google.cn/reference/kotlin/androidx/paging/PagingSource#getrefreshkey)，以使用 `state.anchorPosition` 作为最近访问的索引来映射正确的初始键。

```kotlin
// Replaces ItemKeyedDataSource.
override fun getRefreshKey(state: PagingState): String? {
  return state.anchorPosition?.let { anchorPosition ->
    state.getClosestItemToPosition(anchorPosition)?.id
  }
}

// Replacing PositionalDataSource.
override fun getRefreshKey(state: PagingState): Int? {
  return state.anchorPosition
}
```

#### 2.1.2 列表转换

* 在旧版 Paging 库中，分页数据的转换依赖于以下方法：
  * `DataSource.map()`
  * `DataSource.mapByPage()`
  * `DataSource.Factory.map()`
  * `DataSource.Factory.mapByPage()`

* 在 Paging 3 中，所有转换都作为运算符应用于 `PagingData`。如果使用上述列表中的某个方法转换分页列表，则在使用新的 `PagingSource` 构建 `Pager` 时，必须将转换逻辑从 `DataSource` 变为 `PagingData`。

* 如需详细了解如何使用 Paging 3 转换分页数据，请参阅[转换数据流](https://developer.android.google.cn/topic/libraries/architecture/paging/v3-transform)。

### 2.2 PagedList

* 本部分介绍迁移旧版 Paging 实现以使用 Paging 3 中的 `Pager` 和 `PagingData` 所需的所有更改。

#### 2.2.1 PagedListBuilder 类

* `PagingData` 替换了 Paging 2 中现有的 `PagedList`。为了迁移到 `PagingData`，您必须完成以下更新：
  * 将分页配置从 `PagedList.Config` 变为 `PagingConfig`。
  * 将 `LivePagedListBuilder` 和 `RxPagedListBuilder` 合并成单个 `Pager` 类。
  * `Pager` 使用其 `.flow` 属性公开可观察的 `Flow<PagingData>`。RxJava 和 LiveData 变体也可以作为扩展属性使用，这些属性可通过静态方法从 Java 调用，它们分别从 `paging-rxjava*` 和 `paging-runtime` 模块提供。

```kotlin
val flow = Pager(
  // Configure how data is loaded by passing additional properties to
  // PagingConfig, such as prefetchDistance.
  PagingConfig(pageSize = 20)
) {
  ExamplePagingSource(backend, query)
}.flow
  .cachedIn(viewModelScope)
```

* 如需详细了解如何使用 Paging 3 设置 `PagingData` 对象的响应式流，请参阅[设置 PagingData 流](https://developer.android.google.cn/topic/libraries/architecture/paging/v3-paged-data#pagingdata-stream)。

#### 2.2.2 分层源的 BoundaryCallback

* 在 Paging 3 中，[`RemoteMediator`](https://developer.android.google.cn/reference/kotlin/androidx/paging/RemoteMediator) 取代 `PagedList.BoundaryCallback` 作为处理从网络和数据库分页的处理程序。

* 如需详细了解如何在 Paging 3 中使用 `RemoteMediator` 从网络和数据库分页，请参阅 [Android Paging Codelab](https://codelabs.developers.google.com/codelabs/android-paging)。

### 2.3 PagedListAdapter

* 本部分介绍迁移旧版 Paging 实现以使用 Paging 3 中的 `PagingDataAdapter` 或 `AsyncPagingDataDiffer` 类所需的所有更改。

* Paging 3 提供 `PagingDataAdapter` 来处理新的 `PagingData` 响应式流。否则，`PagedListAdapter` 和 `PagingDataAdapter` 将具有相同的接口。如需从 `PagedListAdapter` 迁移到 `PagingDataAdapter`，请更改 `PagedListAdapter` 的实现，以扩展 `PagingDataAdapter`。

* 如需详细了解 `PagingDataAdapter`，请参阅[定义 RecyclerView 适配器](https://developer.android.google.cn/topic/libraries/architecture/paging/v3-paged-data#recyclerview-adapter)。

#### 2.3.1 AsyncPagedListDiffer

* 如果您目前将自定义 `RecyclerView.Adapter` 实现与 `AsyncPagedListDiffer` 搭配使用，请迁移您的实现以使用 Paging 3 中提供的 `AsyncPagingDataDiffer`：

```kotlin
AsyncPagingDataDiffer(diffCallback, listUpdateCallback)
```