[TOC]

# HashMap源码分析

### 参考

* [**源码解析之HashMap实现原理**](https://blog.csdn.net/pihailailou/article/details/82053420)

## 一、HashMap结构

HashMap底层采用的是数组+链表+红黑树的结构。如果链表大小大于阈值，链表会转换成红黑树；如果红黑树小于阈值，红黑树会转换成链表。

## 二、源码分析

### 2.1 Node

* HashMap中采用实现的Node进行存储键值对的数据
* 源码

```java
 static class Node<K,V> implements Map.Entry<K,V> {
	final int hash;
	final K key;
	V value;
	Node<K,V> next;
	......
	public final int hashCode() {
		return Objects.hashCode(key) ^ Objects.hashCode(value);
	}     
	public final boolean equals(Object o) {
		if (o == this)
			return true;
		if (o instanceof Map.Entry) {
			Map.Entry<?,?> e = (Map.Entry<?,?>)o;
			if (Objects.equals(key, e.getKey()) &&
				Objects.equals(value, e.getValue()))
				return true;
		}
		return false;
	}
 }
```

* 源码分析

```text
通过Node源码分析，很容易看出这是一个链表结构。Node中包含着next的Node。
```

* 源码

```java
    /**
     * The table, initialized on first use, and resized as
     * necessary. When allocated, length is always a power of two.
     * (We also tolerate length zero in some operations to allow
     * bootstrapping mechanics that are currently not needed.)
     */
    transient Node<K,V>[] table;
```

* 源码分析

```text
从源码中可以看到存在一个Node的数组，所以HashMap中既存在数组又存在链表
```

### 2.2 put流程

* 流程

```text
1.首先会调用哈希函数去计算key对应的hash
2.然后执行位运算hash&(table.length-1)，得到结点Node在数组中的存储位置index
3.若数组的index位置没有结点，则直接将改结点存入数组，若该index位置有结点，又分为两种情况
	* 该位置存放一个链表，在链表上进行插入，若key相同更新value值
	* 该位置存放一个红黑树，在红黑树上插入元素，若key相同更新value值
```

* 链表存在的目的

```text
第一步会计算hash值，由于哈希函数的局限性，所以必然存在不同的对象hash值相同，即哈希碰撞。
在出现哈希碰撞的时候，多个不同的key对应的存储数组位置index相同，链表就是用来解决这个问题的。
```

* 红黑树存在的目的

```java
// 由于链表的局限性，当链表过长的时候，会导致查询效率大大降低。
static final int TREEIFY_THRESHOLD = 8;
// 所以HashMap规定当链表的结点数大于8时候，会将链表转换为红黑树。
static final int UNTREEIFY_THRESHOLD = 6;
// HashMap中规定，当红黑树的结点个数小于6时，会将红黑树转换为链表结构
```

### 2.3 位运算hash&(table.length-1)

* 源码

```java
static final int hash(Object key) {
	int h;
	return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

* (h = key.hashCode()) ^ (h >>> 16) 解释

```text
首先将key的值取hashCode得到32位的h，然后将h无符号的右移16位，得到高16位和h进行异或。
结果就是h的高16位不变，低16位和高16位异或得到结果的低16位。
```

* 原因

```text
hash&(table.length-1)时，由于table.length的长度一定是2的倍数，所以
当hash=1，length=16，hash&(table.length-1)值为1
当hash=6，length=16，hash&(table.length-1)值为6
当hash=16，length=16，hash&(table.length-1)值为0
当hash=17，length=16，hash&(table.length-1)值为1
这样保证了索引值index一定在table数组的索引范围内
因为table.length为2的倍数，则table.length-1的高位永远为0，低位对应的为1。
所以与运算，高位永远为0，低位则是hash的低位的值。
```

### 2.4 边界变量

* 源码

```java
//数组的初始容量 2的4次方 16
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
//数组的最大容量 2的30次方
static final int MAXIMUM_CAPACITY = 1 << 30;
//加载因子
static final float DEFAULT_LOAD_FACTOR = 0.75f;
//当链表长度大于8，转换成红黑树
static final int TREEIFY_THRESHOLD = 8;
//当红黑树结点个数小于6 转换成链表
static final int UNTREEIFY_THRESHOLD = 6;
/**
* The smallest table capacity for which bins may be treeified.
* (Otherwise the table is resized if too many nodes in a bin.)
* Should be at least 4 * TREEIFY_THRESHOLD to avoid conflicts
* between resizing and treeification thresholds.
*/
//这个值至少是TREEIFY_THRESHOLD的4倍，以免resizing和treeification之间产生冲突
static final int MIN_TREEIFY_CAPACITY = 64;
```

在HashMap中有一个threshold变量，threshold=数组大小*加载因子。当集合中的结点个数大于threshold时，会进行数组扩容。

### 2.5 put方法

* 源码

```java
public V put(K key, V value) {
	return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,boolean evict) {
    Node<K,V>[] tab;//存放table
    Node<K,V> p;//临时存放一个结点
    int n, i;//n数组的长度
	if ((tab = table) == null || (n = tab.length) == 0)
		n = (tab = resize()).length;//如果为空的话，初始化table
	if ((p = tab[i = (n - 1) & hash]) == null)//hash&(n-1)计算index看是否已存在
		tab[i] = newNode(hash, key, value, null);//不存在 创建
	else {//存在
		Node<K,V> e;//临时 存放key相同情况下的结点
         K k;
         //判断现有的hash和key与put的hash和key相等
		if (p.hash == hash &&((k = p.key) == key || (key != null && key.equals(k))))
			e = p;
		else if (p instanceof TreeNode)//红黑树的情况
			e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
		else {//链表结构的情况
			for (int binCount = 0; ; ++binCount) {//遍历链表
				if ((e = p.next) == null) {//如果遍历结束 没有发现相同的 则p.next=新的Node
					p.next = newNode(hash, key, value, null);
					if (binCount >= TREEIFY_THRESHOLD - 1) // 判断是否需要转变成红黑树
						treeifyBin(tab, hash);//转换
					break;
				}
                //如果发现存在key，则break，跳出循环
				if (e.hash == hash &&((k = e.key) == key || (key != null && key.equals(k))))
					break;
				p = e;
			}
		}
        //已存在的话 则直接替换value的值即可
		if (e != null) { // existing mapping for key
			V oldValue = e.value;
			if (!onlyIfAbsent || oldValue == null)
				e.value = value;
			afterNodeAccess(e);
			return oldValue;
		}
	}
	++modCount;
	if (++size > threshold)//判断是否需要扩容 默认为0.75*数组长度
		resize();//数组扩容
	afterNodeInsertion(evict);
	return null;
}
```

### 2.6 resize方法

* 源码

```java
final Node<K,V>[] resize() {
	Node<K,V>[] oldTab = table;//临时变量 存储数组table
	int oldCap = (oldTab == null) ? 0 : oldTab.length;
	int oldThr = threshold;
	int newCap, newThr = 0;
	if (oldCap > 0) {//数组长度 >0 表示数组有结点
		if (oldCap >= MAXIMUM_CAPACITY) {//数组大于阈值 2的30次方 时处理
			threshold = Integer.MAX_VALUE;
			return oldTab;
         //判断是否能够扩容 
		}else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&oldCap >= DEFAULT_INITIAL_CAPACITY)
			newThr = oldThr << 1; //改变数组最大数 0.75*数组长度*2
	}else if (oldThr > 0) // initial capacity was placed in threshold
		newCap = oldThr;
	else {//数组没有初始化 第一次进行初始化
		newCap = DEFAULT_INITIAL_CAPACITY;//16
		newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);//16*0.75
	}
    
	if (newThr == 0) {
		float ft = (float)newCap * loadFactor;
		newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
	}
    
	threshold = newThr;//修改threshold的值
	@SuppressWarnings({"rawtypes","unchecked"})
		Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];//初始化数组 或 创建扩容后的数组
	table = newTab;//修改table变量
	if (oldTab != null) {//处理数组扩容的情况
		for (int j = 0; j < oldCap; ++j) {//遍历旧数组的结点
			Node<K,V> e;
			if ((e = oldTab[j]) != null) {
				oldTab[j] = null;//旧数组结点置空
                 //在新数组中重新确定结点的位置 进行插入
				if (e.next == null)
					newTab[e.hash & (newCap - 1)] = e;
				else if (e instanceof TreeNode)
					((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
				else { // preserve order
					Node<K,V> loHead = null, loTail = null;
					Node<K,V> hiHead = null, hiTail = null;
					Node<K,V> next;
                    
					do {
						next = e.next;
						if ((e.hash & oldCap) == 0) {
							if (loTail == null)
								loHead = e;
							else
								loTail.next = e;
							loTail = e;
						}else {
							if (hiTail == null)
								hiHead = e;
							else
								hiTail.next = e;
							hiTail = e;
						}
					} while ((e = next) != null);
						if (loTail != null) {
							loTail.next = null;
							newTab[j] = loHead;
					}
                    
					if (hiTail != null) {
						hiTail.next = null;
						newTab[j + oldCap] = hiHead;
					}
				}
			}
		}
	}
	return newTab;
}
```

* 源码解释

```text
hashmap在扩容的时候，会创建新的数组，由于数组table.length的长度发生了改变，因此hash&(table.length-1)得到的值发生了改变。
由于二进制高位新增了一个1。如00001111变成00011111。要么保持原有index不变，要么位置index+原有数组长度。
因此扩容的时候hashmap存储元素的位置并不稳定。
```

### 2.7 get方法

* 源码

```java
public V get(Object key) {
	Node<K,V> e;
	return (e = getNode(hash(key), key)) == null ? null : e.value;
}

final Node<K,V> getNode(int hash, Object key) {
	Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
	if ((tab = table) != null && (n = tab.length) > 0 &&(first = tab[(n - 1) & hash]) != null) {
		if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
			return first;
		if ((e = first.next) != null) {
			if (first instanceof TreeNode)
				return ((TreeNode<K,V>)first).getTreeNode(hash, key);
			do {
				if (e.hash == hash &&
					((k = e.key) == key || (key != null && key.equals(k))))
					return e;
			} while ((e = e.next) != null);
		}
	}
	return null;
}
```

### 2.8 HashMap判断key是否相等

* if (p.hash == hash &&((k = p.key) == key || (key != null && key.equals(k))))

```text
由于源码中判断key值是否相同，既判断了hash的值，又判断了key.equals()方法.
所以想正确的获取HashMap的元素，判断key是否相同，要同时重写hashCode方法和equals方法
```

## 三、常见问题





