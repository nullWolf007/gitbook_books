[TOC]

## left join、right join和join的区别

#### 转载

* [【mySQL】left join、right join和join的区别](https://segmentfault.com/a/1190000017369618)

* 哈，好久没更新文章了，今天来说说关于mySQL那些年的小事。说到mySQL啊，用了挺久的了，但是有个问题一直在困扰着我，就是left join、join、right join和inner join等等各种join的区别。网上搜，最常见的就是一张图解图，如下：

![clipboard.png](https://segmentfault.com/img/bVbk2mR?w=966&h=760)

* 真的是一张图道清所有join的区别啊，可惜我还是看不懂，可能人比较懒，然后基本一个left join给我就是够用的了，所以就没怎么去仔细研究了，但是现实还是逼我去搞清楚，索性自己动手，总算理解图中的含义了，下面就听我一一道来。

* 首先，我们先来建两张表，第一张表命名为kemu，第二张表命名为score：

![clipboard.png](https://segmentfault.com/img/bVbk2or?w=118&h=92)![clipboard.png](https://segmentfault.com/img/bVbk2oz?w=128&h=94)

### 一、left join

* 顾名思义，就是“左连接”，表1左连接表2，以左为主，表示以表1为主，关联上表2的数据，查出来的结果显示左边的所有数据，然后右边显示的是和左边有交集部分的数据。如下：

```mysql
select
   *
from
   kemu
left join score on kemu.id = score.id
```

* 结果集：
  ![clipboard.png](https://segmentfault.com/img/bVbk2uE?w=205&h=144)![clipboard.png](https://segmentfault.com/img/bVbk2qQ?w=238&h=103)

### 二、right join

* “右连接”，表1右连接表2，以右为主，表示以表2为主，关联查询表1的数据，查出表2所有数据以及表1和表2有交集的数据，如下：

```mysql
select
   *
from
   kemu
right join score on kemu.id = score.id
```

* 结果集：

![clipboard.png](https://segmentfault.com/img/bVbk2uI?w=222&h=143)![clipboard.png](https://segmentfault.com/img/bVbk2uP?w=228&h=104)

### 三、join

* join，其实就是“inner join”，为了简写才写成join，两个是表示一个的，内连接，表示以两个表的交集为主，查出来是两个表有交集的部分，其余没有关联就不额外显示出来，这个用的情况也是挺多的，如下

```mysql
select
   *
from
   kemu
join score on kemu.id = score.id
```

* 结果集：

![clipboard.png](https://segmentfault.com/img/bVbk2v1?w=227&h=145)![clipboard.png](https://segmentfault.com/img/bVbk2MW?w=231&h=69)

* 以上就是三种连接的区别！