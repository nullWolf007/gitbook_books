[TOC]

## AsyncTask

#### 参考

* [AsyncTask 面试解析](https://juejin.im/post/6844903842841100295#heading-8)
* [理解 AsyncTask 原理](https://juejin.im/post/6844903784968093704#heading-3)

#### 注意点

* 基于Android8.0

### 一、基础知识

#### 1.1 概述

* AsyncTask是一种轻量级异步任务类，封装了Thread和Handle。可以方便的执行后台任务，并且能够在主线程中对返回结果进行更新UI

* AsyncTask是一个抽象类

  ```java
  public abstract class AsyncTask<Params, Progress, Result> {
  ```

### 二、主要方法

#### 2.1 参数

```java
public abstract class AsyncTask<Params, Progress, Result> {
```

* 这三个参数都是泛型。

##### 2.1.1 Params

* 输入参数
* 任务开始执行时客户端发送开始参数
* execute()中发送，在doInBackground()中调用。

##### 2.1.2 Progress

* 过程参数
*  任务后台执行过程中服务端发布的当前执行进度
* 在doInBackground()中产生并通过publishProgess()发送，在onProgressUpdate()调用。

##### 2.1.3 Result

* 结果参数
* 任务执行完成后服务端发送的执行结果
* 在doInBackground()中产生并在onPostExecute()中调用。

#### 2.1 onPreExecute

##### 2.1.1 概述

```java
@MainThread
protected void onPreExecute() {
}
```

* 在主线程中运行，主要是在后台线程开始执行任务之前进行某些UI的初始化
* 可以重写此方法

#### 2.2 doInBackground

##### 2.2.2 概述

```java
@WorkerThread
protected abstract Result doInBackground(Params... params);
```

* 在后台线程中运行，主要是接收客户端发送过来的参数，在后台执行耗时任务并发布执行进度和执行结果，例如文件下载任务，是在使用过程中必须实现的接口
* 是一个抽象方法，必须实现此方法
* params参数表示异步任务的输入参数。需要返回计算结果给onPostExecute()方法。
* 这个方法通过publishProgress方法来更新任务的进度，publishProgress方法会调用onProgressUpdate方法

#### 2.3 publishProgress

##### 2.3.1 概述

```java
@WorkerThread
protected final void publishProgress(Progress... values) {
	if (!isCancelled()) {
    	getHandler().obtainMessage(MESSAGE_POST_PROGRESS,
        		new AsyncTaskResult<Progress>(this, values)).sendToTarget();
	}
}
```

* 在后台线程中运行，主要是发布任务的当前执行进度，以方便在主线程中显示，不需要重新实现直接调用
* 不可重写

#### 2.4 onProgressUpdate

##### 2.4.1 概述

```java
@SuppressWarnings({"UnusedDeclaration"})
@MainThread
protected void onProgressUpdate(Progress... values) {
}
```

*  在主线程中运行，主要是更新当前的执行进度，例如更新进度条进度，可选择实现
* 可重写

#### 2.5 onPostExecute

##### 2.5.1 概述

```java
@SuppressWarnings({"UnusedDeclaration"})
@MainThread
protected void onPostExecute(Result result) {
}
```

* 在主线程中运行，主要是接收`doInBackground`返回的执行结果并在主线程中显示，例如显示下载文件大小

### 三、实例

* R.layout.activity_main

```xml
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    tools:context=".MainActivity">

    <TextView
        android:id="@+id/main_tv_download"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:gravity="center"
        android:text="模拟下载"
        android:textSize="20sp" />

    <ProgressBar
        android:id="@+id/main_progress_bar"
        style="@android:style/Widget.ProgressBar.Horizontal"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_gravity="center"
        android:layout_marginTop="10dp"
        android:max="100"
        android:visibility="gone" />
</FrameLayout>
```

* MainActivity.java

```java
public class MainActivity extends AppCompatActivity {

    private ProgressBar progressBar;
    private DownLoadAsyncTask downLoadAsyncTask;
    private String[] testStrs = new String[10];

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        progressBar = findViewById(R.id.main_progress_bar);
        downLoadAsyncTask = new DownLoadAsyncTask(progressBar);

        for (int i = 0; i < 10; i++) {
            testStrs[i] = "test" + i;
        }

        findViewById(R.id.main_tv_download).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                downLoadAsyncTask.execute(testStrs);
            }
        });
    }

    static class DownLoadAsyncTask extends AsyncTask<String, Integer, Integer> {
        private WeakReference<ProgressBar> weakReference;

        public DownLoadAsyncTask(ProgressBar progressBar) {
            weakReference = new WeakReference<>(progressBar);
        }

        @Override
        protected void onPreExecute() {
            ProgressBar progressBar = weakReference.get();
            progressBar.setVisibility(View.VISIBLE);
            progressBar.setMax(100);
            progressBar.setProgress(0);
        }

        @Override
        protected Integer doInBackground(String... strings) {
            int count = strings.length;
            for (int i = 1; i <= count; i++) {
                try {
                    // 休眠2秒模拟下载过程
                    Thread.sleep(2 * 1000);
                    // 发布进度更新
                    publishProgress((100 * i) / count);
                } catch (InterruptedException ie) {
                    ie.printStackTrace();
                }
            }
            return null;
        }

        @Override
        protected void onProgressUpdate(Integer... values) {
            ProgressBar progressBar = weakReference.get();
            progressBar.setProgress(values[0]);
        }

        @Override
        protected void onPostExecute(Integer integer) {
            ProgressBar progressBar = weakReference.get();
            progressBar.setVisibility(View.GONE);
        }
    }
}
```

* 其中采取了静态内部类+弱引用防止内存泄漏
* 其他的就是AsyncTask的基本使用，这里就不说明了

### 四、源码分析-主要成员变量

#### 4.1 线程池部分

```java
private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();
// We want at least 2 threads and at most 4 threads in the core pool,
// preferring to have 1 less than the CPU count to avoid saturating
// the CPU with background work
private static final int CORE_POOL_SIZE = Math.max(2, Math.min(CPU_COUNT - 1, 4));
private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
private static final int KEEP_ALIVE_SECONDS = 30;

private static final ThreadFactory sThreadFactory = new ThreadFactory() {
	private final AtomicInteger mCount = new AtomicInteger(1);

    public Thread newThread(Runnable r) {
    	return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
	}
};

private static final BlockingQueue<Runnable> sPoolWorkQueue =
	new LinkedBlockingQueue<Runnable>(128);

/**
* An {@link Executor} that can be used to execute tasks in parallel.
*/
public static final Executor THREAD_POOL_EXECUTOR;

static {
	ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
    		CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
            sPoolWorkQueue, sThreadFactory);
	threadPoolExecutor.allowCoreThreadTimeOut(true);
    THREAD_POOL_EXECUTOR = threadPoolExecutor;
}
```

#### 4.2 Status

```java
public enum Status {
	/**
    * Indicates that the task has not been executed yet.
    */
    PENDING,
    /**
    * Indicates that the task is running.
    */
    RUNNING,
    /**
    * Indicates that {@link AsyncTask#onPostExecute} has finished.
    */
    FINISHED,
}
```

* PENDING：等待中
* RUNNING：运行中
* FINISHED：已完成

### 五、源码分析-构造方法

#### 5.1 构造方法

* 我么首先看一下构造方法做了哪些操作
* 这里有几个重载的构造方法，但是最终都会调用如下这个构造方法

```java
public AsyncTask(@Nullable Looper callbackLooper) {
    //判断初始化为主线程Handler或者new一个Handler
	mHandler = callbackLooper == null || callbackLooper == Looper.getMainLooper()
    	? getMainHandler()
        : new Handler(callbackLooper);

    //创建WorkerRunnable实例 并实现了call方法
	mWorker = new WorkerRunnable<Params, Result>() {
    	public Result call() throws Exception {
        	mTaskInvoked.set(true);
            Result result = null;
            try {
            	Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                //noinspection unchecked
                result = doInBackground(mParams);
                Binder.flushPendingCommands();
			} catch (Throwable tr) {
            	mCancelled.set(true);
                throw tr;
			} finally {
            	postResult(result);
			}
            return result;
		}
	};

    //把Callable的实例 传递给FutureTask实例
    mFuture = new FutureTask<Result>(mWorker) {
    	@Override
        protected void done() {
        	try {
            	postResultIfNotInvoked(get());
			}......
		}
	};
}
```

* 创建WorkerRunnable实例 mWorker，并实现了call方法，其中call方法中调用了doInBackground
* 创建了FutureTask实例，并把mWorker作为参数传递了进去
* 这里实际上就是FutureTask+Callable的用法，详情可查看[Callable,Future和FutureTask](必备Java知识/并发编程/线程池/Callable,Future和FutureTask.md)

#### 5.2 getMainHandler

```java
private static Handler getMainHandler() {
	synchronized (AsyncTask.class) {
    	if (sHandler == null) {
        	sHandler = new InternalHandler(Looper.getMainLooper());
		}
        return sHandler;
	}
}
```

* 如果callbackLooper为null，或者是主线程Looper的话，则调用getMainHandler方法，返回的是一个InternalHandler实例sHandler

#### 5.3 WorkerRunnable

```java
private static abstract class WorkerRunnable<Params, Result> implements Callable<Result> {
	Params[] mParams;
}
```

* WorkerRunnable是一个抽象类，implements了Callable，关于Callable详情可查看[Callable,Future和FutureTask](必备Java知识/并发编程/线程池/Callable,Future和FutureTask.md)

### 六、源码分析-execute 

#### 6.1 execute

* 此方法是开启异步任务的的方法，故从这里开始分析

```java
@MainThread
public final AsyncTask<Params, Progress, Result> execute(Params... params) {
	return executeOnExecutor(sDefaultExecutor, params);
}
```

* 调用了executeOnExecutor方法

#### 6.2 executeOnExecutor

```java
@MainThread
public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
		Params... params) {
    //判断当前的状态 如果不是等待中的话 就抛出异常
	if (mStatus != Status.PENDING) {
    	switch (mStatus) {
        	case RUNNING:
            	throw new IllegalStateException("Cannot execute task:"
                	+ " the task is already running.");
			case FINISHED:
            	throw new IllegalStateException("Cannot execute task:"
                	+ " the task has already been executed "
                    + "(a task can be executed only once)");
		}
	}
	//修改状态为 运行中
    mStatus = Status.RUNNING;
	
    //执行onPreExecute方法
    onPreExecute();
    //把参数赋值到WorkRunnable的实例中
    mWorker.mParams = params;
    //调用Executor的实例 去执行execute方法
    exec.execute(mFuture);

    return this;
}
```

* 修改了Status的状态值
* 执行onPreExecute方法
* exec是Executor的实例，是传递过来的，本质是sDefaultExecutor，然后执行了其execute方法

#### 6.3 sDefaultExecutor

##### 3.2.3 sDefaultExecutor

```java
private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
```

* 是SERIAL_EXECUTOR赋值过来的

```java
public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
```

* 创建了SerialExecutor实例，所以exec.execute(mFuture);实际上调用的是SerialExecutor的execute方法

#### 6.4 SerialExecutor

```java
private static class SerialExecutor implements Executor {
	final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
    Runnable mActive;

    public synchronized void execute(final Runnable r) {
    	mTasks.offer(new Runnable() {
        	public void run() {
            	try {
                	r.run();
				} finally {
                	scheduleNext();
				}
			}
		});
		if (mActive == null) {
			scheduleNext();
		}
	}

    protected synchronized void scheduleNext() {
    	if ((mActive = mTasks.poll()) != null) {
        	THREAD_POOL_EXECUTOR.execute(mActive);
		}
	}
}
```

* SerialExecutor的execute方法中，我们可以看到使用创建了Runnable的匿名内部类，并加入到了ArrayDeque的数据结构中。同时使用了synchronized，保证其线程安全，同时最终会调用finally里的方法scheduleNext()
* scheduleNext方法又会调用THREAD_POOL_EXECUTOR的execute方法
* 所以客户端的execute方法最终会调用THREAD_POOL_EXECUTOR的execute方法

#### 6.5 THREAD_POOL_EXECUTOR

```java
private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();
// We want at least 2 threads and at most 4 threads in the core pool,
// preferring to have 1 less than the CPU count to avoid saturating
// the CPU with background work
private static final int CORE_POOL_SIZE = Math.max(2, Math.min(CPU_COUNT - 1, 4));
private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
private static final int KEEP_ALIVE_SECONDS = 30;

private static final ThreadFactory sThreadFactory = new ThreadFactory() {
	private final AtomicInteger mCount = new AtomicInteger(1);

    public Thread newThread(Runnable r) {
    	return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
	}
};

private static final BlockingQueue<Runnable> sPoolWorkQueue =
	new LinkedBlockingQueue<Runnable>(128);

/**
* An {@link Executor} that can be used to execute tasks in parallel.
*/
public static final Executor THREAD_POOL_EXECUTOR;

static {
	ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
    		CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
            sPoolWorkQueue, sThreadFactory);
	threadPoolExecutor.allowCoreThreadTimeOut(true);
    THREAD_POOL_EXECUTOR = threadPoolExecutor;
}
```

* THREAD_POOL_EXECUTOR是一个final不可修改的，在static静态代码块里赋值的，把threadPoolExecutor赋值给了THREAD_POOL_EXECUTOR，所以本质上是threadPoolExecutor，threadPoolExecutor是一个ThreadPoolExecutor实例
* ThreadPoolExecutor详细知识请查看[线程池基本解析](必备Java知识/并发编程/线程池/线程池基本解析.md)
* 上面的都是ThreadPoolExecutor的参数，这个参数说明也可以查看[线程池基本解析](必备Java知识/并发编程/线程池/线程池基本解析.md)
* 总体上来说，客户端调用的execute其实本质上就是线程池的execute方法，AsyncTask只是在此基础上进行了封装

#### 6.6 部分小结

* 总体上来说，客户端调用的execute其实本质上就是线程池的execute方法。

* 由于传入线程池的是Runnable对象，见6.4 SerialExecutor。所以最后会执行其中的run方法

```java
mTasks.offer(new Runnable() {
	public void run() {
    	try {
        	r.run();
		} finally {
        	scheduleNext();
		}
	}
});
```

* run方法中又会执行r.run，其中r是传递过来的，就是如下代码中的mFuture，即FutureTask实例，所以会执行mFuture的run方法

```java
exec.execute(mFuture);
```

* 其中mFuture是在构造函数中调用的，见5.1

```java
//把Callable的实例 传递给FutureTask实例
mFuture = new FutureTask<Result>(mWorker) {
	@Override
    protected void done() {
    	try {
        	postResultIfNotInvoked(get());
		}......
	}
};
```

* 同时创建mFuture实例的时候，把Callable的实例作为参数传入了

#### 6.7 run

* FutureTask.java#run

```java
public void run() {
	if (state != NEW ||
        !UNSAFE.compareAndSwapObject(this, runnerOffset,null, Thread.currentThread())){
		return;
    }
    try {
    	Callable<V> c = callable;
        if (c != null && state == NEW) {
        	V result;
            boolean ran;
            try {
            	result = c.call();
                ran = true;
			} catch (Throwable ex) {
            	result = null;
                ran = false;
                setException(ex);
                }
			if (ran)
            	set(result);
		}
	} finally {
    	// runner must be non-null until state is settled to
        // prevent concurrent calls to run()
        runner = null;
        // state must be re-read after nulling runner to prevent
        // leaked interrupts
        int s = state;
        if (s >= INTERRUPTING)
        	handlePossibleCancellationInterrupt(s);
	}
}
```

* 从run方法中可以看到，最后会调用c.call，也就是Callable的实例的call方法
* 所以最终回调用Callable的实例的call方法，其实例创建见5.1 

```java
//创建WorkerRunnable实例 并实现了call方法
mWorker = new WorkerRunnable<Params, Result>() {
	public Result call() throws Exception {
    	mTaskInvoked.set(true);
        Result result = null;
        try {
        	Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
            //noinspection unchecked
            result = doInBackground(mParams);
            Binder.flushPendingCommands();
		} catch (Throwable tr) {
        	mCancelled.set(true);
            throw tr;
		} finally {
        	postResult(result);
		}
        return result;
	}
};
```

* 在call方法中，可以看到调用了doInBackground方法，到此就执行了doInBackground方法
* finally最终执行了postResult(result);方法

#### 6.8 postResult

```java
private Result postResult(Result result) {
	@SuppressWarnings("unchecked")
    Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
    		new AsyncTaskResult<Result>(this, result));
	message.sendToTarget();
    return result;
}
```

* 从代码中可以看到这里使用的是，Handler+Looper+Message+MessageQueue的通信方式
* 发送的消息的what值为MESSAGE_POST_RESULT，在Handler的handleMessage中处理消息

#### 6.9 handleMessage

```java
private static class InternalHandler extends Handler {
	public InternalHandler(Looper looper) {
    	super(looper);
	}

    @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
    @Override
    public void handleMessage(Message msg) {
    	AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
        switch (msg.what) {
        	case MESSAGE_POST_RESULT:
            	// There is only one result
                result.mTask.finish(result.mData[0]);
                break;
			case MESSAGE_POST_PROGRESS:
            	result.mTask.onProgressUpdate(result.mData);
                break;
		}
	}
}
```

* 可以看到其what值，调用了 result.mTask.finish(result.mData[0]);方法，其中result是我们doInBackground方法的返回值，并且把它类型转换成AsyncTaskResult<?>对象

```java
@SuppressWarnings({"RawUseOfParameterizedType"})
private static class AsyncTaskResult<Data> {
	final AsyncTask mTask;
    final Data[] mData;

    AsyncTaskResult(AsyncTask task, Data... data) {
    	mTask = task;
        mData = data;
	}
}
```

* 通过上面的类的成员变量可知 result.mTask就是AsyncTask实例，然后调用其finish方法

#### 6.10 finish

* AsyncTask#finish

```java
private void finish(Result result) {
	if (isCancelled()) {
    	onCancelled(result);
	} else {
    	onPostExecute(result);
	}
    mStatus = Status.FINISHED;
}
```

* 如果判断没有取消的话，就会执行onPostExecute方法，同时把状态变成"已完成"

### 七、源码解析-publishProgress

#### 7.1 publishProgress

```java
@WorkerThread
protected final void publishProgress(Progress... values) {
	if (!isCancelled()) {
    	getHandler().obtainMessage(MESSAGE_POST_PROGRESS,
        		new AsyncTaskResult<Progress>(this, values)).sendToTarget();
	}
}
```

* 和上面一样的方式，所以我们还是要找到Handler的handleMessage方法

#### 7.2 handleMessage

```java
private static class InternalHandler extends Handler {
	public InternalHandler(Looper looper) {
    	super(looper);
	}

    @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
    @Override
    public void handleMessage(Message msg) {
    	AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
        switch (msg.what) {
        	case MESSAGE_POST_RESULT:
            	// There is only one result
                result.mTask.finish(result.mData[0]);
                break;
			case MESSAGE_POST_PROGRESS:
            	result.mTask.onProgressUpdate(result.mData);
                break;
		}
	}
}
```

* 对其what值MESSAGE_POST_PROGRESS，和上面一样，最终会调用AscyncTask的onProgressUpdate方法
* 至此我们重写或者实现的方法都被调用了。

### 八、总结

#### 8.1 execute主要流程

* 客户端.execute
* AsyncTask.executeOnExecutor

  * 此方法里会执行onPreExecute
* sDefaultExecutor.execute

  * sDefaultExecutor=SERIAL_EXECUTOR
  * SERIAL_EXECUTOR = new SerialExecutor()
* sDefaultExecutor.execute等同于SerialExecutor.execute
* THREAD_POOL_EXECUTOR.execute(Runnable)

  * THREAD_POOL_EXECUTOR=threadPoolExecutor
  * threadPoolExecutor=new ThreadPoolExecutor()
* THREAD_POOL_EXECUTOR.execute等同于ThreadPoolExecutor.execute(Runable)
* Runable.run
* r.run 

  * r=mFuture
  * mFuture = new FutureTask<Result>(mWorker)
* r.run等同于FutureTask实例mFuture .run
* mWorker.call

  * 此方法中会调用doInBackground
  * 此方法中会调用postResult
* message.sendToTarget

  * message.what=MESSAGE_POST_RESULT
* mHandler.handleMessage
*  result.mTask.finish(result.mData[0])

  * result.mTask是AsyncTask的实例
*   AsyncTask.finish
*   AsyncTask.onPostExecute

### 九、常见问题

#### 9.1 常用方法所在的线程

* AsyncTask 目前只支持一个构造方法，其他两个是隐藏的，所以Looper一定是主线程的Looper，Handler也是主线程的Handler
* onPreExecute方法在AsyncTask所在的线程(AsyncTask在主线程就在主线程，AsyncTask在子线程就在子线程。但是AsyncTask在子线程时，自然就无法对UI进行初始化)
* doInBackground方法在AsyncTask所在的线程的子线程池中，(因为是FutureTask+Callable+ThreadPoolExecutor的方式创建的)
* onProgressUpdate方法在主线程中(因为这里是Handler+Looper方式，都是主线程的对象)
* onPostExecute方法在主线程中(因为这里是Handler+Looper方式，都是主线程的对象)

#### 9.2 为什么 AsyncTask 的对象只能被调用一次，否则会出错？（每个状态只能执行一次）

* 从上面我们知道，AsyncTask 有 3 个状态，分别为 PENDING、RUNNING、FINSHED，而且每个状态在 AsyncTask 的生命周期中有且只执行一次。由于在执行完 execute 方法的时候会先对 AsyncTask 的状态进行判断，如果是 PENDING（等待中）的状态，就会往下执行并将 AsyncTask 的状态设置为 RUNNING（运行中）的状态；否则会抛出错误。AsyncTask finish 的时候，AsyncTask 的状态会被设置为 FINSHED 状态。

```java
//判断当前的状态 如果不是等待中的话 就抛出异常
if (mStatus != Status.PENDING) {
	switch (mStatus) {
    	case RUNNING:
        	throw new IllegalStateException("Cannot execute task:"
            	+ " the task is already running.");
		case FINISHED:
        	throw new IllegalStateException("Cannot execute task:"
            	+ " the task has already been executed "
                + "(a task can be executed only once)");
	}
}
//修改状态为 运行中
mStatus = Status.RUNNING;
```

#### 9.3 AsyncTask 调用 cancel() 任务是否立即停止执行？onPostExecute() 还会被调用吗？onCancelled() 什么时候被调用？

* 任务不会立即停止的，我们调用 cancel 的时候，只是将 AsyncTask 设置为 canceled（可取消）状态，我们从以下代码可以看出，AsyncTask 设置为已取消的状态，那么之后 onProgressUpdate 和 onPostExecute 都不会被调用，而是调用了 onCancelled() 方法。onCancelled() 方法是在异步任务结束的时候才调用的。时机是和 onPostExecute 方法一样的，只是这两个方法是互斥的，不能同时出现。

```java
@WorkerThread
protected final void publishProgress(Progress... values) {
	if (!isCancelled()) {
    	getHandler().obtainMessage(MESSAGE_POST_PROGRESS,
        	new AsyncTaskResult<Progress>(this, values)).sendToTarget();
	}
}
    
// 线程执行完之后才会被调用
private void finish(Result result) {
	if (isCancelled()) {
    	onCancelled(result);
	} else {
    	onPostExecute(result);
	}
    mStatus = Status.FINISHED;
}
```




