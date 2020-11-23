[TOC]

## RecyclerView解析

#### 参考

* [Android 攒了一个月的面试题及解答](https://mp.weixin.qq.com/s?__biz=MzAxMTI4MTkwNQ==&mid=2650834187&idx=2&sn=706a9adca914b17068eed8229f23f12f&chksm=80b75195b7c0d8830112b514db00263577b17e5e6fc8fac6f09717d4c9ea67646ffdce97d763&mpshare=1&scene=1&srcid=1119d6zOeKGBPUfJj9BICS1t&sharer_sharetime=1606039636870&sharer_shareid=a33b8dedb16ee1f0a3a490b521e6b177&key=d7a8d1742847f01d393206b27be543280e6f28213e59f7582c2af6a6a9f5b280da40bbcba4fa65f3d1d3325145b5c24774306620f885e0893847ef18d84955639c1252638200f5fd33d85b2df651b8805de6bd78dfb49093f8b3ee203f458916fa33aa3ed3499f620b6e35e0464518ff804d6e85f40fd3be232390d820868cab&ascene=1&uin=OTU1NDk2NDE3&devicetype=Windows+10+x64&version=62090070&lang=zh_CN&exportkey=AznZIp3bCL7ty2SIVAvyrDU%3D&pass_ticket=9gX60QapKVHvQjJeMVpvHDRTaRZ%2B3Vk123XonsedIXSNd407tLSHq6qwvpi1UcBb&wx_header=0)

#### Recycleview和listview区别

- Recycleview布局效果更多，增加了纵向，表格，瀑布流等效果
- Recycleview去掉了一些api，比如setEmptyview，onItemClickListener等等，给到用户更多的自定义可能
- Recycleview去掉了设置头部底部item的功能，专向通过viewholder的不同type实现
- Recycleview实现了一些局部刷新，比如notifyitemchanged
- Recycleview自带了一些布局变化的动画效果，也可以通过自定义ItemAnimator类实现自定义动画效果
- Recycleview缓存机制更全面，增加两级缓存，还支持自定义缓存逻辑





 

#### Recycleview有几级缓存，缓存过程？

Recycleview有四级缓存，分别是mAttachedScrap(屏幕内)，mCacheViews(屏幕外)，mViewCacheExtension(自定义缓存)，mRecyclerPool(缓存池)

- mAttachedScrap(屏幕内)，用于屏幕内itemview快速重用，不需要重新createView和bindView
- mCacheViews(屏幕外)，保存最近移出屏幕的ViewHolder，包含数据和position信息，复用时必须是相同位置的ViewHolder才能复用，应用场景在那些需要来回滑动的列表中，当往回滑动时，能直接复用ViewHolder数据，不需要重新bindView。
- mViewCacheExtension(自定义缓存)，不直接使用，需要用户自定义实现，默认不实现。
- mRecyclerPool(缓存池)，当cacheView满了后或者adapter被更换，将cacheView中移出的ViewHolder放到Pool中，放之前会把ViewHolder数据清除掉，所以复用时需要重新bindView。

四级缓存按照顺序需要依次读取。所以完整缓存流程是：

1) 保存缓存流程：

- 插入或是删除itemView时，先把屏幕内的ViewHolder保存至AttachedScrap中
- 滑动屏幕的时候，先消失的itemview会保存到CacheView，CacheView大小默认是2，超过数量的话按照先入先出原则，移出头部的itemview保存到RecyclerPool缓存池（如果有自定义缓存就会保存到自定义缓存里），RecyclerPool缓存池会按照itemview的itemtype进行保存，每个itemTyep缓存个数为5个，超过就会被回收。

2) 获取缓存流程：

- AttachedScrap中获取，通过pos匹配holder——>获取失败，从CacheView中获取，也是通过pos获取holder缓存 ——>获取失败，从自定义缓存中获取缓存——>获取失败，从mRecyclerPool中获取 ——>获取失败，重新创建viewholder——createViewHolder并bindview。

需要注意的是，如果从缓存池找到缓存，还需要重新bindview。







#### RecyclerView的优化

- bindViewHolder方法是在UI线程进行的，此方法不能耗时操作，不然将会影响滑动流畅性。比如进行日期的格式化。
- 对于新增或删除的时候，可以使用diffutil进行局部刷新，少用全局刷新
- 对于itemVIew进行布局优化，比如少嵌套等。
- 25.1.0 (>=21)及以上使用Prefetch 功能，也就是预取功能，嵌套时且使用的是LinearLayoutManager，子RecyclerView可通过setInitialPrefatchItemCount设置预取个数
- 加大RecyclerView缓存，比如cacheview大小默认为2，可以设置大点，用空间来换取时间，提高流畅度
- 如果高度固定，可以设置setHasFixedSize(true)来避免requestLayout浪费资源，否则每次更新数据都会重新测量高度。

```java
void onItemsInsertedOrRemoved() {
   if (hasFixedSize) layoutChildren();
   else requestLayout();
}
```

- 如果多个RecycledView 的 Adapter 是一样的，比如嵌套的 RecyclerView 中存在一样的 Adapter，可以通过设置 RecyclerView.setRecycledViewPool(pool);来共用一个 RecycledViewPool。这样就减少了创建VIewholder的开销。
- 在RecyclerView的元素比较高，一屏只能显示一个元素的时候，第一次滑动到第二个元素会卡顿。这种情况就可以通过设置额外的缓存空间，重写getExtraLayoutSpace方法即可。

```java
new LinearLayoutManager(this) {
    @Override
    protected int getExtraLayoutSpace(RecyclerView.State state) {
        return size;
    }
};
```

- 设置RecyclerView.addOnScrollListener();来在滑动过程中停止加载的操作。
- 减少对象的创建，比如设置监听事件，可以全局创建一个，所有view公用一个listener，并且放到CreateView里面去创建监听，因为CreateView调用要少于bindview。这样就减少了对象创建所造成的消耗
- 用notifyDataSetChange时，适配器不知道整个数据集中的那些内容以及存在，再重新匹配ViewHolder时会发生闪烁。设置adapter.setHasStableIds(true)，并重写getItemId()来给每个Item一个唯一的ID，也就是唯一标识，就使itemview的焦点固定，解决了闪烁问题。