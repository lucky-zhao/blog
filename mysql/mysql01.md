# mysql

本文主要是对mysql数据库的一些学习记录和小结。之所以选择mysql，主要还是工作中遇到的比较多，自己撸点小项目mysql也是首选。

- [mysql](#一-mysql)
    - [mysql架构图](#11-mysql架构图)
    - [常用的存储引擎](#12-常用的存储引擎)
    - [七种join](#13-七种join)
    - [索引](#14-索引)
        - [索引分类](#141-索引分类)
        - [索引的数据结构是什么](#142-索引的数据结构是什么)
    
    
## 1.1 mysql架构图
## 1.2 常用的存储引擎
mysql一般常见的存储引擎是：InnoDB、MyISAM。这两种引擎有什么特别和区别呢？

| 对比项 | MyISAM | InnoDB |
| :------| :------ | :------ |
| 事务 | 不支持 | 支持 |
| 外键 | 不支持 | 支持 |
| 行表锁 | 表锁，即使操作一条记录也会锁住整个表 | 行锁，操作时只锁某一行 |
| 表空间 | 小 | 大 |
| 缓存 | 只缓存索引 | 不光缓存索引，还缓存真实数据，对内存要求较高 |
| 关注点 | 性能 | 事务 |

还有一些区别，后面写例子的时候会记录；怎么选择用哪个引擎呢？我个人觉得主要还是看事务，如果你要支持事务，那就必须InnoDB。注意一点，存储引擎是对表而言的，不是对数据库的操作。

## 1.3 七种join
建了两张表：用户表、部门表，并且插入几条数据。
```sql
CREATE TABLE `sys_dept` (
  `d_id` int(11) NOT NULL AUTO_INCREMENT,
  `dept_name` varchar(30) NOT NULL,
  PRIMARY KEY (`d_id`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `sys_user` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(30) NOT NULL,
  `dept_id` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

insert into sys_dept (`dept_name`) VALUES ('技术部');
insert into sys_dept (`dept_name`) VALUES ('行政部');
insert into sys_dept (`dept_name`) VALUES ('销售部');
insert into sys_user (`name`,`dept_id`) VALUES ('张三',1);
insert into sys_user (`name`,`dept_id`) VALUES ('李四',1);
insert into sys_user (`name`,`dept_id`) VALUES ('王五',2);
insert into sys_user (`name`,`dept_id`) VALUES ('赵六',88);
```
为了演示效果上面插入了一条`insert into sys_user (`name`,`dept_id`) VALUES ('赵六',88)`部门ID为88的 其实是没有这个部门的。先看第一种join
* inner join(内连接)
```sql
select * from sys_user su inner join sys_dept sd on su.dept_id=sd.id
```
结果：

| id | name | dept_id| d_id | dept_name |
| :------| :------ | :------ |:------ |:------ |
| 1 | 张三 | 1 | 1 | 技术部 |
| 2 | 李四 | 1 | 1 | 技术部 |
| 3 | 王五 | 2 | 2 | 行政部 |

可以看到`inner join`取的是两个表中都存在的数据,和`join`是相同的效果。

* left join 或者 left outer join(左外连接)
查询所有user以及他们所在的部门-如果有的话
```sql
select * from sys_user su left join sys_dept sd on su.dept_id=sd.d_id 
```
结果：

| id | name | dept_id| d_id | dept_name |
| :------| :------ | :------ |:------ |:------ |
| 1 | 张三 | 1 | 1 | 技术部 |
| 2 | 李四 | 1 | 1 | 技术部 |
| 3 | 王五 | 2 | 2 | 行政部 |
| 4 | 赵六 | 88 |  |  |

发现把用户 赵六也查询出来了，但是他所对应的部门为空，没有ID为88的部门，查询为null。左外连接取左表所有的行，如果左表在右表中没有匹配行也会查询出来，结果都是空值。

* right join 或者 right outer join(右外连接)
```sql
select * from sys_user su right join sys_dept sd on su.dept_id=sd.d_id
```
结果：

| id | name | dept_id| d_id | dept_name |
| :------| :------ | :------ |:------ |:------ |
| 1 | 张三 | 1 | 1 | 技术部 |
| 2 | 李四 | 1 | 1 | 技术部 |
| 3 | 王五 | 2 | 2 | 行政部 |
|  |  |  | 3 | 销售部 |

结果和`left join`相反，取的是右表的所有行。如果右表的某行在左表中没有匹配行，则将为左表返回空值。

* full join(全连接)
不好意思mysql没有full join，但是可以用其他方式实现，full join返回的是两个表所有的行，当某行在另一个表中没有匹配行时，则另一个表的选择列表列包含空值。那不就是把left join和right join联合起来使用吗？
```sql
select * from sys_user su left join sys_dept sd on su.dept_id=sd.d_id 
UNION
select * from sys_user su right join sys_dept sd on su.dept_id=sd.d_id 
```
结果：

| id | name | dept_id| d_id | dept_name |
| :------| :------ | :------ |:------ |:------ |
| 1 | 张三 | 1 | 1 | 技术部 |
| 2 | 李四 | 1 | 1 | 技术部 |
| 3 | 王五 | 2 | 2 | 行政部 |
| 4 | 赵六 | 88 |  |  |
|  |  |  | 3 | 销售部 |

结果就是两个表的所有行，当某行在另一个表中没有匹配行时，则另一个表的选择列表列包含空值。

* 左连接(左独有连接)
首先看一下上面的左外连接的结果集，是左表所有行加上右表匹配行，如果有不嘛匹配的就显示空值。而左连接(左独有连接)就是左表的行在右表中没有匹配的。
```sql
select * from sys_user su left join sys_dept sd on su.dept_id=sd.d_id where sd.d_id is null
```

| id | name | dept_id| d_id | dept_name |
| :------| :------ | :------ |:------ |:------ |
| 4 | 赵六 | 88 |  |  |

查询出来只有一个赵六，因为只有他的部门ID在部门表中没有。

* 右连接(右独有连接)
```sql
select * from sys_user su right join sys_dept sd on su.dept_id=sd.d_id  where su.dept_id is NULL
```
结果：

| id | name | dept_id| d_id | dept_name |
| :------| :------ | :------ |:------ |:------ |
|  |  |  | 3 | 销售部 |

和上面的相反。结果就是 右表中的销售部 在左表中没有匹配项的被查出来。

* 全连接去交集(取两张表中都没有匹配的数据集)
```sql
select * from sys_user su left join sys_dept sd on su.dept_id=sd.d_id where sd.d_id is null
UNION
select * from sys_user su right join sys_dept sd on su.dept_id=sd.d_id  where su.dept_id is NULL
```
结果:

| id | name | dept_id| d_id | dept_name |
| :------| :------ | :------ |:------ |:------ |
| 4 | 赵六 | 88 |  |  |
|  |  |  | 3 | 销售部 |

## 1.4 索引
索引是一种帮助mysql高效的获取数据的数据结构。可以简单的理解成排序好的快速查找数据结构。
* 索引的优点：1.天生排序。2.快速查找
* 索引的缺点：1.占用空间。2.降低更新表速度
### 1.4.1 索引分类
* 普通索引:最基本的索引，没有什么约束；
* 唯一索引：和普通索引类型，但是有唯一性约束；
* 主键索引：特殊的索引，不允许有空值；
* 复合索引：将多个列组成的一个索引；
* 外键索引：只有InnoDB引擎的表才可以使用外键索引，保证数据的一致性、完整性；
* 全文索引：mysql自带的全文索引只能用于InnoDB、MyISAM，并且只能对英文进行全文检索，一般使用全文索引引擎(ES，Solr)；
注意：主键就是唯一索引，但是唯一索引不一定是主键，唯一索引可以为空，但是空值只能有一个，主键不能为空。

### 1.4.2 索引的数据结构是什么？
mysql的索引使用了B+Tree结构。为什么不使用其他的数据结构呢？
* 使用二叉树的缺陷

在一些情况下会出现结构偏移，极端情况下会形成一条链表的形式。
![二叉树缺陷](https://github.com/lucky-zhao/blog/blob/master/mysql/img/binary_tree.png "二叉树缺陷")

* 使用平衡二叉树缺陷

上面说的二叉树的缺陷，是可能形成链表，那么使用平衡二叉树不就行了吗？先看下平衡二叉树的定义：在二叉树的基础上，保证每个节点左子树和右子树的高度差小于等于1。那么mysql为什么不用平衡二叉树这样的数据结构呢？

1. 维护这种高度平衡所付出的代价比从中获取的收益要大，简单地说就是弊大于利，在每次插入、删除元素的时候会通过一次或者多次的左旋或者右旋来保证树的平衡。所以实际应用不多，更多的是追求局部平衡的红黑树结构；
2. 查询效率不高，一般来说数的高度和IO次数是成正比的，当数据量很大的时候，树的高度就会很恐怖，导致查询性能降低；
3. 节点存储的数据内容太少。没有很好利用操作系统和磁盘数据交换特性，也没有利用好磁盘IO的预读能力。因为操作系统和磁盘之间一次数据交换是已页为单位的，一页 = 4K，即每次IO操作系统会将4K数据加载进内存。但是，在二叉树每个节点的结构只保存一个关键字，一个数据区，两个子节点的引用，并不能够填满4K的内容。幸幸苦苦做了一次的IO操作，却只加载了一个关键字，在树的高度很高，恰好又搜索的关键字位于叶子节点或者支节点的时候，取一个关键字要做很多次的IO。


* 使用红黑树的缺陷

红黑树和平衡二叉树很类似，都是在进行插入或者删除操作的时候通过特定操作保证树的平衡，从而获得比较高的查询性能，但是红黑树在最坏的情况下运行也是比较良好的。但是红黑树是二叉树，也就是一个节点下有两个分支，而B树可以有多个叶子节点，所以在大规模数据存储的时候红黑树的高度会更高一点，从而造成磁盘IO读写次数过多，导致索引效率低下。

定义：
1. 节点是红色或者黑色；
2. 根节点是黑色；
3. 每个红色节点的两个子节点都是黑色。(从每个叶子到根的所有路径上不能有两个连续的红色节点)；
4. 从任一节点到其每个叶子的所有路径都包含相同数目的黑色节点。

第三条定义规则约束了红黑树从根到叶子的最长的可能路径不多于最短的可能路径的两倍长。结果是这个树大致上是平衡的。因为操作比如插入、删除和查找某个值的最坏情况时间都要求与树的高度成比例，这个在高度上的理论上限允许红黑树在最坏情况下都是高效的。根据定义4所有最长的路径都有相同数目的黑色节点，这就表明了没有路径能多于任何其他路径的两倍长。
虽然红黑树相对于二叉树而言不会有太严重的单边偏移(形成链表)，也不会像平衡二叉树一样需要保证高度的平衡，但是避免不了极端情况下树的重心出现偏移的情况。而且mysql是基于磁盘的数据库，索引是以索引文件的形式存在于磁盘中，索引的查找过程就会涉及到磁盘IO消耗，磁盘IO的消耗相比较于内存IO的消耗要高好几个数量级，所以索引的组织结构要设计得在查找关键字时要尽量减少磁盘IO的次数。实际过程中，磁盘并不是每次严格按需读取，而是每次都会预读。磁盘读取完需要的数据后，会按顺序再多读一部分数据到内存中，这样做的理论依据是计算机科学中注明的局部性原理：`当一个数据被用到时，其附近的数据也通常会马上被使用`，

* B-Tree('-'不是读 减，多路平衡树)

BTree是一种多路自平衡搜索树，和二叉树类型，但是BTree允许每个节点有更多的叶子节点。BTree有个特点:不管是叶子节点还是非叶子节点都有个Data域，用来保存实际的数据。BTree广泛的应用在文件存储系统以及数据库系统中，
https://www.jianshu.com/p/f635a6a94254

缺点：
1. 

* B+Tree

B+Tree和B-Tree非常相似，区别在于：
1. B+Tree的所有根节点都不存储实际数据，只有索引信息，所有的数据都存在叶子节点里
2. B+Tree所有的叶子结点和相邻的节点使用链表相连，便于区间查找和遍历。
3. B+Tree还有一个相应的优质特性，就是B+Tree的查询效率是非常稳定的，因为所有信息都存储在了叶子节点里面，从根节点到所有叶子节点的路径是相同的。