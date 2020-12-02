[TOC]

## RxJava2教程

### 参考文章

* [**这可能是最好的RxJava 2.x 教程（完结版）**](https://www.jianshu.com/p/0cd258eecf60)
* [**RxJava2 系列 -1：一篇的比较全面的 RxJava2 方法总结**](https://www.jianshu.com/p/823252f110b0)
* [RxJava官方wiki](https://github.com/ReactiveX/RxJava/wiki/)
* [**Android:随笔——RxJava的线程切换**](https://www.jianshu.com/p/d9da64774f7b)

## 一、前言

### 1.1 简介

* RxJava是一个在Java VM上使用可观测的序列来组成**异步**的、基于事件的程序的库。

### 1.2 RxJava和AsyncTask区别

* 在Android中，我们可以使用AsyncTask来完成异步任务操作，但是当任务的梳理比较多的时候，我们要为每个任务定义一个AsyncTask就变得非常繁琐。 RxJava能帮助我们在实现异步执行的前提下保持代码的清晰。
  它的原理就是创建一个`Observable`来完成异步任务，组合使用各种不同的链式操作，来实现各种复杂的操作，最终将任务的执行结果发射给`Observer`进行处理。

### 1.3 特点

- 简洁、简单、优雅
- 比Handler,AsyncTask灵活

## 二、解析

### 2.1 常用基础类

#### 2.1.1 Observable

* 被观察者：产生事件

* 多个流，无背压

#### 2.1.2 Flowable

* 被观察者：产生事件

* 多个流，响应式流和背压

#### 2.1.3 Event

* 事件：观察者，被观察者 沟通的载体

#### 2.1.4 Observer

* 观察者：接收事件

#### 2.1.5 Subscriber

* 观察者：接收事件

#### 2.1.6 Subscribe

* 订阅：连接 被观察者和观察者

#### 2.1.7 Disposable

* isDisposed()：该方法用来判断否停止了观察指定的流
* dispose()：该方法用来放弃观察指定的流
* 一般在Activity的onDestory中需要调用dispose方法，防止内存泄漏

#### 2.1.8 Single

* 只有一个元素或者错误的流

#### 2.1.9 Completable

* 没有任何元素，只有一个完成和错误信号的流

#### 2.1.10 Maybe

* 没有任何元素或者只有一个元素或者只有一个错误的流

### 2.2 背压(backpressure)

#### 2.2.1 含义

* 背压是流速控制的一种策略
* 背压是指在异步场景中，被观察者发送事件速度远快于观察者的处理速度的情况下，一种告诉上游的被观察者降低发送速度的策略。

### 2.3 两种观察者模式图解

![**RxJava两种观察者模式**](https://github.com/nullWolf007/images/raw/master/android/%E8%BF%9B%E9%98%B6/%E4%B8%89%E6%96%B9%E6%A1%86%E6%9E%B6/RxJava%E4%B8%A4%E7%A7%8D%E8%A7%82%E5%AF%9F%E8%80%85%E6%A8%A1%E5%BC%8F.png)

## 三、API说明

### 3.1 创建操作

#### 3.1.1 interval和intervalRange

* interval：表示每隔3秒发送一个整数，整数从0开始

  ```java
  Observable.interval(3, TimeUnit.SECONDS).subscribe(System.out::println);
  ```

* intervalRange：表示发送第一个整数停顿1秒，每隔3秒。发送一个整数。整数从2开始，发送10个数字，每次加1

  ```java
  Observable.intervalRange(2, 10, 1, 3, TimeUnit.SECONDS).subscribe(System.out::println);
  ```

#### 3.1.2 range和rangeLong

* range：整数从5开始，发送10个数字，每次加1。

  ```java
  Observable.range(5, 10).subscribe(i -> System.out.println("test" + i + ""));
  ```

* rangeLong：整数从5开始，发送10个数字，每次加1。区别是long类型的。

  ```java
  Observable.rangeLong(5, 10).subscribe(i -> System.out.println("test" + i + ""));
  ```

#### 3.1.3 create

* 创建Observable对象，需要传入发射器ObservableEmitter

* 通用写法

  ```java
  //输出 test1 test2
  Observable.create(new ObservableOnSubscribe<Integer>() {
      @Override
      public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
      	emitter.onNext(1);
          emitter.onNext(2);
          emitter.onComplete();
  	}
  }).subscribe(new Consumer<Integer>() {
  	// 每次接收到Observable的事件都会调用Consumer.accept（）
      @Override
      public void accept(Integer i) throws Exception {
      	System.out.println("test" + i);
  	}
  });
  
  //lambda写法
  //输出 test4 test5
  Observable.create(emitter -> {
  	emitter.onNext(4);
      emitter.onNext(5);
      emitter.onComplete();
  }).subscribe(i -> System.out.println("test" + i));
  
  //输出 test:onSubscribe test1 test2 test:onComplete
  Observable.create(emitter -> {
  	emitter.onNext(1);
      emitter.onNext(2);
      emitter.onComplete();
  }).subscribe(new Observer<Integer>() {
  	@Override
      public void onSubscribe(Disposable d) {
      	System.out.println("test:onSubscribe");
  	}
  
      @Override
      public void onNext(Integer integer) {
      	System.out.println("test" + integer);
  	}
  
      @Override
      public void onError(Throwable e) {
  	}
  
      @Override
      public void onComplete() {
      	System.out.println("test:onComplete");
  	}
  });
  ```


#### 3.1.4 defer

* `defer`直到有观察者订阅时才创建Observable，并且为每个观察者创建一个新的Observable。

* `public static <T> Observable<T> defer(Callable<? extends ObservableSource<? extends T>> supplier)`

* 示例代码

  ```java
  Observable<Long> observable = Observable.defer((Callable<ObservableSource<Long>>) () -> Observable.just(System.currentTimeMillis()));
  observable.subscribe(System.out::println);
  observable.subscribe(System.out::println);
  ```

* 说明：`defer`操作符会一直等待直到有观察者订阅它，然后它使用Observable工厂方法生成一个Observable。所以两个订阅输出的结果是不一致的

* 比较示例：这两个订阅输出的结果是一致的

  ```java
  Observable observable = Observable.just(System.currentTimeMillis());
  observable.subscribe(System.out::println);
  observable.subscribe(System.out::println);
  ```

#### 3.1.5 empty和never和error

* `public static<T> Observable empty()`：创建一个不发射任何数据但是正常终止的Observable；

* `public static<T> Observable never()`：创建一个不发射数据也不终止的Observable；

* `public static Observable error(Throwable exception)`：创建一个不发射数据以一个错误终止的Observable，它有几个重载版本，这里给出其中的一个。

* 示例代码

  ```java
  System.out.println("----empty()----");
  Observable.empty().subscribe(i -> System.out.println("next"), i -> System.out.println("error"), () -> System.out.println("complete"));
  
  System.out.println("----never()----");
  Observable.never().subscribe(i -> System.out.println("next"), i -> System.out.println("error"), () -> System.out.println("complete"));
  
  System.out.println("----error()----");
  Observable.error(new Exception()).subscribe(i -> System.out.println("next"), i -> System.out.println("error"), () -> System.out.println("complete"));
  /*
  * 输出
  * ----empty()----
  * complete
  * ----never()----
  * ----error()----
  * error
  */
  ```

#### 3.1.6 from

* `from`系列的方法用来从指定的数据源总获取一个Obserable。把数据源的数据依次发出

* `public static<T> Observable fromArray(T... items)`：从数组中获取；

* `public static<T> Observable fromCallable(Callable<? extends T> supplier)`：从Callable中获取；

* `public static<T> Observable fromFuture(Future<? extends T> future)`：从Future中获取，有多个重载版本，可以用来指定线程和超时等信息；

* `public static<T> Observable fromIterable(Iterable<? extends T> source)`：从Iterable中获取；

* `public static<T> Observable fromPublisher(Publisher<? extends T> publisher)`：从Publisher中获取。

* 示例代码

  ```java
  Observable.fromArray(new String[]{"1", "2"}).subscribe(System.out::println);
  //输出 1 2
  ```

#### 3.1.7 just

* just把传入的item依次发出

* `public static<T> Observable just(T item)`，它还有许多个重载的版本，区别在于接受的参数的个数不同，最少1个，最多10个。

#### 3.1.8 repeat

* 指定序列要发出多少次

* `public final Observable<T> repeat()`:无限次地发送指定的序列

* `public final Observable<T> repeat(long times)`:指定的序列重复发射指定的次数

* `public final Observable<T> repeatUntil(BooleanSupplier stop)`:在满足指定的要求的时候停止重复发送，否则会一直发送

* 示例代码

  ```java
  Observable.just(1,2,3).repeat(3).subscribe(i -> System.out.println(i));
  //输出三遍 1 2 3
  ```

#### 3.1.9 timer

* timer操作符创建一个在给定的时间段之后返回一个特殊值的Observable，它在延迟一段给定的时间后发射一个简单的数字0

* 实例代码：

  ```java
  Observable.timer(5, TimeUnit.SECONDS)
  	.subscribe(new Observer<Long>() {
      	@Override
          public void onSubscribe(Disposable d) {
          	System.out.println("onSubscribe");
  		}
  
          @Override
          public void onNext(Long aLong) {
          	System.out.println("onNext" + aLong);
  		}
  
          @Override
          public void onError(Throwable e) {
          	System.out.println("onError");
          }
  
          @Override
          public void onComplete() {
          	System.out.println("onComplete");
  		}
  });
  Thread.sleep(7000);
  //输出
  //onSubscribe
  //onNext0
  //onComplete
  ```

### 3.2 变换操作

#### 3.2.1 map和cast

**map**

* `map`操作符对原始Observable发射的每一项数据应用一个你选择的函数，然后返回一个发射这些结果的Observable。默认不在任何特定的调度器上执行。

* 示例代码

  ```java
  Observable.range(10, 3).map(String::valueOf).subscribe(System.out::println);
  //输出 10 11 12 
  ```

**cast**

* `cast`操作符将原始Observable发射的每一项数据都强制转换为一个指定的类型（多态），然后再发射数据，它是map的一个特殊版本。

* 示例代码

  ```java
  Observable.just(new Date()).cast(Object.class).subscribe(System.out::print);
  //把Date对象转换成Object对象
  ```

#### 3.2.2 flatMap和contactMap

**flatMap**

* `flatMap`将一个发送事件的上游Observable变换为多个发送事件的Observables，然后将它们发射的事件合并后放进一个单独的Observable里。需要注意的是, **flatMap并不保证事件的顺序**，也就是说转换之后的Observables的顺序不必与转换之前的序列的顺序一致。

* 示例代码

  ```java
  Observable.range(10, 3)
  	.flatMap((Function<Integer, ObservableSource<String>>) i -> Observable.just(String.valueOf(i)))
      .subscribe(System.out::print);
  ```

**contactMap**

* `contactMap`和`flatMap`类似，但是`contactMap`能够保证最终输出的顺序与上游发送的**顺序一致**。

* 示例代码

  ```java
  Observable.range(10, 3)
  	.concatMap((Function<Integer, ObservableSource<String>>) i -> Observable.just(String.valueOf(i)))
      .subscribe(System.out::println);
  ```

#### 3.2.3 flatMapIterable

* `flatMapIterable`可以用来将上流的任意一个元素转换成一个`Iterable`对象，然后我们可以对其进行消费。

* 示例代码

  ```java
  Observable.range(10, 3)
                  .flatMapIterable((Function<Integer, Iterable<?>>) integer -> Collections.singletonList(String.valueOf(integer))).subscribe(System.out::println);
  ```

#### 3.2.4 buffer

* `buffer`该方法用于将整个流进行分组。下面的代码就是以3个为一组。本意就相当于缓冲区，缓冲区满了就发出，当剩余的数据不够填满缓冲区也会发出

* 代码示例

  ```java
  Observable.range(10, 7).buffer(3)
                  .subscribe(integers -> System.out.println(Arrays.toString(integers.toArray())));
  ```

#### 3.2.5 groupBy

* `groupBy`用于分组元素，它可以被用来根据指定的条件将元素分成若干组。它将得到一个`Observable<GroupedObservable<T,M>>`类型的`Observable`。

* 如下所示代码；按照整数进行分组，分为四组，1 2 3 4，其中1到3由两个数据，4只有一个数据

* 实例代码

  ```java
  Observable<GroupedObservable<Integer, Integer>> observable = Observable.concat(Observable.range(1, 3), Observable.range(1, 4)).groupBy(integer -> integer);
  Observable.concat(observable).subscribe(integer -> System.out.println("groupBy : " + integer));
  ```

#### 3.2.6 scan

* `scan`操作符对原始Observable发射的第一项数据应用一个函数，然后将那个函数的结果作为自己的第一项数据发射。它将函数的结果同第二项数据一起填充给这个函数来产生它自己的第二项数据。它持续进行这个过程来产生剩余的数据序列。这个操作符在某些情况下被叫做accumulator。

* 通用公式：y(0)=x(0); y(i)=f(y(i-1), x(i)), i>0

* 示例代码

  ```java
  Observable.range(2, 5).scan((i1, i2) -> i1 * i2).subscribe(i -> System.out.print(i + " "));
  //输出 2 6 24 120 720
  ```

* 重载方法，可以指定初始值

#### 3.2.7 window

* `window`Window和Buffer类似，但不是发射来自原始Observable的数据包，它发射的是Observable，这些Observables中的每一个都发射原始Observable数据的一个子集，最后发射一个onCompleted通知。

* 通俗来说，它得到的是Observable。而不是数据。Buffer得到的是数据

* 示例代码

  ```java
  Observable.range(1, 10).window(3).subscribe(
                  observable -> observable.subscribe(integer -> System.out.println(observable.hashCode() + " : " + integer)));
  ```

### 3.3 过滤操作

#### 3.3.1 filter

* `filter`用来根据指定的规则对源进行过滤，比如下面的程序用来过滤整数1到10中所有大于5的数字

* 示例代码

  ```java
  Observable.range(1, 10).filter(i -> i > 5).subscribe(System.out::println);
  ```

#### 3.3.2 .elementAt和firstElement和lastElement

* `elementAt`用来获取源中指定位置的数据，它有几个重载方法，这里我们介绍一下最简单的一个方法的用法。

* `firstElement`顾名思义就是取第一个

* `lastElement`顾名思义就是取最后一个

* 示例代码

  ```java
  Observable.range(0, 5).elementAt(3).subscribe(System.out::print);
  //输出 3
  ```

#### 3.3.3 distinct和distinctUntilChanged

* `distinct`用来对源中的数据进行过滤，以下面的程序为例，这里会把重复的数字3过滤掉，只留下一个3

* 示例代码

  ```java
  Observable.just(1,2,3,3,2).distinct().subscribe(System.out::print);
  //输出 1 2 3 
  ```

* `distinctUntilChanged`值会当相邻的两个元素相同的时候才会将它们过滤掉

* 示例代码

  ```java
  Observable.just(1, 2, 3, 3, 2).distinctUntilChanged().subscribe(System.out::print);
  //输出 1 2 3 2
  ```

#### 3.3.4 skip和skipLast和skipUntil和skipWhile

* 这些重载的方法都可以设置时间定时

* `skip`方法用于过滤掉数据的前n项，比如下面的程序将会过滤掉前2项，因此输出结果是`345`

* 示例代码

  ```java
  Observable.range(1, 5).skip(2).subscribe(System.out::print);
  //输出 3 4 5 
  ```

* `take`和`skip`是对应关系，`take`用来表示选择数据源的前n项

* `skipLast`用来过滤掉最后几项

#### 3.3.5 take和takeLast和takeUntil和takeWhile

* 这些这些重载的方法都可以设置时间定时

* 这些就是和上面的skip是对应的关系

* `take`用来表示选择数据源的前n项，比如下面的程序将会选择前2项，因此输出结果是`12`

* 示例代码

  ```java
  Observable.range(1, 5).take(2).subscribe(System.out::print);
  //输出 1 2 
  ```

#### 3.3.6 ignoreElements

* `ignoreElements`用来过滤所有源Observable产生的结果，只会把Observable的onComplete和onError事件通知给订阅者

#### 3.3.7 throttleFirst和throttleLast和throttleLatest和throttleWithTimeout

* 这些方法用来对输出的数据进行限制，它们是通过时间的”窗口“来进行限制的，你可以理解成按照指定的参数对时间进行分片，然后根据各个方法的要求选择第一个、最后一个、最近的等进行发射。

* `throttleFirst`只会发射指定的Observable在指定的事件范围内发射出来的第一个数据

* `throttleLast`只会发射指定的Observable在指定的事件范围内发射出来的最后一个数

* `throttleLatest`用来发射距离指定的时间分片最近的那个数据

* `throttleWithTimeout`仅在过了一段指定的时间还没发射数据时才发射一个数据，如果在一个时间片达到之前，发射的数据之后又紧跟着发射了一个数据，那么这个时间片之内之前发射的数据会被丢掉，该方法底层是使用`debounce`方法实现的。如果数据发射的频率总是快过这里的`timeout`参数指定的时间，那么将不会再发射出数据来。

* 示例代码：在每个500毫秒，发送在这500毫秒内最后一位数字

  ```
   Observable.interval(80, TimeUnit.MILLISECONDS)
  	.throttleLast(500, TimeUnit.MILLISECONDS)
      .subscribe(i -> System.out.println("throttleLast" + i));
  Thread.sleep(7000);
  ```

#### 3.3.8  debounce

* `debounce`也是用来限制发射频率过快的，它仅在过了一段指定的时间还没发射数据时才发射一个数据

#### 3.3.9 sample

* 实际上`throttleLast`的实现中内部调用的就是`sample`。
* `sample`就是定期扫描源Observable产生的结果，在指定的间隔周期内拿到最后一个

### 3.4 组合操作

#### 3.4.1 startWith和startWithArray

* `startWith`方法可以用来在指定的数据源的之前插入几个数据，它的功能类似的方法有`startWithArray`

* 下面的程序会插入0，输出`0123`

* 示例代码

  ```java
  Observable.range(1, 3).startWith(0).subscribe(System.out::print);
  //输出 0 1 2 3
  ```

#### 3.4.2 merge和mergeArray

* `merge`可以让多个数据源的数据合并起来进行发射，当然它可能会让`merge`之后的数据交错发射。

* **存在无序性**

* 示例代码

  ```java
  Observable.merge(Observable.range(1,3), Observable.range(4,2)).subscribe(System.out::print);
  //输出 可能为 1 2 3 4 5 
  ```

#### 3.4.3 concat和concatArray和concatEager

* 该方法也是用来将多个Observable拼接起来，但是它会严格按照传入的Observable的顺序进行发射，一个Observable没有发射完毕之前不会发射另一个Observable里面的数据。

* **有序性**

* 示例代码

  ```java
  Observable.concat(Observable.range(1,3), Observable.range(4,2)).subscribe(System.out::print);
  //输出 一定是 1 2 3 4 5 
  ```

* `concatEager`方法，当一个观察者订阅了它的结果，那么就相当于订阅了它拼接的所有`ObservableSource`，并且会先缓存这些ObservableSource发射的数据，然后再按照顺序将它们发射出来

#### 3.4.4 zip和zipArray和zipIterable

* `zip`操作用来将多个数据项进行合并，可以通过一个函数指定这些数据项的合并规则。

* 下面的程序的输出结果是`6 14 24 36 50`，显然这里的合并的规则是相同索引的两个数据的乘积。不过仔细看下这里的输出结果，可以看出，如果一个数据项指定的位置没有对应的值的时候，它是不会参与这个变换过程的

* 示例代码

  ```java
  Observable.zip(Observable.range(1, 6), Observable.range(6, 5), (integer, integer2) -> integer * integer2)
                  .subscribe(i -> System.out.print(i + " "));
  //输出 6 14 24 36 50
  ```

#### 3.4.5 combineLastest

* 与`zip`操作类似，但是这个操作的输出结果与`zip`截然不同。

* 以下面的程序为例，它的输出结果是`36 42 48 54 60`：

* 示例代码

  ```java
  Observable.combineLatest(Observable.range(1, 6), Observable.range(6, 5), (integer, integer2) -> integer * integer2)
                  .subscribe(i -> System.out.print(i + " "));
  //输出 
  ```

* 说明：第一个数据项连续发射了6个数据的时候，第二个数据项一个都没有发射出来，因此没有任何输出；然后第二个数据项开始发射数据，当第二个数据项发射了6的时候，此时最新的数据组合是6和6，故得36；然后，第二个数据项发射了7，此时最新的数据组合是6和7，故得42，依次类推。

### 3.5 辅助操作

#### 3.5.1 delay

* `delay`方法用于在发射数据之前停顿指定的时间。

* 比如下面的程序会在真正地发射数据之前停顿1秒

* 示例代码

  ```java
  Observable.range(1, 5).delay(1000, TimeUnit.MILLISECONDS).subscribe(System.out::print);
  Thread.sleep(7000);
  //停顿一秒后 输出
  ```

#### 3.5.2 do系列

* `public final Observable<T> doAfterNext(Consumer<? super T> onAfterNext)`，会在`onNext`方法之后触发；

* `public final Observable<T> doAfterTerminate(Action onFinally)`，会在Observable终止之后触发；

* `public final Observable<T> doFinally(Action onFinally)`，当`onComplete`或者`onError`的时候触发；

* `public final Observable<T> doOnDispose(Action onDispose)`，当被dispose的时候触发；

* `public final Observable<T> doOnComplete(Action onComplete)`，当complete的时候触发；

* `public final Observable<T> doOnEach(final Observer<? super T> observer)`，当每个`onNext`调用的时候触发；

* `public final Observable<T> doOnError(Consumer<? super Throwable> onError)`，当调用`onError`的时候触发；

* `public final Observable<T> doOnLifecycle(final Consumer<? super Disposable> onSubscribe, final Action onDispose)`

* `public final Observable<T> doOnNext(Consumer<? super T> onNext)`，，会在`onNext`的时候触发；

* `public final Observable<T> doOnSubscribe(Consumer<? super Disposable> onSubscribe)`，会在订阅的时候触发；

* `public final Observable<T> doOnTerminate(final Action onTerminate)`，当终止之前触发。

* 示例代码

  ```java
  Observable.range(1, 5)
  	.doOnEach(integerNotification -> System.out.println("Each : " + integerNotification.getValue()))
      .doOnComplete(() -> System.out.println("complete"))
      .doFinally(() -> System.out.println("finally"))
      .doAfterNext(i -> System.out.println("after next : " + i))
      .doOnSubscribe(disposable -> System.out.println("subscribe"))
      .doOnTerminate(() -> System.out.println("terminal"))
      .subscribe(i -> System.out.println("subscribe : " + i));
  ```

#### 3.5.3 timeout

* 用来设置一个超时时间，如果指定的时间之内没有任何数据被发射出来，那么就会执行我们指定的数据项。

* 示例代码

  ```java
  Observable.interval(1000, 200, TimeUnit.MILLISECONDS)
  	.timeout(500, TimeUnit.MILLISECONDS, Observable.rangeLong(1, 5))
      .subscribe(System.out::print);
  Thread.sleep(2000);
  ```

### 3.6 错误处理操作符

#### 3.6.1 catch

* catch操作会拦截原始的Observable的`onError`通知，将它替换为其他数据项或者数据序列，让产生的Observable能够正常终止或者根本不终止

* **onErrorReturn**：这种操作会在onError触发的时候返回一个特殊的项替换错误，并调用观察者的onCompleted方法，而不会将错误传递给观察者

* **onErrorResumeNext**：会在onError触发的时候发射备用的数据项给观察者

* **onExceptionResumeNext**：如果onError触发的时候onError收到的Throwable不是Exception，它会将错误传递给观察者的onError方法，不会使用备用的Observable。

* 代码说明：下面是`onErrorReturn`和`onErrorResumeNext`的程序示例，这里第一段代码会在出现错误的时候输出`666`，而第二段会在出现错误的时候发射数字`12345`：

* 示例代码

  ```java
  Observable.create((ObservableOnSubscribe<Integer>) observableEmitter -> {
  	observableEmitter.onError(null);
      observableEmitter.onNext(0);
  }).onErrorReturn(throwable -> 666).subscribe(System.out::print);
  
  System.out.println();
  
  Observable.create((ObservableOnSubscribe<Integer>) observableEmitter -> {
  	observableEmitter.onError(null);
  	observableEmitter.onNext(0);
  }).onErrorResumeNext(Observable.range(1, 5)).subscribe(System.out::print);
  ```

### 3.6.2 retry

* `retry`使用了一种错误重试机制，它可以在出现错误的时候进行重试，我们可以通过参数指定重试机制的条件

* 代码说明：，这里我们设置了当出现错误的时候会进行2次重试，因此，第一次的时候出现错误会调用`onNext`，重试2次又会调用2次`onNext`，第二次重试的时候因为重试又出现了错误，因此此时会触发`onError`方法。也就是说，下面这段代码会触发`onNext`3次，触发`onError()`1次

* 示例代码

  ```java
  Observable.create(((ObservableOnSubscribe<Integer>) emitter -> {
  	emitter.onNext(0);
      emitter.onError(new Throwable("Error1"));
  })).retry(2).subscribe(i -> System.out.println("onNext : " + i), error -> System.out.print("onError : " + error));
  //输出
  //onNext : 0
  //onNext : 0
  //onNext : 0
  //onError : java.lang.Throwable: Error1
  ```

### 3.7 条件操作符和布尔操作符

#### 3.7.1 all和any

**all**

* `all`用来判断指定的数据项是否全部满足指定的要求，这里的“要求”可以使用一个函数来指定

**any**

* `any`用来判断指定的Observable是否存在满足指定要求的数据项

**示例代码**

```java
Observable.range(5, 5).all(i -> i > 5).subscribe(System.out::println); // false
Observable.range(5, 5).any(i -> i > 5).subscribe(System.out::println); // true
```

#### 3.7.2 contains和isEmpty

* 这两个方法分别用来判断数据项中是否包含我们指定的数据项，判断数据项是否为空

* 示例代码

  ```java
  Observable.range(5, 5).contains(4).subscribe(System.out::println); // false
  Observable.range(5, 5).isEmpty().subscribe(System.out::println); // false
  ```

#### 3.7.3 sequenceEqual

* `sequenceEqual`用来判断两个Observable发射出的序列是否是相等的

* 示例代码

  ```java
  Observable.sequenceEqual(Observable.range(1, 5), Observable.range(1, 5)).subscribe(System.out::println);//true
  ```

#### 3.7.4 amb

* `amb`作用的两个或多个Observable，但是只会发射最先发射数据的那个Observable的全部数据：

* 示例代码

  ```java
  Observable.amb(Arrays.asList(Observable.range(1, 5), Observable.range(6, 5))).subscribe(System.out::print);//12345
  ```

#### 3.7.5 defaultIfEmpty

* `defaultIfEmpty`用来当指定的序列为空的时候指定一个用于发射的值

* 示例代码

  ```java
  Observable.range(5, 0).defaultIfEmpty(6).subscribe(System.out::print);//6
  ```

### 3.8 转换操作符

#### 3.8.1 toList和toSortedList

* `toList`和`toSortedList`用于将序列转换成列表，后者相对于前者增加了排序的功能

* 示例代码：转换之后就是直接发送一个列表了 而不是依次发送数据

  ```java
  Observable.range(1, 5).toList().subscribe(System.out::println);//[1, 2, 3, 4, 5]
  Observable.range(1, 5).toSortedList(Comparator.comparingInt(o -> -o)).subscribe(System.out::println);//[5, 4, 3, 2, 1]
  ```

#### 3.8.2 toMap和toMultimap

* `toMap`用于将发射的数据转换成另一个类型的值，它的转换过程是针对每一个数据项的

* **无序性**

* 下面的代码为例，它会将原始的序列中的每个数字转换成对应的十六进制。但是，`toMap`转换的结果不一定是按照原始的序列的发射的顺序来的

* 示例代码

  ```java
  Observable.range(8, 10).toMap(Integer::toHexString).subscribe(System.out::print);
  //输出 {11=17, a=10, b=11, c=12, d=13, e=14, f=15, 8=8, 9=9, 10=16}
  ```

#### 3.8.3 toFlowable

* 方法用于将一个Observable转换成Flowable类型

#### 3.8.4 to

* `to`方法的限制更加得宽泛，你可以将指定的Observable转换成任意你想要的类型（如果你可以做到的话）

## 四、线程控制

### 4.1 RxJava五种线程

#### 4.1.1 Schedulers.io()

* `Schedulers.io()`：代表适用于io操作的调度器，增长或缩减来自适应的线程池，通常用于网络、读写文件等io密集型的操作。重点需要注意的是线程池是无限制的，大量的I/O调度操作将创建许多个线程并占用内存。

#### 4.1.2 Schedulers.computation()

* `Schedulers.computation()`：计算工作默认的调度器，代表CPU计算密集型的操作，与I/O操作无关。它也是许多RxJava方法，比如`buffer()`,`debounce()`,`delay()`,`interval()`,`sample()`,`skip()`，的默认调度器。

#### 4.1.3 Schedulers.newThread()

* `Schedulers.newThread()`：代表一个常规的新线程。

#### 4.1.4 Schedulers.immediate()

* `Schedulers.immediate()`：这个调度器允许你立即在当前线程执行你指定的工作。它是`timeout()`,`timeInterval()`以及`timestamp()`方法默认的调度器。

#### 4.1.5 Schedulers.trampoline()

* `Schedulers.trampoline()`：当我们想在当前线程执行一个任务时，并不是立即，我们可以用`trampoline()`将它入队。这个调度器将会处理它的队列并且按序运行队列中每一个任务。它是`repeat()`和`retry()`方法默认的调度器。

### 4.2 RxAndroid线程

#### 4.2.1 AndroidSchedulers.mainThread()

* `AndroidSchedulers.mainThread()`用来指代Android的主线程

### 4.3 subscribeOn和observeOn

**subscribeOn**

* `subscribeOn`用于指定Observable自身运行的线程，也就是被观察者的线程。影响的是被观察者所在的线程。
* **当使用多个`subscribeOn()` 的时候，只有第一个 `subscribeOn()` 起作用**
* 对于Android而言，一般耗时操作都发生在被观察者这，所以一般使用Schedulers.io()

**observeOn**

* observeOn() : 影响的是跟在后面的操作（指定观察者运行的线程）。
* **所以如果想要多次改变线程，可以多次使用 observeOn**
* 一般数据处理显示等都放在`observeOn`指定的线程 ,对于Android而言通常是AndroidSchedulers.mainThread()

**示例**

* 对于Android网络请求而言，被观察者一般在Schedulers.io()，观察者在AndroidSchedulers.mainThread()中

* 示例代码

  ```java
  Observable.range(1, 10)
      .subscribeOn(Schedulers.io())
  	.observeOn(AndroidSchedulers.mainThread())
      .subscribe(i -> Log.e("MainActivity", "onCreate: " + i));
  ```


## 五、实例核心代码

* 接口地址(金山词霸API)：http://fy.iciba.com/ajax.php?a=fy&f=auto&t=auto&w=hello,world
* 主要分给几部分：NetUrl存储路径的后半部分，Api存储接口方法，AppConfig存储了路径的前半部分(接口前半部分都是同一的)，NetUtils提供获取Retrofit的方法，GetDataResultBean为返回类

* 查看核心源代码请点击[RxJava2+Retrofit2+OkHttp实例核心代码](https://github.com/nullWolf007/ToolProject/tree/master/RxJava2%2BRetrofit2%2BOkHttp实例核心代码)







