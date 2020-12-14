[TOC]

## RxJava2线程切换原理

### 一、前言

#### 1.1 前提知识

* [RxJava2基本流程分析](设计思想解读开源框架库\RxJava响应式编程\RxJava2基本流程分析.md)

#### 1.2 实例

* 基于RxJava2
* 以下分析都基于此实例

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

### 二、observeOn()线程切换原理

#### 2.1 ObservableObserveOn#subscribeActual

* 从[RxJava2基本流程分析](设计思想解读开源框架库\RxJava响应式编程\RxJava2基本流程分析.md)我们可知，首先调用的就是ObservableObserveOn对象的subscribeActual方法

```java
@Override
protected void subscribeActual(Observer<? super T> observer) {
	if (scheduler instanceof TrampolineScheduler) {
    	source.subscribe(observer);
	} else {
        //分析1
    	Scheduler.Worker w = scheduler.createWorker();
		
        source.subscribe(new ObserveOnObserver<T>(observer, w, delayError, bufferSize));
	}
}
```

* 由于scheduler指定的是AndroidSchedulers.mainThread()，所以会进入else中的代码
* 其中source.subscribe就是向上传递的过程，这里就不细述了。可参考[RxJava2基本流程分析](设计思想解读开源框架库\RxJava响应式编程\RxJava2基本流程分析.md)
* 由[RxJava2基本流程分析](设计思想解读开源框架库\RxJava响应式编程\RxJava2基本流程分析.md)可知onXXX方法是通过Observer对象向下传递的，所以ObserveOnObserver对象会调用相应的onXXX方法，那么我们看看ObserveOnObserver对象的onNext方法来作为实例说明把

#### 2.2 ObservableObserveOn#ObserveOnObserver#onNext

```java
static final class ObserveOnObserver<T> extends BasicIntQueueDisposable<T>
implements Observer<T>, Runnable {
    ......
	@Override
	public void onNext(T t) {
    	if (done) {
        	return;
		}

        if (sourceMode != QueueDisposable.ASYNC) {
        	queue.offer(t);
		}
        //分析1
        schedule();
	}
}
```

* 分析1：其中调用了schedule方法，此方法就是用来切换线程的方法，是我们需要研究的重点

#### 2.3 ObservableObserveOn#ObserveOnObserver#schedule

```java
void schedule() {
	if (getAndIncrement() == 0) {
        //直接调用了 worker 的 schedule 方法，需要注意的是这里他把自己传了进去
    	worker.schedule(this);
	}
}
```

* 调用Scheduler.Worker对象worker的schedule方法

#### 2.4 Scheduler#Worker#schedule

```java
@NonNull
public Disposable schedule(@NonNull Runnable run) {
	return schedule(run, 0L, TimeUnit.NANOSECONDS);
}
```

```java
@NonNull
public abstract Disposable schedule(@NonNull Runnable run, long delay, @NonNull TimeUnit unit);
```

* 调用的schedule是抽象类，所以实际调用的是实现类的schedule方法
* 由于这里不说明RxAndroid，我们先假设使用的是IO，那么实现类就是IoScheduler，那么调用的就是IoScheduler的schedule方法
* 传递的参数是Runnable对象，由2.2可知ObserveOnObserver就实现了Runnable方法

#### 2.5 IoScheduler#schedule

```java
@NonNull
@Override
public Disposable schedule(@NonNull Runnable action, long delayTime, @NonNull TimeUnit unit) {
	if (tasks.isDisposed()) {
    	// don't schedule, we are unsubscribed
        return EmptyDisposable.INSTANCE;
	}

    return threadWorker.scheduleActual(action, delayTime, unit, tasks);
}
```

* 调用threadWorker.scheduleActual方法，其中threadWorker是ThreadWorker对象

#### 2.6 ThreadWorker#scheduleActual

* ThreadWorker没有重写此方法，是继承过来的，所以调用的其实是NewThreadWorker的scheduleActual方法

```java
static final class ThreadWorker extends NewThreadWorker {
```

#### 2.7 NewThreadWorker#scheduleActual

```java
@NonNull
public ScheduledRunnable scheduleActual(final Runnable run, long delayTime, @NonNull TimeUnit unit, @Nullable DisposableContainer parent) {
	Runnable decoratedRun = RxJavaPlugins.onSchedule(run);
	//创建runnable对象
    ScheduledRunnable sr = new ScheduledRunnable(decoratedRun, parent);

    if (parent != null) {
    	if (!parent.add(sr)) {
        	return sr;
		}
	}

    Future<?> f;
    try {
    	if (delayTime <= 0) {
        	f = executor.submit((Callable<Object>)sr);
		} else {
        	f = executor.schedule((Callable<Object>)sr, delayTime, unit);
		}
        sr.setFuture(f);
	} catch (RejectedExecutionException ex) {
    	if (parent != null) {
        	parent.remove(sr);
		}
        RxJavaPlugins.onError(ex);
	}

    return sr;
}
```

* executor其实就是一个线程池对象，调用线程池的方法，并把runnable对象传递进去，然后执行，会调用runnable对象的run方法
* 由于run就是传递进来的ObserveOnObserver对象，所以最终会调用ObserveOnObserver的run方法

#### 2.8 ObserveOnObserver#run

```java
@Override
public void run() {
	if (outputFused) {
    	drainFused();
	} else {
    	drainNormal();
	}
}
```

* 然后会调用drainFused/drainNormal方法，其中就会调用onNext方法，至此就结束了线程的切换过程

### 三、subscribeOn()的线程切换原理

#### 3.1 ObservableSubscribeOn#subscribeActual

* 和上面同理，会调用此方法

```java
@Override
public void subscribeActual(final Observer<? super T> observer) {
    //创建与之绑定的 SubscribeOnObserver
	final SubscribeOnObserver<T> parent = new SubscribeOnObserver<T>(observer);

    observer.onSubscribe(parent);
	//1. 创建 SubscribeTask 实际上就是个 Runnable
    //2. 然后调用 scheduler.scheduleDirect 方法
    parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent)));
}
```

* 创建 SubscribeTask 实际上就是个 Runnable
* 然后调用 scheduler.scheduleDirect 方法

#### 3.2 Scheduler#scheduleDirect 

```java
@NonNull
public Disposable scheduleDirect(@NonNull Runnable run) {
	return scheduleDirect(run, 0L, TimeUnit.NANOSECONDS);
}

@NonNull
public Disposable scheduleDirect(@NonNull Runnable run, long delay, @NonNull TimeUnit unit) {
	final Worker w = createWorker();

    final Runnable decoratedRun = RxJavaPlugins.onSchedule(run);
	//runnable
    DisposeTask task = new DisposeTask(decoratedRun, w);
	//分析1
    w.schedule(task, delay, unit);

    return task;
}
```

* 分析1：调用Scheduler#Worker的schedule方法，是一个抽象方法，假设是IO，那么调用的就是IoScheduler的schedule方法，这部分与上面类似，就不再分析了，同理最终会调用DisposeTask的run方法。

#### 3.3 Scheduler#DisposeTask#run

```java
@Override
public void run() {
	runner = Thread.currentThread();
    try {
    	decoratedRun.run();
	} finally {
    	dispose();
        runner = null;
	}
}
```

* 会调用decoratedRun.run();方法，其中decoratedRun就是传递进来的ObservableSubscribeOn，所以最终调用ObservableSubscribeOn#SubscribeTask的run方法

#### 3.4 ObservableSubscribeOn#SubscribeTask#run

```java
public void run() {
	source.subscribe(parent);
}
```

* 此时切换线程完成，然后往上调用subscribe方法，他又开始往上一层层的去订阅，所以 create(new ObservableOnSubscribe(){}）这个匿名实现接口运行 subscribe 的线程运行环境都被改变了，再去调用 onNext() 等方法线程环境也是被改变的

### 四、常见问题

#### 4.1 为什么 subscribeOn() 只有第一次切换有效

* 因为 RxJava 最终能影响 ObservableOnSubscribe 这个匿名实现接口的运行环境的只能是最后一次运行的 subscribeOn() ，又因为 RxJava 订阅的时候是从下往上订阅，所以从上往下第一个 subscribeOn() 就是最后运行的，这就造成了写多个 subscribeOn() 并没有什么乱用的现象。




