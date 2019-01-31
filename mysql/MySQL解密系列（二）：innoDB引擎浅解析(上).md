![](https://img.hacpai.com/bing/20180325.jpg?imageView2/1/w/960/h/520/interlace/1/q/100)

版本 | 时间 | 简介
---|--- |---
V1.0 | 2018/12/14 | 基本完成innoDB内存中部分的介绍


## 前言
在这两篇文章里面，我希望可以和你讨论一下InnoDB存储引擎的架构。本篇文章主要依赖于MySQL官方文档以及姜承尧的《innoDB存储引擎》来完成，很多方面由于我的水平有限，暂时不能够深入到足够的底层，希望在后期的学习中能够不断的完善这篇文章的内容。也欢迎读者们指出文章中的不足和给出建议。

![InnoDB Architecture](https://dev.mysql.com/doc/refman/5.7/en/images/innodb-architecture.png)
上面的图来自于MySQL的官方手册，描述了InnoDB引擎的两大组成部分——`In-Memory Structures`和`On-Disk Structures`,即内存中组件和磁盘中组件，在下面的文章中，我将与读者们讨论`In-Memory Structures`中每一个部分的组成和作用。

首先要介绍的部分就是内存中各个部件的整体结构，主要有三大部分组成，它们分别是缓冲池（buffer pool）,日志缓冲区（log buffer）和额外内存池，其中缓冲池又包含着更改缓冲区（change buffer）、数据页（data page）缓冲、索引页（index page）缓冲等多个部分。
![imagepng](http://qiniuyun.indispensable.cn//file/2018/12/fe7c9900537a499aa40d93422e3a3239_image.png)


## 存储引擎

### 缓冲池（Buffer Pool）
相信读者们也都知道，innoDB存储引擎是基于磁盘进行数据存储的。并根据页对其中包含的记录进行管理，因此可以把它视为基于磁盘的数据库系统（Disk-base Database）。由于磁盘速度和CPU的速度相差甚远，甚至有“磁盘一秒，CPU一年”的说法，所以基于磁盘的数据库系统一般使用`缓冲池`技术来提高数据库的整体性能。
简单来说，缓冲池一般就是一块内存区域，使用内存缓冲数据以弥补磁盘速度较慢对数据库性能的影响。在数据库进行读取也得操作时，首先判断该页是否存在于缓冲池中，如果存在于缓冲池中，就直接读取缓存池中的数据返回，否则就从磁盘上读取对应的数据页放入缓冲池中，并将对应的数据返回。
对于数据库中数据页的修改操作，首先修改数据在缓冲池中的数据页，然后再根据一定的频率刷新到磁盘上去，这里需要注意的地方是--数据页从缓冲池刷新回磁盘的操作并不是在每次页发生更新时触发的，而是通过一种称为`checkPoint`的机制刷新回磁盘。
![imagepng](http://qiniuyun.indispensable.cn//file/2018/12/ccb62175eabd49c883be1a0d78aa2759_image.png)


具体来说，缓冲池中缓存的数据页的类型有：索引页、数据页、undo页、更改缓存、自适应哈希索引、InnoDB存储的锁信息、数据字典信息等，其中数据页和索引页的缓存占据了缓存池中很大的一部分。在专门用于运行MySQL的服务器上面，一般会将60%-80%的内存分配给缓存池。

可以通过如下命令查看数据库缓冲池大小，由输出可以知道我本地MySQL的innoDB缓冲池大小为128M。
```
mysql> show variables like 'innodb_buffer_pool_size'\G;
*************************** 1. row ***************************
Variable_name: innodb_buffer_pool_size
        Value: 134217728
1 row in set (0.02 sec)
```

### 缓冲池所使用的LRU算法

从上面的文章里面，我们了解到了缓冲池是一个很大的内存区域，里面存放着各种各样的页，那么innoDB对于这些页到底是怎样进行管理的呢？这也是我在这篇文章中想和读者朋友们进行讨论的，了解如何利用缓冲池将频繁访问的数据保存在内存中是MySQL调优的一个重要方面。

innoDB引擎中的缓冲池是通过LRU(Latest Recent Uses,最近最少使用)算法来进行管理的，将最近最频繁使用到的数据页放在LRU列表的前端，将最近最少使用到的数据页放在LRU列表的末端，这些LRU列表末端的数据页，当缓冲池剩余空间不足时，将首先被移除。

![imagepng](http://qiniuyun.indispensable.cn//file/2018/12/a1ed62286ae64cbfb24ec67e75270b63_image.png)
默认情况下，innoDB中的LRU算法是这样执行的：

-  3/8的缓冲区域作为为旧子列表（Old Sublist）。
-  列表的中间位置是新子列表(new Sublist)的尾部(tail)与旧子列表(Old Sublist)的头部(head)相交的边界。
-  当InnoDB将页面读入缓冲池时，它最初将其插入列表的中间位置（旧子列表的头部）。可这些页面中的数据可以被读取，因为它是用户指定的操作（例如SQL查询）所需要的，或者是InnoDB自动执行的预读操作的一部分。
- 访问旧子列表中的页面使其被移动到缓冲池的头部（新子列表的头部），这个操作被称为 `page made young`。如果因为查询某数据而读入对应页面，那么会立即进行第一次访问，并使页面被移动到缓冲池的头部。如果是由于预读而读入了页面，则第一次访问不会立即发生（并且在页面被移除出缓冲区之前可能根本不会发生）。
- 当数据库运行时，缓冲池中的页面由于不会被访问而渐渐的被移到旧子列表的尾部。新旧子列表中的页面随着其他页面的变化而发生变化。旧子列表中的页面也会随着页面插入缓冲区中间位置而老化。最终，长期未使用的页面到达旧子列表的尾部并被从缓冲区中移除。

默认情况下，查询语句读取的页面会立即移动到新的子列表中，这意味着它们会更长时间地保留在缓冲池中。表扫描的操作（例如没有WHERE子句的SELECT语句）可以将大量数据加载到缓冲池中并从缓冲池中移除相同数量的旧数据，即使新数据从未再次使用过。类似地，由预读后台线程加载然后仅访问一次的页面移动到新列表的头部。这样的情况会将经常使用的页面移动到旧的子列表中，在那里它们会被移除出缓存区。如果读者朋友想要了解这方面的相关优化，可以阅读[Making the Buffer Pool Scan Resistant](https://dev.mysql.com/doc/refman/5.7/en/innodb-performance-midpoint_insertion.html)。

### 更改缓冲区（Change Buffer）

  change Buffer的主要目的是将对二级索引的数据操作缓存下来，以此减少二级索引的随机IO，并达到操作合并的效果。在MySQL5.5之前的版本中，由于只支持缓存insert操作，所以最初叫做insert buffer，只是后来的版本中支持了更多的操作类型缓存，才改叫change buffer。change buffer的物理存储方式上是一颗的b+树，存储在ibdata系统表空间中，根页为ibdata的第4个page(`FSP_IBUF_TREE_ROOT_PAGE_NO`)。
  在innoDB存储引擎中，主键是唯一的标识符。通常行记录的插入是根据主键递增的顺序来进行的，因此插入聚集索引（Primary Key）一般是顺序写入，不需要进行磁盘中另一个页进行随机读取，这种情况下的操作速度是非常快的。但是不可能每个表都只有一个聚集索引，比如说这里我们有这样的一张表：
  ```
  create table user(
	id int auto_increment,
	name varchar(50),
	age int,
	primary key(id),
	key(name)
  );

  ```
  由于name并非主键字段同时也可能不唯一，所以在这张表中就会产生一个非聚集的且不是唯一的索引（正常情况下我们说的辅助索引大部分都属于这种类型）。在对`user`表进行插入操作的时候，数据页的存放还是根据主键id来进行顺序存放的，但是对于字段`name`的索引，插入就不再是顺序的了，这个时候需要访问离散的访问`name`字段的索引页。由于要进行随机读取会对性能产生较大的影响。而change buffer 的出现大大缓解了这种情况导致的性能下降问题。比如说，对于非聚集索引的插入和更新操作并非是每一次都插入到索引页中去，而是先判断这条数据对应的非聚集索引页是否存在于change buffer的insert buffer中。如果存在就插入到insert buffer中的索引页，否则就先放入到一个insert buffer对象中去。然后在一个适当的时间将insert buffer 和对应的非聚集索引的子节点进行合并（merge）操作，这个时候一般可以将多个插入合并到一个操作中去（因为在一个索引页中），这样就大大提高了非聚集索引插入的性能。
  change buffer 的使用必须要满足以下两个条件：

  1. 索引是辅助索引（secondary index）；
  2. 索引不是唯一的。

  索引必须是辅助索引，也就是非聚集索引的原因是因为主键索引一般是顺序插入，也就无需使用change buffer来进行相关存储了。
  索引不是唯一的原因是，如果索引是唯一的话，那么在对数据进行相关操作时，还需要通过查找索引页来判断插入记录的唯一性，这样就会涉及到离散读取的现象，从而无法充分利用change buffer来提高性能。
  需要注意的是，当change buffer缓冲区中存在了大量的change buffer对象，数据库系统宕机了，这时一点会有大量的change buffer 对象没有合并（merge)到对应的非聚集索引中去，这个时候进行的数据库数据恢复可能需要很长的时间。出于以上原因以及不愿意change buffer对象占用过多的缓冲池空间，所以change buffer最大默认值为缓冲池大小的25%，当超过25%时，可能触发用户线程同步缩减change buffer (由于历史原因实际名为ibuf) btree。

一条change buffer 记录大概包含以下几列：
![](http://mysql.taobao.org/monthly/pic/2015-07-01/ibuf-record.png)
在这里对其内部结构不做过多介绍，感兴趣的读者可以阅读[淘宝数据库内核月报-2015/07：Innodb change buffer介绍](http://mysql.taobao.org/monthly/2015/07/01/)这篇文章。

### 自适应哈希索引
  哈希表是一种查找非常迅速的数据结构，在一般情况下查找的时间复杂度为O(1)，而B+树的查找次数取决于B+树的高度，一般情况下生产环境的MySQL，其B+树的高度为3-4层，故一次查找一般需要进行3-4次的读取。InnoDB存储引擎会对表上各个索引页的查询进行监控。如果发现某些索引页被多次的使用，通过建立哈希索引可以提升查询速度，那么就会建立哈希索引，称之为自适应哈希索引(Adaptive Hash Index,AHI)，AHI是根据缓冲池中的索引页（B+树页）来构建的，所以创建速度会很快，而且也并不需要为某张表的所有数据建立哈希索引。`InnoDB 引擎会根据访问的频率和访问的方式来自动地为某些热点数据建立哈希索引`。
  但是并非所有的热点数据都会被建立哈希索引页，建立自适应哈希索引页（AHI）的索引页，对于这个页的连续访问模式必须是一样的，`访问模式指的是查询的条件一样`。比如我们这里有一个多值索引（a,b）,对于其缓冲区中的索引页的访问方式有以下两张方式：
  1. where a = XXX
  2. where a = XXX and b = XX
如果交替的执行上述两张查询，那么InnoDB存储引擎并不会为该页创建自适应哈希索引。

可以在MySQL上通过命令`show engine innodb status`查看当前change buffer和自适应哈希索引的使用情况。(注：在前文中已经说过change buffer是由insert buffer扩充而来，所以在数据库中很多地方包括源码中都使用`ibuf`来表示change buffer对象)。
```
mysql> show engine innodb status\G;
*************************** 1. row ***************************
  Type: InnoDB
  Name:
Status:
	......
-------------------------------------
INSERT BUFFER AND ADAPTIVE HASH INDEX
-------------------------------------
Ibuf: size 1, free list len 0, seg size 2, 0 merges
merged operations:
 insert 0, delete mark 0, delete 0
discarded operations:
 insert 0, delete mark 0, delete 0
Hash table size 276707, node heap has 0 buffer(s)
0.00 hash searches/s, 0.00 non-hash searches/s


```
## 日志缓冲区（Log Buffer）
日志缓冲区是用来保存要写入磁盘上日志文件的数据的内存区域，这里的日志缓冲区实际上存储的仅仅是重做日志（redo log）,撤销日志（undo log）记录在缓冲池中的一个非常小的内存空间。当对MySQL中存储引擎为innoDB的表进行修改时，InnoDB存储引擎会将重做日志信息先放入到这个缓冲区中，然后再按照一定的频率将这些重做日志信息刷新到磁盘上的重做日志文件中。 日志缓冲区大小由`innodb_log_buffer_size`变量定义。 默认大小为`16MB`。(官方5.7的版本中说默认为16MB,但我在Windows下的本地新安装的5.7MySQL中，查看该变量值为`8MB`,找了一下相关资料发现如果版本大于等于`5.7.6`，那么日志缓冲区的默认大小才为`16MB`) 。
```
mysql> show variables like 'innodb_log_buffer_size'\G;
*************************** 1. row ***************************
Variable_name: innodb_log_buffer_size
        Value: 8388608
1 row in set (0.01 sec)

```
在一般情况下，默认的日志缓冲区大小足够满足绝大部分的应用，因为重做日志在下列三种情况下会将日志缓冲区中的内容刷新到磁盘中的重做日志文件中去。
1. Master Thread 每一秒都会将重做日志缓冲刷新到重做日志文件中去。
2. 每个事务提交时都会将重做日志缓冲刷新到重做日志文件中去。
3. 当重做日志缓冲池剩余空间小于1/2时，会将重做日志缓冲刷新到重做日志文件中去。

较大的日志缓冲区使大型事务能够快速的运行，而不需要在事务被提交之前将重做日志数据写入磁盘。 因此，如果读者有更新，插入或删除许多行的事务，那么这个时候增加日志缓冲区的大小可以节省磁盘I/O，这不失为一个明智的选择。这里有两个相关的参数，他们分别是：

`innodb_flush_log_at_trx_commit`变量控制如何写入日志缓冲区的内容并刷新到磁盘。 `innodb_flush_log_at_timeout`变量控制日志刷新频率。


## 尾声
写到这里，这篇关于innoDB引擎的内存中几大组件就基本完成了，希望在接下来的时间里，我能够仔细优化这篇文章。


## 参考
* [XtraDB / InnoDB internals in drawing](https://www.percona.com/blog/2010/04/26/xtradb-innodb-internals-in-drawing/)
* [InnoDB Architecture From MySQL 5.7 Reference Manual](https://dev.mysql.com/doc/refman/5.7/en/innodb-architecture.html)
* [InnoDB From Wikipedia](https://en.wikipedia.org/wiki/InnoDB)
* [淘宝数据库内核月报-2015/07：Innodb change buffer介绍](http://mysql.taobao.org/monthly/2015/07/01/)
* [ Analyzing and Optimizing the MySQL InnoDB Log Buffer and InnoDB Redo Log](https://www.virtual-dba.com/analyzing-optimizing-mysql-innodb-buffer-innodb-redo-log/)

