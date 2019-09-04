# mysql

本文主要是对mysql数据库的一些学习记录和小结。之所以选择mysql，主要还是工作中遇到的比较多，自己撸点小项目mysql也是首选。

- [mysql](#一-mysql)
    - [mysql架构图](#11-mysql架构图)
    - [常用的存储引擎](#12-常用的存储引擎)
    - [七种join](#13-七种join)
    - [索引](#14-索引)
        - [索引分类](#141-索引分类)
    
    
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

### 1.4.1 索引分类