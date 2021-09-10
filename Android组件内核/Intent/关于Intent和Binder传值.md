# 关于Bundle和Intent传值

## 比较

Intent可以使用putExtra("key",value)进行传值，为什么还需要Bundle呢，其实你查看putExtra的源代码，你就会发现里面也创建了Bundle对象，所以Intent的传值是在Bundle的基础上实现的;Bundle是为了存储数据的，Intent是使用了bundle来传递数据的，Bundle相对于而言，底层一点，但是更加灵活。

```java
public Intent putExtra(String name, boolean value){ 
  if (mExtras == null) {
    mExtras = new Bundle(); 
    } 
    mExtras.putBoolean(name, value); 
    return this; 
}
```

Bundle是可以对对象进行操作的，而Intent不可以,Bundle相对于Intent更加灵活，有的时候更加方便

## 例子

**从A界面跳转到B界面或者C界面**

- 就需要写2个Intent,如果要涉及传值的话,你的Intent就要写两遍添加值的方法。如果我用1个Bundle,先存值，然后再存到Intent中,就更加简洁

**把值通过Activity A经过Activity B传给Activity C**

- 对于Intent的话，A-B一遍，再在B中都取出来 然后在把值塞到Intent中，再跳到C。如果在A中用了 Bundle 的话，把Bundle传给B，在B中再转传到C，C就可以直接取了。

## Bundle的使用场景

- 在设备旋转时保存数据
- Fragment之间传递数据