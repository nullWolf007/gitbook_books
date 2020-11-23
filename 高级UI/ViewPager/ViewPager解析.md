[TOC]

## ViewPager解析

#### 参考

* [Android 攒了一个月的面试题及解答](https://mp.weixin.qq.com/s?__biz=MzAxMTI4MTkwNQ==&mid=2650834187&idx=2&sn=706a9adca914b17068eed8229f23f12f&chksm=80b75195b7c0d8830112b514db00263577b17e5e6fc8fac6f09717d4c9ea67646ffdce97d763&mpshare=1&scene=1&srcid=1119d6zOeKGBPUfJj9BICS1t&sharer_sharetime=1606039636870&sharer_shareid=a33b8dedb16ee1f0a3a490b521e6b177&key=d7a8d1742847f01d393206b27be543280e6f28213e59f7582c2af6a6a9f5b280da40bbcba4fa65f3d1d3325145b5c24774306620f885e0893847ef18d84955639c1252638200f5fd33d85b2df651b8805de6bd78dfb49093f8b3ee203f458916fa33aa3ed3499f620b6e35e0464518ff804d6e85f40fd3be232390d820868cab&ascene=1&uin=OTU1NDk2NDE3&devicetype=Windows+10+x64&version=62090070&lang=zh_CN&exportkey=AznZIp3bCL7ty2SIVAvyrDU%3D&pass_ticket=9gX60QapKVHvQjJeMVpvHDRTaRZ%2B3Vk123XonsedIXSNd407tLSHq6qwvpi1UcBb&wx_header=0)





#### Fragment遇到viewpager遇到过什么问题吗

- 滑动的时候，调用setCurrentItem方法，要注意第二个参数smoothScroll。传false，就是直接跳到fragment，传true，就是平滑过去。一般主页切换页面都是用false。
- 禁止预加载的话，调用setOffscreenPageLimit(0)是无效的，因为方法里面会判断是否小于1。需要重写setUserVisibleHint方法，判断fragment是否可见。
- 不要使用getActivity()获取activity实例，容易造成空指针，因为如果fragment已经onDetach()了，那么就会报空指针。所以要在onAttach方法里面，就去获取activity的上下文。
- FragmentStatePagerAdapter对limit外的Fragment销毁，生命周期为onPause->onStop->onDestoryView->onDestory->onDetach, onAttach->onCreate->onCreateView->onStart->onResume。也就是说切换fragment的时候有可能会多次onCreateView，所以需要注意处理数据。
- 由于可能多次onCreateView，所以我们可以把view保存起来，如果为空再去初始化数据。见代码：

```java
@Override
public View onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState) {
    if (null == mFragmentView) {
        mFragmentView = inflater.inflate(getContentViewLayoutID(), null);
        ButterKnife.bind(this, mFragmentView);
        isDestory = false;
    	initViewsAndEvents();
	}
    return mFragmentView;
}
```