---
layout: post
title: Mysql 项目应用笔记（一）
subtitle: Mysql 日常使用总结
date: 2022-02-01
author: pandaychen
catalog: true
tags:
  - MySQL
---

##  0x00    前言
本文是对项目中 Mysql 使用的一些经验汇总，包含如下几个方面：
-   分区表
-   索引的使用
-   插入 / 查询优化

![mysql-optimization](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2022/mysql/mysql-optimistic-all.png)

##  0x01    分区表
对于操作日志、任务日志等大量日志的存储场景，方便管理，比如日志仅保留最近半年（遇到使用 innodb 引擎的 MySQL 进行 `delete` 操作后，底层文件不会变小的问题），过期的冷备等，采用分区表是个不错的选择。关于分区表的使用，需要理解下面几点：
-   **分区表设计不解决性能问题，更多的是解决数据迁移和备份的问题**
-   需要根据业务场景设计好分区的方法及字段
-   分区表对于应用是透明的
-   MySQL 中的分区表是把一张大表拆成了多张表，每张表有自己的索引，从逻辑上看是一张表，但物理上存储在不同文件中
-   大多数需求场景是为了快速清理过期数据（避免数据量过大）

常用的几种分区方式：
-   `RANGE` 分区：范围分区，最常用，基于属于一个给定连续区间的列值，把多行分配给分区。最常见的是基于时间字段，`RANGE` 分区的特点是多个分区的范围要连续，但是不能重叠，默认情况下使用 `VALUES LESS THAN` 属性，即每个分区不包括指定的那个值
-   `LIST` 分区：同 `RANGE` 分区类似，区别在于 `LIST` 是枚举值列表的集合，`RANGE` 是连续的区间值的集合
-   `HASH` 分区：基于用户定义的表达式的返回值来进行选择的分区，该表达式使用将要插入到表中的这些行的列值进行计算。这个函数可以包含 MySQL 中有效的、产生非负整数值的任何表达式
-   `KEY` 分区：类似于按 `HASH` 分区，区别在于 `KEY` 分区只支持计算一列或多列，且 MySQL 服务器提供其自身的哈希函数。必须有一列或多列包含整数值

####    分区表创建例子
1、`RANGE` 分区，基于属于一个给定连续区间的列值，把多行分配给分区。最常见的是基于时间字段. 基于分区的列最好是整型，如果日期型的可以使用函数转换为整型 <br>
```sql
CREATE TABLE `t_my_range_datetime`(
    id INT,
    hiredate DATETIME,
    c varchar(128),
    PRIMARY KEY (id,hiredate,c),
    KEY idx_e (c)
)  ENGINE=InnoDB DEFAULT CHARSET=utf8
PARTITION BY RANGE (TO_DAYS(hiredate) ) (
    PARTITION p1 VALUES LESS THAN (TO_DAYS('20211202') ),
    PARTITION p2 VALUES LESS THAN (TO_DAYS('20211203') ),
    PARTITION p3 VALUES LESS THAN (TO_DAYS('20211204') ),
    PARTITION p4 VALUES LESS THAN (TO_DAYS('20211205') ),
    PARTITION p5 VALUES LESS THAN (TO_DAYS('20211206') ),
    PARTITION p6 VALUES LESS THAN (TO_DAYS('20211207') ),
    PARTITION p7 VALUES LESS THAN (TO_DAYS('20211208') ),
    PARTITION p8 VALUES LESS THAN (TO_DAYS('20211209') ),
    PARTITION p9 VALUES LESS THAN (TO_DAYS('20211210') ),
    PARTITION p10 VALUES LESS THAN (TO_DAYS('20211211') ),
    PARTITION p11 VALUES LESS THAN (MAXVALUE)
);
```

注意：使用 `RANGE` 分区方式，后续需要手动创建分区（如果分区用完的话，比如时间在 `20211211` 之后的数据），现网中通常是单独启动一个创建分区的进程，预先创建需要的分区。也可以使用上面的默认分区 `p11`（`p11` 是一个可选分区，如果在定义表的没有指定的这个分区，当我们插入大于 `20211211` 的数据的时候，会出错）

2、`HASH` 分区 <br>
和 `RANGE` 分区类似，区别在于 `LIST` 是枚举值列表的集合，`RANGE` 是连续的区间值的集合。`LIST` 分区只支持 `INT`，非整形字段需要通过函数转换成 `INT`
```sql
CREATE TABLE `t_my_list`(
    a int(11),
    b int(11),
    c varchar(128),
) ENGINE=InnoDB DEFAULT CHARSET=utf8
PARTITION BY LIST (b)(
    PARTITION p0 values in (1,3,5,7,9),
    PARTITION p1 values in (2,4,6,8,0)
);
```

注意：`LIST` 分区列需要是非 `NULL` 列，否则插入值如果枚举列表里面不存在 `NULL` 值会插入失败。这点和其它的分区方式不一样，`RANGE` 分区会将其作为最小分区值存储，`HASH` 及 `KEY` 分为会将其转换成 `0` 存储

3、`HASH` 分区 <br>
针对单个 `INT` 型的字段（列）进行 `HASH` 分区：
```sql
CREATE TABLE `t_my_hash_id` (
    id INT NOT NULL,
    fname VARCHAR(30),
    lname VARCHAR(30),
    created DATE NOT NULL DEFAULT '1970-01-01',
    separated DATE NOT NULL DEFAULT '9999-12-31',
    job_code INT,
    store_id INT
) ENGINE=InnoDB DEFAULT CHARSET=utf8
PARTITION BY HASH(id)
PARTITIONS 4;
```
一般 `HASH` 分区常用于并没有明显可以分区的特征字段且数据非常大的表，分区个数是预先设置好的。此外，`HASH` 分区只能针对整形字段，对于非整形的字段只能通过 MySQL 表达式（MySQL 任意有效的函数或者表达式）将其转换成整数。对于非整形的 HASH 往表插入数据的过程中会多一步表达式的计算操作，不建议使用复杂的表达式，对性能有影响。同 `RANGE` 分区和 `LIST` 分区一样，`PARTITION BY HASH (expr)` 子句中的 `expr` 返回的必须是整数值。

4、`LINEAR HASH` 分区 <br>
`LINEAR HASH` 分区是 `HASH` 分区的一种特殊类型，与 `HASH` 分区是基于 `MOD` 函数不同的是，本方式基于 Linear Hash[算法](https://dev.mysql.com/doc/refman/5.7/en/partitioning-linear-hash.html)：

```sql
CREATE TABLE my_members (
    id INT NOT NULL,
    fname VARCHAR(30),
    lname VARCHAR(30),
    hired DATE NOT NULL DEFAULT '1970-01-01',
    separated DATE NOT NULL DEFAULT '9999-12-31',
    job_code INT,
    store_id INT
) ENGINE=InnoDB DEFAULT CHARSET=utf8
PARTITION BY LINEAR HASH(id)
PARTITIONS 4;
```
本分区的优点是在数据量大的场景，譬如 TB 级，增加、删除、合并和拆分分区会更快；缺点是，相对于 `HASH` 分区，它数据分布不均匀的概率更大


5、`KEY` 分区 <br>
`KEY` 分区允许多列，而 `HASH` 分区只允许一列。如果在有主键或者唯一键的情况下，`KEY` 中分区列可不指定，默认为主键或者唯一键，如果没有，则必须显性指定列（如下例子，而KEY 分区对象必须为列，而不能是基于列的表达式），`HASH` 分区 `PARTITION BY KEY (column_list)`，基于的是列的 MD5 值

```sql
CREATE TABLE k1 (
    id INT NOT NULL PRIMARY KEY,
    name VARCHAR(20)
)
PARTITION BY KEY()
PARTITIONS 2;

CREATE TABLE tm1 (
    s1 CHAR(32)
)
PARTITION BY KEY(s1)
PARTITIONS 10;
```

####    其他要点
1.  分区表的创建需要主键（primary key）包含分区列
2.  在分区表中唯一索引（unique index）仅在当前分区文件唯一，而不是全局唯一
3.  分区表唯一索引推荐使用类似 `UUID` 的全局唯一实现
4.  分区表不解决性能问题，如果使用非分区列查询，性能反而会更差


## 0x02 参考
-   [哪些场景我不建议用分区表？](https://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/MySQL%E5%AE%9E%E6%88%98%E5%AE%9D%E5%85%B8/14%20%20%E5%88%86%E5%8C%BA%E8%A1%A8%EF%BC%9A%E5%93%AA%E4%BA%9B%E5%9C%BA%E6%99%AF%E6%88%91%E4%B8%8D%E5%BB%BA%E8%AE%AE%E7%94%A8%E5%88%86%E5%8C%BA%E8%A1%A8%EF%BC%9F.md)

