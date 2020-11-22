[TOC]

## Serializable和Parcelable比较

#### 参考

* [Serializable 和Parcelable 的区别(Android每日面试题)](https://juejin.im/post/6883309627933458445)

* [android中的序列化机制原理]([https://sufushi.github.io/2018/01/24/android%E4%B8%AD%E7%9A%84%E5%BA%8F%E5%88%97%E5%8C%96%E6%9C%BA%E5%88%B6%E5%8E%9F%E7%90%86/#Parcel%E7%9A%84%E4%BD%BF%E7%94%A8%E4%B8%8E%E5%AE%9E%E7%8E%B0](https://sufushi.github.io/2018/01/24/android中的序列化机制原理/#Parcel的使用与实现))
* [Java对象序列化底层原理源码解析WhatHowWhyOther](https://cloud.tencent.com/developer/article/1125165)

### 一、前提知识

#### 1.1序列化和反序列化
* 序列化：由于内存中的对象是暂时的，无法长期保存，为了把对象的状态保存下来，写入磁盘或者其他介质中的操作，这就是序列化。序列化的对象可以在网络上进行传输，也可以保存在本地。
* 反序列化：反序列化就是序列化的反向操作，也就是说把在磁盘或者其他介质中的对象，反序列化到内存中去，以便进行操作。

#### 1.2.为什么需要序列化？
* 永久的保存对象，将对象数据保存在文件、磁盘或者数据库中 。
* 通过序列化操作将对象数据在网络上进行传输 。
* 将对象序列化之后在进程间进行传输 。
* 在安卓中使用 Intent 进行传输时候，数据类型较为复杂的需要进行序列化操作 。

#### 1.3.怎么进行序列化？
* 一个对象需要实现序列化操作，该类就必须实现了Serializable接口或者Parcelable接口

### 二、Serializable

#### 2.1 简介

* Serializable是java提供的一个序列化接口，它是一个空接口，专门为对象提供标准的序列化和反序列化操作，使用Serializable实现类的序列化比较简单，只要在类声明中实现Serializable接口即可，同时强烈建议声明序列化标识
* 对于序列化和反序列化的时候，serialVersionUID必须是相同的，不然反序列化会报错的。如果自己设置了serialVersionUID的话，那么如果增删了某些成员变量的话，这时候反序列化还是会成功的。如果不设置的话，修改这些的话，系统会重新计算serialVersionUID，如果不一致的话，就会反序列化失败，程序就会crash。
* Serializable使用了反射，过程比较慢，在序列化的时候会产生大量的临时变量，从而引起频繁的GC
* 静态成员变量属于类不属于对象，所以不会参与序列化过程；其次用transient关键字标记的成员变量不参与序列化过程。

#### 2.2 实例

* Test.java

```java
public class Test {
    public static void main(String[] args) {
        try {
            //序列化写入
            FileOutputStream fos = new FileOutputStream("C:\\Users\\ZHHN\\Desktop\\temp.out");
            ObjectOutputStream oos = new ObjectOutputStream(fos);
            Book book = new Book(1, "书1", 10.99);
            oos.writeObject(book);
            oos.flush();
            oos.close();

            //反序列化
            FileInputStream fis = new FileInputStream("C:\\Users\\ZHHN\\Desktop\\temp.out");
            ObjectInputStream ois = new ObjectInputStream(fis);
            Book readBook = (Book) ois.readObject();
            System.out.println(readBook.toString());
        } catch (ClassNotFoundException | IOException e) {
            e.printStackTrace();
        }
    }
}

class Book implements Serializable {
    private static final long serialVersionUID = 1L;

    private int id;
    private String name;
    private double price;

    public Book(int id, String name, double price) {
        this.id = id;
        this.name = name;
        this.price = price;
    }

    @Override
    public String toString() {
        return "Book{" +
                "id=" + id +
                ", name='" + name + '\'' +
                ", price=" + price +
                '}';
    }
}
```

* temp.out

```text
aced 0005 7372 0004 426f 6f6b 0000 0000
0000 0001 0200 0349 0002 6964 4400 0570
7269 6365 4c00 046e 616d 6574 0012 4c6a
6176 612f 6c61 6e67 2f53 7472 696e 673b
7870 0000 0001 4025 fae1 47ae 147b 7400
04e4 b9a6 31
```

#### 2.3 原理源码解析

##### 2.3.1 getDefaultSerialFields

* 序列化的时候所有的数据都是来自于ObejctStreamClass对象，在生成ObjectStreamClass的构造函数中会调用fields = getSerialFields(cl);这句代码来获取需要被序列化的字段，getSerialFields()方法实际上是调用getDefaultSerialFields()方法的

```java
private static ObjectStreamField[] getDefaultSerialFields(Class<?> cl) {
    Field[] clFields = cl.getDeclaredFields();
    ArrayList<ObjectStreamField> list = new ArrayList<>();
    int mask = Modifier.STATIC | Modifier.TRANSIENT;
 
    for (int i = 0; i < clFields.length; i++) {
        if ((clFields[i].getModifiers() & mask) == 0) {
            // 如果字段既不是static也不是transient的才会被加入到需要被序列化字段列表中去
            list.add(new ObjectStreamField(clFields[i], false, true));
        }
    }
    int size = list.size();
    return (size == 0) ? NO_FIELDS :
        list.toArray(new ObjectStreamField[size]);
}
```

* 从上面的代码中可以很明显的看到，在计算需要被序列化的字段的时候会把被static和transient修饰的字段给过滤掉。在进行反序列化的时候会给默认值。所以static和transient修饰的字段不参与序列化过程。

### 三、Parcelable

#### 3.1 简介

* Parcelable是android特有的序列化API，它的出现是为了解决Serializable在序列化的过程中消耗资源严重的问题。

#### 3.2 Parcelable的使用

* **注意写入和读取的顺序一致**

* implements Parcelable
* 重写writeToParcel方法,序列化写入(writeInt之类的)
* 重写describeContents方法
* 实例化静态内部对象CREATOR,实现接口Parcelable.Creator,反序列化读取(readInt之类的)

#### 3.3 实例代码

```java
public class Book implements Parcelable {
    private int id;
    private String name;
    private double price;

    public Book(int id, String name, double price) {
        this.id = id;
        this.name = name;
        this.price = price;
    }

    protected Book(Parcel in) {
        id = in.readInt();
        name = in.readString();
        price = in.readDouble();
    }

    //反序列化
    public static final Creator<Book> CREATOR = new Creator<Book>() {
        @Override
        public Book createFromParcel(Parcel in) {
            return new Book(in);
        }

        @Override
        public Book[] newArray(int size) {
            return new Book[size];
        }
    };

    //内容描述
    @Override
    public int describeContents() {
        return 0;
    }


    //序列化
    @Override
    public void writeToParcel(Parcel parcel, int i) {
        parcel.writeInt(id);
        parcel.writeString(name);
        parcel.writeDouble(price);
    }
}
```

### 四、Parcel

#### 4.1 简介

* Parcel是一个容器，可以包含数据或者对象引用，并且能够用于Binder的传输，同时支持序列化以及跨进程之后进行反序列化，同时其提供了很多方法帮助开发者完成这些功能。

#### 4.2 Parcel的使用

* Parcel的获取

```java
Parcel parcel = Parcel.ontain();
```

* 向Parcel这个容器中传入数据

```java
parcel.writeInt(int val);
parcel.writeString(String str);
```

* 进行数据的序列化

```java
parcel.marshall();
```

* 在经过上一步处理之后，返回一个byte数组，主要的IPC相关的操作主要就是围绕此byte数组进行的。同时，由于parcel的读写都是一个指针操作的，这一步涉及到native的操作，所以，在将数据写入之后，需要将指针手动指向最初的位置。

```java
parcel.setDataPosition(0);
```

* 最后使用完Parcel还需要回收销毁

```java
parcel.recycle();
```

* 在IPC的另一端，需要进行Parcel的获取处理。在进行了IPC操作后，一般读取出来的就是之前序列化的byte数组，所以，首先要进行一个反序列化操作

```java
parcel.unmarshall(byte[] data, int offset, int length);
```

* 此时得到的parcel就是一个正常的parcel对象，这时就可以将之前我们所存入的数据按照顺序进行获取。

```java
parcel.readInt();
parcel.readString();
```

* 读取完毕后，同样是需要对parcel进行一个回收操作

```java
parcel.recycle();
```

#### 4.3 源码解析

##### 4.3.1 obtain

```java
public static Parcel obtain() {
	final Parcel[] pool = sOwnedPool;
    synchronized (pool) {
    	Parcel p;
        for (int i=0; i<POOL_SIZE; i++) {
        	p = pool[i];
            if (p != null) {
            	pool[i] = null;
                if (DEBUG_RECYCLE) {
                	p.mStack = new RuntimeException();
				}
                return p;
			}
		}
	}
    return new Parcel(0);
}
```

* Parcel的初始化，主要是使用一个对象池进行的，这样可以提高性能以及内存消耗。
* sOwnedPool主要是用来存储parcel的，obtain()方法首先会检索池子中的parcel对象，若是能取出parcel，那么将这个parcel返回，同时将这个位置置空，若是现在连池子都不存在的话，那么就直接新建一个parcel对象

##### 4.3.2 new Parcel(0)

```java
private Parcel(long nativePtr) {
	if (DEBUG_RECYCLE) {
    	mStack = new RuntimeException();
	}
	init(nativePtr);
}
```

* 最终会调用init方法

##### 4.3.3 init

```java
private void init(long nativePtr) {
	if (nativePtr != 0) {
    	mNativePtr = nativePtr;
        mOwnsNativeParcelObject = false;
	} else {
    	mNativePtr = nativeCreate();
        mOwnsNativeParcelObject = true;
	}
}
```

* 这里首先对参数进行检查，因为初始化传入的参数是0，那么直接执行nativeCreate()，并且将标志位mOwnsNativeParcelObject置为true，表示这个parcel已经在native进行了创建。此处的nativeCreate()是一个JNI方法

##### 4.3.4 nativeCreate

* nativeCreate对应的是android_os_Parcel_create方法

```java
static jint android_os_Parcel_create(JNIEnv* env, jclass clazz)
{
    Parcel* parcel = new Parcel();
    return reinterpret_cast<jint>(parcel);
}
```

* 这是一个jni的实现，首先调用了native的初始化，并且，返回操作这个对象的指针。接下来继续看到C++层的实现。

```
Parcel::Parcel()
{
    initState();
}
void Parcel::initState()
{
    mError = NO_ERROR;
    mData = 0;
    mDataSize = 0;
    mDataCapacity = 0;
    mDataPos = 0;
    ALOGV("initState Setting data size of %p to %d\n", this, mDataSize);
    ALOGV("initState Setting data pos of %p to %d\n", this, mDataPos);
    mObjects = NULL;
    mObjectsSize = 0;
    mObjectsCapacity = 0;
    mNextObjectHint = 0;
    mHasFds = false;
    mFdsKnown = true;
    mAllowFds = true;
    mOwner = NULL;
}
```

* 可以看出，对parce的初始化，只是在native层初始化了一些数据值，在完成初始化之后，就将这个操作指针返回，这样就完成了parcel的初始化。初始化完毕之后，就可以进行数据的写入了，首先写入一个int型数据，其Java层实现如下。

##### 4.3.5 writeXXX

* 以writeInt为例

```
public final void writeInt(int val) {
    nativeWriteInt(mNativePtr, val);
}
```

* 可以看出，Java层就纯粹是一个对于native实现的封装了，这时候的分析来到jni。

##### 4.3.6 nativeWriteInt

* nativeWriteInt对应的是android_os_Parcel_writeInt

```
static void android_os_Parcel_writeInt(JNIEnv* env, jclass clazz, jint nativePtr, jint val) {
    // 指针实际上是一个整型地址值，所以这里使用强转将int值转化为parcel类型的指针，然后使用这个指针来操作		   native的parcel对象
    Parcel* parcel = reinterpret_cast<Parcel*>(nativePtr);
    const status_t err = parcel->writeInt32(val);
    if (err != NO_ERROR) {
        signalExceptionForError(env, clazz, err);
    }
}
```

* 这里注意两个参数，一个是nativePtr，即之前传上去的指针，另一个是val，即需要写入的整型数据。再深入看一下写入的操作。

##### 4.3.7 writeInt32

```
status_t Parcel::writeInt32(int32_t val)
{
    return writeAligned(val);
}
```
```
template<class T>
status_t Parcel::writeAligned(T val) {
    COMPILE_TIME_ASSERT_FUNCTION_SCOPE(PAD_SIZE(sizeof(T)) == sizeof(T));

    if ((mDataPos+sizeof(val)) <= mDataCapacity) {
restart_write:
        *reinterpret_cast<T*>(mData+mDataPos) = val;
        return finishWrite(sizeof(val));
    }

    status_t err = growData(sizeof(val));
    if (err == NO_ERROR) goto restart_write;
    return err;
}
```

* 这个函数首先是一个断言检查，然后对输入的参数取size值，再加上之前已经移动的位置，判断是否超过了该Parcel所定义的能力值mDataCapacity。若是超过了能力值，那么直接将能力值进行扩大，扩大的值是val值的大小，并且，写入时候是以4字节对齐写入，通过PAD_SIZE(sizeof(T))宏定义来实现。

* 至此，Parcel就成功写入一个数据了。

### 五、两者的选用原则

- 在使用内存的时候，Parcelable比Serializable性能高，所以推荐使用Parcelable
- Serializable使用了反射，过程比较慢，在序列化的时候会产生大量的临时变量，从而引起频繁的GC
- Parcelable不能使用在要将数据存储在磁盘上的情况，因为Parcelable不能很好的保证数据的持久性在外界有变化的情况下，尽管Serializable效率低点，但此时还是建议使用Serializable 