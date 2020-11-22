

[TOC]

# Handler和Message和MessageQueue和Looper

参考文章：

* [Android 异步消息处理机制（Handler 、 Looper 、MessageQueue）源码解析](http://blog.csdn.net/amazing7/article/details/51424038)
* [Handler消息机制，讲解Handler、Message、MessageQueue、Looper之间的关系](https://blog.csdn.net/nmyangmo/article/details/82260616)

### 一、前言

* 是进程内部、线程间的一种通信机制

#### 1.1 Message

* `handler`接受和处理的消息对象

#### 1.2 MessageQueue

* 存储消息对象的队列
* 优先级队列，最早执行的消息越早出队列(根据执行时间排序的)。

#### 1.3 Looper

* 每一个线程中只有一个`Looper`，它负责管理对应的`MessageQueue`，会不断的在`MessageQueue`中取出消息，并将消息分给对应的`handler`进行处理。一个Looper只有一个MessageQueue。

#### 1.4 Handler

* 发送消息，它能把消息发送给`Looper`管理的`MessageQueue`。

* 处理消息，它负责处理`Looper`分给它的消息

#### 1.5 说明

* 要想使用`handler`，首先要保证当前线程存在`Looper`对象；主线程不需要主动创建`Looper`对象是因为主线程已经帮你准备好了，见`android.app.ActivityThread->Looper.prepareMainLooper()`；如果我们想要在子线程使用handler来接受数据，需要先通过`Looper.prepare()`创建`Looper`。

  ```java
  Looper.prepare();
  ...//创建Handler并传入
  Looper.loop();
  ```

### 二、Message源码解析

#### 2.1 主要成员变量

* Message的部分成员变量如下所示

```java
public final class Message implements Parcelable {
    ......
	/*package*/ Message next;

    private static final Object sPoolSync = new Object();
    private static Message sPool;
    private static int sPoolSize = 0;

    private static final int MAX_POOL_SIZE = 50;

    private static boolean gCheckRecycle = true;
    ......
}  
```

* 从next的成员变量，我们可以看出来这是一个单链表的结构
* 通过sPool以及sPoolSize以及MAX_POOL_SIZE成员变量，我们可以知道维系了一个池子，sPoolSize记录了当前池子的大小，MAX_POOL_SIZE表示池子的最大容量。
* 从后续具体代码我们知道采用的是头插法进入的池子，同时取数据也是从头部取出的，头部的sPoolSize就代表了当前池子的大小。
* gCheckRecycle变量用来标志是否可以回收

#### 2.2 obtain

##### 2.2.1 obtain的好处

* 使用obtain()获取Message对象，会优先从池里拿数据，如果池里没有再创建对象。对于大量Message的时候，使用obtain能够有效的避免频繁的创建对象(new Message)以及置空后的GC。这样能够有效的防止内存抖动，避免程序OOM。

##### 2.2.2 源码解析

* 通过obtain获取Message消息，有许多的重载方法，最终会调用如下的方法

```java
public static Message obtain() {
	synchronized (sPoolSync) {
    	if (sPool != null) {
        	Message m = sPool;
            sPool = m.next;
            m.next = null;
            m.flags = 0; // clear in-use flag
            sPoolSize--;
            return m;
		}
	}
    return new Message();
}
```

* 如果Message对象已存在，可以使用obtain(msg)方法，最终也会调用obtain()。
* 从上述代码，我们可以知道如果池子不为空，会从池子里拿，并且是从头部拿的。

#### 2.3 recycleUnchecked

* 消息回收

```java
void recycleUnchecked() {
	// Mark the message as in use while it remains in the recycled object pool.
    // Clear out all other details.
    flags = FLAG_IN_USE;
    what = 0;
    arg1 = 0;
    arg2 = 0;
    obj = null;
    replyTo = null;
    sendingUid = -1;
    when = 0;
    target = null;
    callback = null;
    data = null;

    synchronized (sPoolSync) {
    	if (sPoolSize < MAX_POOL_SIZE) {
        	next = sPool;
            sPool = this;
            sPoolSize++;
		}
	}
}
```

* 消息的回收不是将Message对象销毁，而是将Message对象的值恢复初始化值，如果小于MAX_POOL_SIZE就放入池里，等待使用，减少申请开辟和回收空间的时间。
* 同时从代码里我们也看出来使用的头插法，最头的Message中的sPoolSize值就是池子的当前大小

### 三、Handler源码解析

#### 3.1 构造函数

```java
public Handler(Callback callback, boolean async) {
	if (FIND_POTENTIAL_LEAKS) {
    	final Class<? extends Handler> klass = getClass();
        if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) 			&&(klass.getModifiers() & Modifier.STATIC) == 0) {
        	Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
            klass.getCanonicalName());
		}
	}

    //获取Looper对象
    mLooper = Looper.myLooper();
    if (mLooper == null) {
    	throw new RuntimeException("Can't create handler inside thread that has not 				called Looper.prepare()");
	}
	//获取MessageQueue对象
    mQueue = mLooper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```

* 会获取当前线程的`Looper`对象，再通过`Looper`获取`MessageQueue`对象，这样就可以方便的把消息加入到`MessageQueue`中

#### 3.2 sendMessageAtTime

* 有很多重载的发送消息的方法，但是最后都会调用到该方法sendMessageAtTime

```java
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
	MessageQueue queue = mQueue;
    if (queue == null) {
    	RuntimeException e = new RuntimeException(
        	this + " sendMessageAtTime() called with no mQueue");
		Log.w("Looper", e.getMessage(), e);
        return false;
	}
    return enqueueMessage(queue, msg, uptimeMillis);
}
```

* 最后会调用enqueueMessage方法

#### 3.3 enqueueMessage

```java
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
	msg.target = this;
    if (mAsynchronous) {
    	msg.setAsynchronous(true);
	}
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

* 最终调用`MessageQueue`的`enqueueMessage`方法，把消息加入到队列中去。同时把Handler实例赋值给msg.target。

#### 3.4 dispatchMessage

##### 3.4.1 源码解析

* `Looper.loop`的时候，我们获取到消息的时候，最终会调用`msg.target.dispatchMessage(msg)`，其中msg.target对象就是Handler实例，所以就会调用Handler的dispatchMessage方法处理消息。

```java
public void dispatchMessage(Message msg) {
	if (msg.callback != null) {
    	handleCallback(msg);
	} else {
    	if (mCallback != null) {
        	if (mCallback.handleMessage(msg)) {
            	return;
			}
		}
		handleMessage(msg);
	}
}
```

* 上述各种判断如下实例所示

##### 3.4.2 实例(msg.callback != null)

* 如果msg.callback为空的话，就会执行handleCallback()方法

* handleCallback()

```java
private static void handleCallback(Message message) {
	message.callback.run();
}
```

* 实例

```java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        Handler handler = new Handler() {
            @Override
            public void handleMessage(@NonNull Message msg) {
                Log.e("MainActivity", "handleMessage");
                super.handleMessage(msg);
            }
        };

        Message message = Message.obtain(handler, new Runnable() {
            @Override
            public void run() {
                Log.e("MainActivity", "msg.callback != null");
            }
        });
        handler.sendMessage(message);
    }
}
```

* 输出：MainActivity: msg.callback != null

##### 3.4.3 实例(mCallback != null)

* 如果mCallback != null，就会执行mCallback.handleMessage(msg)。

```java
public interface Callback {
	public boolean handleMessage(Message msg);
}
```

* 对应的就是它的实现类的handleMessage方法

* 实例

```java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        Handler handler = new Handler(new Handler.Callback() {
            @Override
            public boolean handleMessage(@NonNull Message msg) {
                Log.e("MainActivity", "Handler.Callback:handleMessage");
                return false;
            }
        });

        Message message = handler.obtainMessage();
        handler.sendMessage(message);
    }
}
```

* 输出：MainActivity: Handler.Callback:handleMessage

##### 3.4.4 handleMessage

* 都为空的情况下就会进入handleMessage，我们可以重写这个方法实现自己的逻辑代码

```java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        Handler handler = new Handler(){
            @Override
            public void handleMessage(@NonNull Message msg) {
                Log.e("MainActivity", "handleMessage");
                super.handleMessage(msg);
            }
        };

        Message message = handler.obtainMessage();
        handler.sendMessage(message);
    }
}
```

* 输出：MainActivity: handleMessage

### 四、Looper源码解析

#### 4.1 主要成员变量

```java
//使用ThreadLocal来存储Looper，以及使用final关键字，实现了线程隔离
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
//主线程的Looper
private static Looper sMainLooper;
//Looper所属线程的消息队列
final MessageQueue mQueue;
//Looper所属线程
final Thread mThread;
```

#### 4.2 prepare()

```java
private static void prepare(boolean quitAllowed) {
	if (sThreadLocal.get() != null) {
    	throw new RuntimeException("Only one Looper may be created per thread");
    }
	sThreadLocal.set(new Looper(quitAllowed));
}
```

* 使用ThreadLocal的方式以及sThreadLocal被final关键字修饰，保证了当前线程只有一个Looper对象。同时又进行了非空判断，不允许取修改Looper对象。保证了一个线程只有一个固定不能修改的Looper。

#### 4.3 构造方法

```java
private Looper(boolean quitAllowed) {
	mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
}
```

* 构造方法是私有的，所以不能直接调用。需要使用上述的prepare创建。创建`Looper`对象的时候，同时创建了`MessageQueue`，并让`Looper`绑定当前线程。由于一个线程只有一个Looper，所以该构造方法只会调用一次，所以MessageQueue也是唯一的。所以线程、Looper、MessageQueue都是唯一的。

#### 4.4 loop()

```java
public static void loop() {
    //获取当前线程的Looper
	final Looper me = myLooper();
    //如果为空 则表示没有Looper 抛出异常
    if (me == null) {
    	throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
	}
    
    //获取当前Looper的MessageQueue
    final MessageQueue queue = me.mQueue;

    Binder.clearCallingIdentity();
	final long ident = Binder.clearCallingIdentity();

    //死循环遍历
	for (;;) {
        //获取队列中的下一条消息，
        //调用的是MessageQueue的next方法，后续讲解，需要知道的是可能会进入阻塞休眠状态
		Message msg = queue.next(); 
        
        //如果msg==null，则退出死循环，用来quit用的
        if (msg == null) {
			return;
		}

        //log打印
        final Printer logging = me.mLogging;
        if (logging != null) {
        	logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
		}

        //
        final long slowDispatchThresholdMs = me.mSlowDispatchThresholdMs;

		final long traceTag = me.mTraceTag;
        if (traceTag != 0 && Trace.isTagEnabled(traceTag)) {
        	Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
		}
        final long start = (slowDispatchThresholdMs == 0) ? 0 : SystemClock.uptimeMillis();
        final long end;
        try {
            //msg.target是一个Handler对象
			//哪个Handler把这个Message发到队列里，这个Message会持有这个Handler的引用
            //调用handler的dispatchMessage方法
        	msg.target.dispatchMessage(msg);
            end = (slowDispatchThresholdMs == 0) ? 0 : SystemClock.uptimeMillis();
		} finally {
        	if (traceTag != 0) {
            	Trace.traceEnd(traceTag);
			}
		}
        if (slowDispatchThresholdMs > 0) {
        	final long time = end - start;
            if (time > slowDispatchThresholdMs) {
            	Slog.w(TAG, "Dispatch took " + time + "ms on "
                	+ Thread.currentThread().getName() + ", h=" +
                    msg.target + " cb=" + msg.callback + " msg=" + msg.what);
			}
		}

        if (logging != null) {
        	logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
		}

		final long newIdent = Binder.clearCallingIdentity();
        if (ident != newIdent) {
        	Log.wtf(TAG, "Thread identity changed from 0x"
            	+ Long.toHexString(ident) + " to 0x"
                + Long.toHexString(newIdent) + " while dispatching to "
                + msg.target.getClass().getName() + " "
                + msg.callback + " what=" + msg.what);
		}
		
        //调用Message的recycleUnchecked
        //如果池子没满的话，就会把Meesage的成员变量置成初始值，再放入池子中
        msg.recycleUnchecked();
	}
}
```

* 通过for(;;)来实现死循环，从MessageQueue中循环取数据，如果取到了Message对象数据，就调用msg.target.dispatchMessage(msg)将msg交给Handler去处理。其中msg.target就是Handler对象
* msg.recycleUnchecked();调用的就是Message的recycleUnchecked方法，上面已经做了详细的介绍了

#### 4.5 quitSafely

* 用来退出的，结束死循环的，会调用MessageQueue的quie方法，后面会详细介绍

```java
public void quitSafely() {
	mQueue.quit(true);
}
```

### 五、MessageQueue源码解析

#### 4.1 构造方法

```java
MessageQueue(boolean quitAllowed) {
	mQuitAllowed = quitAllowed;
    mPtr = nativeInit();
}
```

* `MessageQueue`初始化的时候会同时初始化底层的`NativeMessageQueue`对象，并且持有`NativeMessageQueue`的内存地址(`long`)。
* quitAllowed用来区分主线程的Looper和其他线程的Looper的，对于主线程的Looper来说，是不允许退出的。四大组件等都依赖于主线程的Looper和Handler。

#### 4.2 next()

* 取数据的方法

```java
Message next() {
	final long ptr = mPtr;
    if (ptr == 0) {
    	return null;
	}

    int pendingIdleHandlerCount = -1; // -1 only during first iteration
    int nextPollTimeoutMillis = 0;
    for (;;) {
    	if (nextPollTimeoutMillis != 0) {
        	Binder.flushPendingCommands();
		}

        //native底层实现堵塞逻辑，堵塞状态会到时间唤醒
        //也可被新消息唤醒，一旦唤醒会重新获取头消息，重新评估是否堵塞或者直接返回消息
        nativePollOnce(ptr, nextPollTimeoutMillis);

        //锁
        synchronized (this) {
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            
            if (msg != null && msg.target == null) {
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
			}
            
            //存在消息
			if (msg != null) {
        		if (now < msg.when) {
            		//当前消息还没有到时间 计算下一次时间
	                nextPollTimeoutMillis = (int) Math.min(msg.when - now, 			
						Integer.MAX_VALUE);
				} else {
					//获取到消息 且时间到了
					mBlocked = false;
                    if (prevMsg != null) {
                    	prevMsg.next = msg.next;
					} else {
                    	mMessages = msg.next;
					}
                    msg.next = null;
                    if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                    msg.markInUse();
                    return msg;
				}
			} else {
            	//没有消息
				nextPollTimeoutMillis = -1;
			}

            if (mQuitting) {
            	dispose();
				return null;
			}

            if (pendingIdleHandlerCount < 0
            	&& (mMessages == null || now < mMessages.when)) {
				pendingIdleHandlerCount = mIdleHandlers.size();
			}
            if (pendingIdleHandlerCount <= 0) {
            	// No idle handlers to run.  Loop and wait some more.
                mBlocked = true;
                continue;
			}

            if (mPendingIdleHandlers == null) {
            	mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
			}
            mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
		}

        for (int i = 0; i < pendingIdleHandlerCount; i++) {
        	final IdleHandler idler = mPendingIdleHandlers[i];
            mPendingIdleHandlers[i] = null; // release the reference to the handler

            boolean keep = false;
            try {
            	keep = idler.queueIdle();
			} catch (Throwable t) {
            	Log.wtf(TAG, "IdleHandler threw exception", t);
			}

            if (!keep) {
            	synchronized (this) {
                	mIdleHandlers.remove(idler);
				}
			}
		}

        pendingIdleHandlerCount = 0;

        nextPollTimeoutMillis = 0;
	}
}
```

* `next()`中，因为消息队列是按照延迟时间排序的，所以先考虑延迟最小的也就是头消息。当头消息为空，说明队列中没有消息了，`nextPollTimeoutMIllis`就被赋值为-1,当头消息延迟时间大于当前时间，把延迟时间和当前时间的差值赋给`nextPollTimeoutMIllis`。当消息延迟时间小于等于0，直接返回`msg`给`handler`处理
* `nativePollOnce(ptr, nextPollTimeoutMillis)`方法是`native`底层实现堵塞逻辑，堵塞状态会到时间唤醒，也可被新消息唤醒，一旦唤醒会重新获取头消息，重新评估是否堵塞或者直接返回消息。

#### 4.3 enqueueMessage()

* 把消息加入到优先级队列中取

```java
boolean enqueueMessage(Message msg, long when) {
	if (msg.target == null) {
    	throw new IllegalArgumentException("Message must have a target.");
	}
    if (msg.isInUse()) {
    	throw new IllegalStateException(msg + " This message is already in use.");
	}

    synchronized (this) {
    	if (mQuitting) {
        	IllegalStateException e = new IllegalStateException(
           		msg.target + " sending message to a Handler on a dead thread");
                Log.w(TAG, e.getMessage(), e);
                msg.recycle();
                return false;
		}

        msg.markInUse();
        msg.when = when;
        Message p = mMessages;
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
		} else {
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            for (;;) {
            	prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                	break;
                }
                if (needWake && p.isAsynchronous()) {
                	needWake = false;
                }
			}
            msg.next = p; // invariant: p == prev.next
            prev.next = msg;
		}

        if (needWake) {
        	nativeWake(mPtr);
        }
	}
    return true;
}
```

* 消息入栈时，首先会判断新消息如果是第一个消息或者新消息没有延迟或者新消息延迟时间小于队列第一个消息的，都会立刻对这个消息进行处理。只有当消息延迟大于队列消息时，才会依次遍历消息队列，将消息按延迟时间插入消息队列响应位置。

#### 4.4 quit

* 退出，用来结束Looper的死循环的

```java
void quit(boolean safe) {
	if (!mQuitAllowed) {
    	throw new IllegalStateException("Main thread not allowed to quit.");
	}

    synchronized (this) {
    	if (mQuitting) {
        	return;
		}
        mQuitting = true;

        if (safe) {
        	removeAllFutureMessagesLocked();
		} else {
            //把所有的消息recycleUnchecked掉
            //message.recycleUnchecked
        	removeAllMessagesLocked();
		}
	//唤醒堵塞
	nativeWake(mPtr);
	}
}
```

* 把mQuitting赋值为true，并唤醒next中的线程阻塞。
* 然后接着向下执行next中的代码，重点代码如下，由于mQuitting为true，所以Message的next方法会返回null

```java
if (mQuitting) {
	dispose();
	return null;
}
```

* 对于Looper的loop方法中，拿到了queue.next的返回值为null，会接着执行到如下代码，就会退出死循环，就能释放资源了

```java
//如果msg==null，则退出死循环，用来quit用的
if (msg == null) {
	return;
}
```

### 六、整体流程图

![handler工作机制](..\..\images\Android组件内核\Android消息机制\handler工作机制.png)

### 七、面试常见问题

#### 7.1 一个线程有几个Handler？

* 多个，想new几个就有几个。

#### 7.2 一个线程有几个Looper？如何保证?

* 一个线程只有一个Looper，使用ThreadLocal来保证的。具体看[ThreadLocal解析](ThreadLocal解析.md)。同时Looper的prepare方法会判断是否已经set过，如果set过就会抛出异常。

#### 7.3 为什么需要Handler?

* 当主线程处理一个消息超过5秒，就会出现ANR，所以需要把一些处理时间比较长的消息，放在一个单独的线程中进行处理，把处理以后的结果，返回给主线程运行，就需要用Handle进行线程间的通信

#### 7.4 Handler内存泄露的原因？为什么其他的内部类没有这个问题？怎么解决？

* 匿名内部类
* 内部类：持有外部类的对象 
* MessageQueue->Message->Handler->外部类Activity。所以即使Activity调用了onDestory，由于有强引用的持有链关系，所以如果MessageQueue没有释放，那么Activity就不能释放，就会造成内存泄漏问题
* 静态内部类+弱引用解决这个问题。静态内部类不持有外部类的引用。弱引用方式使用外部类的成员。
* 正确的使用Handler代码如下实例所示

```java
public class MainActivity extends AppCompatActivity {
    private MyHandler myHandler = new MyHandler(this);

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        //开启线程
        new Thread(new MyRunnable()).start();
    }

    public static class MyHandler extends Handler {
        //把所依赖的引用对象，使用弱引用 当Activity为空的时候 可以被回收
        WeakReference<MainActivity> mReference;

        //通过构造方法把activity传入进来
        public MyHandler(MainActivity activity) {
            mReference = new WeakReference<MainActivity>(activity);
        }

        @Override
        public void handleMessage(@NonNull Message msg) {
            Log.e("MainActivity", "handleMessage");

            Activity activity = mReference.get();
            if (activity != null) {
                switch (msg.what) {
                    case 1:
                        //通过弱引用的方式持有了外部Activity 可以进行各种相应的操作了
                        Log.e("MainActivity", "startActivity");
//                        activity.startActivity("自己的Intent");
                        break;
                }
            }

            super.handleMessage(msg);
        }
    }

    class MyRunnable implements Runnable {

        @Override
        public void run() {
            //执行一些耗时操作
            //执行完成 然后发送消息
            Message msg = myHandler.obtainMessage();
            msg.what = 1;
            myHandler.sendMessage(msg);
        }
    }
}
```

#### 7.5 为何主线程可以new Handler?

* 因为在ActivityThread中已经 `Looper.prepareMainLooper();`以及`Looper.loop()`了，所以在主线程中不需要Looper.prepare就可以直接new Handler。

#### 7.6 如何在子线程中使用Handler?

1. 调用Looper的prepare()方法为当前非主线程创建Looper对象，创建Looper对象时，他的构造器创建与之配套的MessageQueue
2. 创建Handler子类实例，重写handleMessage()方法，该方法处理来自其他线程的消息
3. 调用Looper的loop()方法来启动Looper
```java
使用这个handler实例在任何其他线程中发送消息，最终处理消息的代码都会在你创建的handler实例的线程中运行
```

#### 7.7 子线程中维护的Looper，消息队列无消息的时候处理方案是什么？有什么用？

* 消息队列无消息的时候，Looper会阻塞进入休眠，当有新的消息或者消息到时间了就会唤醒。阻塞休眠是底层实现的。
* Looper#loop

```
Message msg = queue.next(); // might block
```

* MessageQueue#next

```java
//native底层实现堵塞逻辑，堵塞状态会到时间唤醒
//也可被新消息唤醒，一旦唤醒会重新获取头消息，重新评估是否堵塞或者直接返回消息
//如果nextPollTimeoutMillis等于-1会触发堵塞
nativePollOnce(ptr, nextPollTimeoutMillis);
```

* 如果确定没有消息了，looper.quitSafely()退出死循环，结束堵塞。子线程会接着执行Looper.loop后面的，执行完毕后可以释放资源。

#### 7.8  多个Handler往一个MessageQueue中发送数据，怎么保证线程安全？

* Synchronized
* Synchronized(this)：修饰代码块
* 一个线程只有一个Looper，一个Looper只有一个MessageQueue

#### 7.9 使用Message时应该怎么创建它？

* Handler对象的obtainMessage，然后会调用Message的obtain方法

* obtain()；避免频繁的创建和GC，有效避免内存抖动，防止OOM
* 应该避免使用new Message，而是使用obtainMessage获取message，再赋值
* 享元设计模式

#### 7.10 Looper死循环为什么不会导致应用卡死？

* ANR：五秒内没有相应输入的事件；广播接收器在10秒内没有执行完成
* 产生的ANR的问题不是主线程睡眠了，而是因为输入事件没有响应，输入事件没有响应就没办法唤醒这个Looper，才加了这个5秒的限制。