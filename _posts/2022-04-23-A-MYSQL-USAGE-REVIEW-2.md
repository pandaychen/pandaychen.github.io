---
layout: post
title: Mysql 项目应用笔记（二）
subtitle: Mysql 索引与优化总结 - InnoDB 篇
date: 2022-04-23
author: pandaychen
catalog: true
tags:
  - MySQL
  - InnoDB
---

##  0x00    前言
本篇介绍下 InnoDB 引擎的索引使用及优化。重点关注几个问题：
- 索引的原理
- 如何建索引
- 如何使用索引
- 如何调优？

####  索引的本质
索引就像一本书的目录。而当用户通过索引查找数据时，就好比用户通过目录查询某章节的某个知识点。这样就帮助用户有效地提高了查找速度。所以，使用索引可以有效地提高数据库系统的整体性能。

####  索引分类
- 从用户使用的角度：单列索引（主键索引 / 唯一索引 / 普通索引) 与组合索引，用户的设置与 `SELECT` 会受此影响
- 从磁盘存储结构：聚簇索引和非聚簇索引
- 从索引的数据结构：`B+` 树索引 / Hash 索引 / 倒排索引


#### 聚簇索引 VS 非聚簇索引
1、聚簇索引 <br>
一张表只有一个聚簇索引，数据库会以聚簇索引来构造一个 `B+` 树 (即聚簇索引一定也是 `B+` 树索引)，叶子节点即表的行数据。默认的聚簇索引为主键索引。表内行数据按照聚簇索引的顺序在磁盘存放。建议 InnoDB 表使用默认自增 id 做主键（相较于 `uuid` 这类字符串做主键，由于主键使用了聚簇索引，如果主键是自增 id，那么对应的数据一定也是相邻地存放在磁盘上的，写入性能比较高。如果是 `uuid` 的形式，频繁的插入会使 InnoDB 频繁地移动磁盘块，写入性能相对较低）

主键的生成规则：
1.  没有主键时，会用一个唯一且不为空的索引列做为主键，成为此表的聚簇索引
2.  若无此类索引，InnoDB 会隐式定义一个主键来作为聚簇索引

此外，直接 `SELECT *` 就是按照聚簇索引的顺序展示数据。

2、非聚簇索引 <br>
创建的索引，如复合索引、前缀索引、唯一索引，都是属于非聚簇索引，底层有两种数据结构：`B+` 树和 Hash 表。`B+` 树则是在叶子节点存放对应行数据的聚簇索引和该树对应的索引值。

##  0x01  索引查询的基础

####  简单的例子
以 [此文](https://www.cnblogs.com/rjzheng/p/9915754.html) 的例子：

| pId | name | birthday |
| :-----:| :----: | :----: |
| 5	|zhangsan	|2016-10-02 |
| 8	|lisi	|2015-10-04 |
|11	|wangwu	|2016-09-02|
|13	|zhaoliu|	2015-10-07|

存储结构如下图所示（上半部分是由主键形成的 `B+` 树，下半部分就是磁盘上存储的数据）：
![index1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2022/mysql/innodb_index01.png)


####  查询：聚簇索引
```sql
select * from table where pId='11'
```
对应的查询过程为，从根开始，经过 `3` 次查找即完成。若不使用索引，那就要在磁盘上，进行逐行扫描，直到找到数据位置。显然，使用索引速度会快。但是在插入数据的时候，需要维护这颗 `B+` 树的结构，因此写入性能会下降。
![index2](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2022/mysql/innodb_index02.png)

####  查询：非聚簇索引
将 `name` 列建立 index，innodb 根据索引字段 `name` 生成一颗新的 `B+` 树（增加了表空间）。非聚簇索引的叶子节点并非真实数据，它的叶子节点依然是索引节点，存放的是该索引字段的值以及对应的主键索引（聚簇索引）。
```sql
create index index_name on table(name);
```
存储结构变成下面这样：
![index1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2022/mysql/innodb_index03.png)

如果使用 `name` 字段做查询，查询过程变成先从非聚簇索引树开始查找，然后找到聚簇索引后。根据聚簇索引，在聚簇索引的 `B+` 树上，找到完整的数据：
```sql
select * from table where name='lisi'
```
![index1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2022/mysql/innodb_index04.png)

####  查询：非聚簇索引的特殊情况
如果在非聚簇索引树上找到了想要的值，就不会去聚簇索引树上查询（常见的优化场景就是尽量用 `SELECT col` 代替 `SELECT *`）。如下面的查询：
```sql
select name from table where name='lisi'
```
![index1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2022/mysql/innodb_index05.png)


####  索引：按需添加
增加一个索引，就会多生成一颗非聚簇索引树。在做插入操作的时候，需要同时维护这几颗树的变化，因此，如果索引太多，插入性能就会下降。因此，索引并非越多越好，如下面增加多一个索引之后的视图：
```sql
create index index_birthday on table(birthday);
```
![index1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2022/mysql/innodb_index06.png)


##  0x02  联合索引

####  最左匹配原则（重点）
最左匹配原则指的 SQL 中用到了联合索引中的最左边的索引（相对顺序），那么这条 SQL 语句就可以利用这个联合索引去进行匹配，当遇到范围查询（`>`/`<`/`between`/`like`）就会停止匹配。

举例来说，`(a,b)` 字段为联合索引，`WHERE` 后条件有如下几种 case：
- `a = 1` 或 `a = 1 AND b = 2`、`b = 2 AND a =1`：可以匹配索引
- `b = 2`：不能匹配

又如：`(a,b,c,d)` 字段的联合索引，当 `WHERE` 为 `a = 1 AND b = 2 AND c > 3 AND d = 4` 时，则只有 `a/b/c` 字段能用到索引，而 `d` 就匹配不到（原因是遇到了范围查询 `c>3` 停止）

####  最左匹配的原理
对 `(a,b)` 字段建立联合索引，本质上还是一棵 `B+` 树，如下所示。`a` 按顺序排列（`1/1/2/2/3/3`），`b` 在 `a` 确定的情况下按顺序排列（全局无序，局部有序）。所以必须基于 `a` 来查找 `b` 字段，否则 `b` 就是无序的，就无法使用索引了。
![img](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2022/mysql/b_plus_tree1.png)

注意，上文描述的匹配不到，不代表不会触发索引，看下面的例子（联合索引字段为 `(a,b,c)`）：

####  查询示例
```sql
create table t_test(
	id bigint(20) primary key,
	a int not null default 0,
  b int not null default 0,
  c int not null default 0,
	`name` varchar(16) not null default '',
	index (a,b,c)
)engine=InnoDB;
```

```text
mysql> EXPLAIN  SELECT b FROM t_test WHERE a=2;
+----+-------------+--------+------------+------+---------------+------+---------+-------+------+----------+-------------+
| id | select_type | table  | partitions | type | possible_keys | key  | key_len | ref   | rows | filtered | Extra       |
+----+-------------+--------+------------+------+---------------+------+---------+-------+------+----------+-------------+
|  1 | SIMPLE      | t_test | NULL       | ref  | a             | a    | 4       | const |    1 |   100.00 | Using index |
+----+-------------+--------+------------+------+---------------+------+---------+-------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)

mysql> EXPLAIN  SELECT b FROM t_test WHERE b=2;
+----+-------------+--------+------------+-------+---------------+------+---------+------+------+----------+--------------------------+
| id | select_type | table  | partitions | type  | possible_keys | key  | key_len | ref  | rows | filtered | Extra                    |
+----+-------------+--------+------------+-------+---------------+------+---------+------+------+----------+--------------------------+
|  1 | SIMPLE      | t_test | NULL       | index | a             | a    | 12      | NULL |    1 |   100.00 | Using where; Using index |
+----+-------------+--------+------------+-------+---------------+------+---------+------+------+----------+--------------------------+
1 row in set, 1 warning (0.00 sec)
```

- `ref`：这种类型表示 mysql 会根据特定的算法快速查找到某个符合条件的索引，而不是会对索引中每一个数据都进行一一的扫描判断，也就是所谓你平常理解的使用索引查询会更快的取出数据。而要想实现这种查找，索引却是有要求的，要实现这种能快速查找的算法，索引就要满足特定的数据结构。简单说，也就是索引字段的数据必须是有序的，才能实现这种类型的查找，才能利用到索引。
- `index`：这种类型表示 mysql 会对整个该索引进行扫描。要想用到这种类型的索引，对这个索引并无特别要求，只要是索引，或者某个联合索引的一部分，mysql 都可能会采用 index 类型的方式扫描。缺点是效率不高，mysql 会从索引中的第一个数据一个个的查找到最后一个数据，直到找到符合判断条件的某个索引。

关于 `EXPLAIN` 的应用，下面再详述。

####  联合索引的意义（建议使用）
- 减少开销：建立 `1` 个联合索引 `(a,b,c)`，相当于 `(a)`、`(a,b)`、`(a,b,c)` 这 `3` 个索引。由于新建索引会增加写操作的开销和磁盘空间的开销，对于大量数据的表，推荐使用联合索引
- 避免回表：如对联合索引 `(a,b,c)`，如果查询的字段落在索引中，如 `SELECT a,b,c FROM t_test WHERE a=1 AND b=2`，那么 MySQL 可以直接通过遍历索引取得数据，而无需回表，减少了很多的随机 IO 操作

此外，联合索引相对于单索引，可以提升查询效率。索引列越多，通过索引筛选出的数据越少。比如要查询需要回表的 `SELECT *` 操作，联合索引筛选出需要回表的数据可能更小。

##  0x03  索引的要点


##  0x04  索引的应用及优化


##  0x05  索引失效

##  0x06  EXPLAIN 的使用

##  0x07  原则


##  0x08  参考
-   [【原创】MySQL(Innodb) 索引的原理](https://www.cnblogs.com/rjzheng/p/9915754.html)
-   [MySQL 索引背后的数据结构及算法原理](https://blog.codinglabs.org/articles/theory-of-mysql-index.html)
-   [为什么 MySQL 使用 B+ 树](https://draveness.me/whys-the-design-mysql-b-plus-tree/)
