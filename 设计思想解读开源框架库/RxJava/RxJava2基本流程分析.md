[TOC]

## RxJava2基本流程分析

### 一、前言

#### 1.1 实例

* 基于RxJava2

* 以下分析都基于此实例而言

```java
Observable.create(new ObservableOnSubscribe<String>() {
	@Override
    public void subscribe(ObservableEmitter<String> emitter) throws Exception {
    	emitter.onNext("1111");
		emitter.onNext("222");
        emitter.onComplete();
	}
}).map(new Function<String, Integer>() {
	@Override
    public Integer apply(String s) throws Exception {
    	return null;
	}
}).subscribeOn(Schedulers.io())
.observeOn(AndroidSchedulers.mainThread())
.subscribe(new Observer<Integer>() {
	@Override
    public void onSubscribe(Disposable d) {

    }

    @Override
    public void onNext(Integer integer) {

    }

    @Override
    public void onError(Throwable e) {

	}

    @Override
    public void onComplete() {

    }
});
```

### 二、源码分析

#### 2.1 Observable#create

```java
@CheckReturnValue
@NonNull
@SchedulerSupport(SchedulerSupport.NONE)
public static <T> Observable<T> create(ObservableOnSubscribe<T> source) {
	ObjectHelper.requireNonNull(source, "source is null");
    return RxJavaPlugins.onAssembly(new ObservableCreate<T>(source));
}
```

* 参数：ObservableOnSubscribe对象，这个对象对应的就是最根部的发射源
* 通过new ObservableCreate<T>(source)，封装了一个ObservableCreate对象，忽略onAssembly方法，其实返回的就是一个ObservableCreate对象
* 当然了ObservableCreate肯定是继承自Observable了

```java
public final class ObservableCreate<T> extends Observable<T> {
```

#### 2.2 Observable#map

```java
@CheckReturnValue
@SchedulerSupport(SchedulerSupport.NONE)
public final <R> Observable<R> map(Function<? super T, ? extends R> mapper) {
	ObjectHelper.requireNonNull(mapper, "mapper is null");
    return RxJavaPlugins.onAssembly(new ObservableMap<T, R>(this, mapper));
}
```

* 参数：传入了Function对象，主要是对其数据进行处理
* 通过new ObservableMap<T, R>(this, mapper)，封装了一个ObservableMap对象，忽略onAssembly方法，其实返回的就是一个ObservableMap对象
* 当然了ObservableMap肯定是继承自Observable了

#### 2.3 Observable#subscribeOn

```java
@CheckReturnValue
@SchedulerSupport(SchedulerSupport.CUSTOM)
public final Observable<T> subscribeOn(Scheduler scheduler) {
	ObjectHelper.requireNonNull(scheduler, "scheduler is null");
    return RxJavaPlugins.onAssembly(new ObservableSubscribeOn<T>(this, scheduler));
}
```

* 参数：传入了Scheduler对象，线程标志
* 通过new ObservableSubscribeOn()，封装了一个ObservableSubscribeOn对象，忽略onAssembly方法，其实返回的就是一个ObservableSubscribeOn对象
* 当然了ObservableSubscribeOn肯定是继承自Observable了

#### 2.4 Observable#observeOn

```java
@CheckReturnValue
@SchedulerSupport(SchedulerSupport.CUSTOM)
public final Observable<T> observeOn(Scheduler scheduler) {
	return observeOn(scheduler, false, bufferSize());
}

@CheckReturnValue
@SchedulerSupport(SchedulerSupport.CUSTOM)
public final Observable<T> observeOn(Scheduler scheduler, boolean delayError, int bufferSize) {
	ObjectHelper.requireNonNull(scheduler, "scheduler is null");
    ObjectHelper.verifyPositive(bufferSize, "bufferSize");
    return RxJavaPlugins.onAssembly(new ObservableObserveOn<T>(this, scheduler, delayError, bufferSize));
}
```

* 参数：传入了Scheduler对象，线程标志
* 通过new ObservableObserveOn()，封装了一个ObservableObserveOn对象，忽略onAssembly方法，其实返回的就是一个ObservableObserveOn对象
* 当然了ObservableObserveOn肯定是继承自Observable了

#### 2.5 Observable#subscribe

```java
@SchedulerSupport(SchedulerSupport.NONE)
@Override
public final void subscribe(Observer<? super T> observer) {
	ObjectHelper.requireNonNull(observer, "observer is null");
    try {
        //判空和hock机制
        //分析2
    	observer = RxJavaPlugins.onSubscribe(this, observer);

        ObjectHelper.requireNonNull(observer, "The RxJavaPlugins.onSubscribe hook returned a null Observer. Please change the handler provided to RxJavaPlugins.setOnObservableSubscribe for invalid null returns. Further reading: https://github.com/ReactiveX/RxJava/wiki/Plugins");
		//分析1
		subscribeActual(observer);
	} catch (NullPointerException e) { // NOPMD
    	throw e;
	} catch (Throwable e) {
    	Exceptions.throwIfFatal(e);
        // can't call onError because no way to know if a Disposable has been set or not
        // can't call onSubscribe because the call might have set a Subscription already
        RxJavaPlugins.onError(e);

        NullPointerException npe = new NullPointerException("Actually not, but can't throw other exceptions due to RS");
        npe.initCause(e);
        throw npe;
	}
}
```

* 参数：传入观察者，这就是用户自己处理的部分
* 分析1：调用了subscribeActual方法，这是关键，我们查看此方法的实现
* 分析2：判空和hock机制

#### 2.6 Observable#subscribeActual

```java
protected abstract void subscribeActual(Observer<? super T> observer);
```

* 从代码中我们可以看出这是一个抽象方法，所以它的实现肯定是在它的实现类里面
* 那么它的实现类是什么呢？从前面的部分我们知道ObservableCreate/ObservableMap等都继承自Observable，那么他们自然都是其实现类

#### 2.7 子类#subscribeActual

* 理论上应该是调用ObservableObserveOn的subscribeActual方法，但是这部分比较复杂，我们暂时先不考虑线程切换部分，先把这部分去除掉

* 那么应该调用的就是ObservableMap的subscribeActual方法， 所以先看看ObservableMap的subscribeActual方法
* ObservableMap#subscribeActual

```java
@Override
public void subscribeActual(Observer<? super U> t) {
	source.subscribe(new MapObserver<T, U>(t, function));
}
```

* 调用soucre的subscribe方法，并传入的是MapObserver对象
* 那么soucre是什么对象呢？

```java
public ObservableMap(ObservableSource<T> source, Function<? super T, ? extends U> function) {
	super(source);
    this.function = function;
}
```

* super(source)等于AbstractObservableWithUpstream的构造函数

```java
AbstractObservableWithUpstream(ObservableSource<T> source) {
	this.source = source;
}
```

* 从上面看出，source就是创建的时候的传入的对象，那么根据2.2可知，传入的是ObservableCreate对象，所以source.subscribe方法就是ObservableCreate的subscribe

#### 2.8 ObservableCreate#subscribe

* 由于ObservableCreate没有重写此方法，所以调用的是Observable的subscribe，其中参数为MapObserver对象，MapObserver继承自Observer对象，所以调用的是如下的subscribe方法
* 同时此时observer是MapObserver对象

```java
@SchedulerSupport(SchedulerSupport.NONE)
@Override
public final void subscribe(Observer<? super T> observer) {
	ObjectHelper.requireNonNull(observer, "observer is null");
    try {
        //判空和hock机制
        //分析2
    	observer = RxJavaPlugins.onSubscribe(this, observer);

        ObjectHelper.requireNonNull(observer, "The RxJavaPlugins.onSubscribe hook returned a null Observer. Please change the handler provided to RxJavaPlugins.setOnObservableSubscribe for invalid null returns. Further reading: https://github.com/ReactiveX/RxJava/wiki/Plugins");
		//分析1
		subscribeActual(observer);
	} catch (NullPointerException e) { // NOPMD
    	throw e;
	} catch (Throwable e) {
    	Exceptions.throwIfFatal(e);
        // can't call onError because no way to know if a Disposable has been set or not
        // can't call onSubscribe because the call might have set a Subscription already
        RxJavaPlugins.onError(e);

        NullPointerException npe = new NullPointerException("Actually not, but can't throw other exceptions due to RS");
        npe.initCause(e);
        throw npe;
	}
}
```

* 至此已经可以看到一个规律了，是逐步向上传递subscribe方法的，最终会调用ObservableCreate的subscribeActual方法

#### 2.9 ObservableCreate#subscribeActual

* 此时observer为MapObserver对象，传递过来的

```java
@Override
protected void subscribeActual(Observer<? super T> observer) {
    //分析1
	CreateEmitter<T> parent = new CreateEmitter<T>(observer);
    observer.onSubscribe(parent);

    try {
        //分析2
    	source.subscribe(parent);
	} catch (Throwable ex) {
    	Exceptions.throwIfFatal(ex);
        parent.onError(ex);
	}
}
```

* 分析1：会调用observer的onSubscribe发方法，其中observer为传递过来的MapObserver对象，所以会调用MapObserver对象的onSubscribe方法，并且参数parent为new CreateEmitter<T>(observer)
* 分析2：其中还会调用source.subscribe方法，此时source就是实例中ObservableOnSubscribe对象，那么ObservableOnSubscribe的subscribe就是我们自己实现的方法，那么就会调用Emitter的我们自己写的代码emitter.onNext

```java
Observable.create(new ObservableOnSubscribe<String>() {
	@Override
    public void subscribe(ObservableEmitter<String> emitter) throws Exception {
    	emitter.onNext("1111");
        emitter.onComplete();
	}
})
```

#### 2.10 MapObserver#onSubscribe

* 由于MapObserver没有重写此方法，是继承过来的，最终调用BasicFuseableObserver的onSubscribe方法

```java
@Override
public final void onSubscribe(Disposable d) {
	if (DisposableHelper.validate(this.upstream, d)) {

    	this.upstream = d;
        if (d instanceof QueueDisposable) {
        	this.qd = (QueueDisposable<T>)d;
		}

		if (beforeDownstream()) {
			downstream.onSubscribe(this);
            afterDownstream();
		}
	}
}
```

* 会调用downstream的onSubscribe方法，其中downstream就是Observer对象，所以会调用Observer的onSubscribe方法

#### 2.11 Observer#onSubscribe

```java
subscribe(new Observer<Integer>() {
	@Override
    public void onSubscribe(Disposable d) {

    }
}
```

* 逐层向下传递的，所以自定义的onNext没有执行之前，onSubscribe方法已经执行了，所以可以在onSubscribe方法中执行一些准备工作

#### 2.12 其他方法的传递

##### 2.12.1 

* 当执行onXXX方法后，这里以onNext举例，调用ObservableEmitter对象的onNext方法

```java
public void subscribe(ObservableEmitter<String> emitter) throws Exception {
                emitter.onNext("1111");
```

* ObservableEmitter是个接口类，它的实现类就是ObservableCreate中的CreateEmitter类
* 所以emitter#onNext等于ObservableCreate#CreateEmitter#onNext

##### 2.12.2 ObservableCreate#CreateEmitter#onNext

```java
static final class CreateEmitter<T>
extends AtomicReference<Disposable>
implements ObservableEmitter<T>, Disposable {
	@Override
    public void onNext(T t) {
    	if (t == null) {
        	onError(new NullPointerException("onNext called with null. Null values are generally not allowed in 2.x operators and sources."));
            return;
		}
        //分析1
        if (!isDisposed()) {
            //分析2
        	observer.onNext(t);
		}
	}
    ......
}
```

* 分析1：isDisposed用来判断是否取消了，没有取消的话调用observer.onNext(t)方法
* 分析2：上买那我们已经讲解了observer就是MapObserver对象，所以observer.onNext(t);等同于MapObserver#onNext对象

##### 2.12.3 MapObserver#onNext

* 通过类似的向下调用observer的onNext方法，最终会调用Observer的onNext的方法

##### 2.12.4 Observer#onNext

* 这样就完成了从起使到结束Observer的onXXX方法的传递

### 三、流程

* 虽然我们没有分析ObservableObserveOn和ObservableSubscribeOn部分，但是我们可以通过规律知道其流程

#### 3.1 实例流程总结

* Observable#create获得ObservableCreate对象
* ObservableCreate#map获得ObservableMap对象
* ObservableMap#subscribeOn获得ObservableSubscribeOn对象
* ObservableSubscribeOn#observeOn获得ObservableObserveOn对象
* ObservableObserveOn#subscribe方法调用subscribeActual方法
* subscribeActual是Observable的抽象方法，所以调用ObservableObserveOn的subscribeActual
* ObservableObserveOn调用source的subscribe方法，其source就是ObservableSubscribeOn，调用ObservableSubscribeOn的subscribe方法
* ObservableSubscribeOn#subscribe调用ObservableSubscribeOn的subscribeActual，进一步调用ObservableMap的subscribe方法
* 以此类推，最终自己的subscribe方法，就是用户自己写的subscribe方法
* 然后会通过其子的observer的onXXX方法逐步向下传递，最后自定义的Observer会调用onXXX方法
* 至此整个流程就结束了

#### 3.2 整体流程图

![RxJava2整体流程图](..\..\images\设计思想解读开源框架库\RxJava\RxJava2整体流程图.png)



