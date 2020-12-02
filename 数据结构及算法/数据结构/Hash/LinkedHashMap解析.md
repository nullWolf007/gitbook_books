## LinkedHashMap解析

### 一、按访问顺序和插入顺序排序

#### 1.1 LinkedHashMap

```java
public class LinkedHashMap<K,V> extends HashMap<K,V> implements Map<K,V>
```

* 可以看到LinkedHashMap继承自HashMap

#### 1.2 LinkedHashMap#构造函数

```java
    public LinkedHashMap(int initialCapacity,
                         float loadFactor,
                         boolean accessOrder) {
        super(initialCapacity, loadFactor);
        this.accessOrder = accessOrder;
    }
```

* 其中accessOrder表示访问顺序规则，当值为true，表示按照访问顺序排序；为false，表示按照插入顺序排序

  * 按照访问的次序来排序的含义：当调用LinkedHashMap的get(key)或者put(key, value)时，碰巧key在map中被包含，那么LinkedHashMap会将key对象的entry放在线性结构的最后。 

  * 按照插入顺序来排序的含义：调用get(key), 或者put(key, value)并不会对线性结构产生任何的影响

#### 1.3 LinkedHashMap#put

* 由于LinkedHashMap没有重写put方法，所以调用的是HashMap的put方法
* HashMap#put

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

* 重点是已存在那部分，对于HashMap而言是直接替换value值即可，但是LinkedHashMap重写了afterNodeAccess方法，那么我们重点就是afterNodeAccess方法

#### 1.4 LinkedHashMap#afterNodeAccess

```java
    void afterNodeAccess(Node<K,V> e) { // move node to last
        LinkedHashMap.Entry<K,V> last;
        if (accessOrder && (last = tail) != e) {
            LinkedHashMap.Entry<K,V> p =
                (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
            //分析1
            p.after = null;
            //分析2
            if (b == null)
                head = a;
            else
                b.after = a;
            //分析3
            if (a != null)
                a.before = b;
            else
                last = b;
            //分析4
            if (last == null)
                head = p;
            else {
                p.before = last;
                last.after = p;
            }
            tail = p;
            ++modCount;
        }
    }
```

* head表示头节点；tail表示尾节点
* last为tail为尾节点，所以如果last为空则说明这是一个空的，直接把p放到head就行
* 分析1：由于p节点需要放到最后，所以p的after节点为null
* 分析2：如果b为null，说明p为头节点，则把a表示后面的移到头部；如果b不为null，则需要把p节点移到最后，所以先把p去除，调用b.after = a使其去除p并连接起来
* 分析3：理论上a不为空，因为p节点不是尾节点，所以a理论上不为空，所以调用a.before = b使其b，a双向连接起来
* 分析4：如果last为null，则说明这是一个空的，直接把p赋值给head即可；如果last不为空，其代表的尾节点，由于上述步骤把p节点已经移除干净了，所以只需把p节点加在尾节点后面即可