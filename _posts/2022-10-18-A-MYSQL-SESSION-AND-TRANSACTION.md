---
layout:     post
title:      Mysql：Session && Transaction
subtitle:   Mysql 基础回顾：会话与事务
date:       2022-08-18
author:     pandaychen
catalog:    true
tags:
    - MySQL
    - 事务
---

##  0x00    前言

-   会话（Session）： 指客户端与 MySQL 服务器之间的一个连接。当客户端连接到 MySQL 服务器时，服务器会为客户端分配一个会话。会话是数据库操作的基本单位，每个会话都有一个唯一的 ID。在会话中，用户可以执行 SQL 语句来查询和修改数据。会话可以持续很长时间，直到客户端断开与服务器的连接。
-   事务（Transaction）：事务是数据库操作的一个逻辑单元，它是由一系列的 SQL 语句（有限的数据库操作序列）组成。事务具有 ACID 特性（原子性 / 一致性 / 隔离性 / 持久性），这些特性保证了数据库在并发操作和系统故障的情况下，仍然能够保持数据的一致性和完整性。事务可以在一个会话中执行，也可以跨多个会话执行。事务是用户定义的一个数据库操作序列，这些操作要么全做，要么全不做，是一个不可分割的工作单位

-	锁：锁是用来解决并发问题的一种机制，主要是为了保证数据的一致性和完整性。锁的类型分为共享锁（读锁）和排他锁（写锁）。共享锁是指多个事务对同一数据可以共享一把锁，可以同时读取数据，但是不能修改数据。排他锁是指一旦一个事务获得了对数据的排他锁，其他事务就不能再对该数据加任何类型的锁。锁的粒度分为行锁和表锁，行锁锁定的是某一行数据，表锁锁定的是整个数据表


####    事务 && 会话 的区别
Session 是客户端与 MySQL 服务器之间的一个连接，它是数据库操作的基本单位；而 Transaction 是数据库操作的一个逻辑单元，它具有 ACID 特性，保证了数据库的一致性和完整性
-   生命周期：Session 的生命周期从客户端连接到服务器开始，直到客户端断开连接；而 Transaction 的生命周期从事务开始（如：START TRANSACTION）到事务结束（如：COMMIT 或 ROLLBACK）
-   范围：Session 是数据库操作的基本单位，可以包含多个事务；而 Transaction 是数据库操作的一个逻辑单元，由一系列的 SQL 语句组成
-   隔离性：Session 之间是相互隔离的，一个会话中的操作不会影响到其他会话；而 Transaction 之间的隔离性取决于事务隔离级别，不同的隔离级别会对并发事务产生不同的影响
-   上下文：Session 具有自己的上下文，例如变量、临时表等；而 Transaction 的上下文是在会话的基础上进行操作，例如锁、回滚段等

####	事务 && 锁的区别
事务和锁的主要区别在于：

-	事务是数据库操作的一个执行单元，而锁是并发控制的一种手段
-	事务是逻辑上的概念，主要解决操作的原子性问题；而锁是物理上的概念，主要解决资源的并发访问问题
-	事务的触发是用户行为，而锁的触发是系统行为

事务和锁配合使用的主要原因是为了更好地解决并发访问下的数据一致性、完整性和隔离性问题。虽然锁机制可以解决并发访问的问题，但在某些场景下，仅仅依靠锁还不足以保证数据的完整性和一致性，事务可以保证一组操作的原子性，这意味着这组操作要么全部成功，要么全部失败。在事务中可以对多个 SQL 语句进行组合，使它们在一个原子操作中执行。即使在并发访问的情况下，也能确保数据的一致性和完整性


####    不需要事务的场景
MySQL 的事务主要用于确保数据的完整性和一致性，特别是在多个操作同时对同一数据进行修改时。然而并非所有操作都需要使用事务。以下是一些不需要使用事务的情况：

-   查询操作：如果只是读取数据，而不进行任何修改，那么就不需要使用事务
-   单一的插入、更新或删除操作：如果只对数据库进行单一的插入、更新或删除操作，而且这个操作不依赖于其他操作的结果，那么就不需要使用事务
-   无需保证原子性的操作：如果操作不需要保证原子性，即使在操作过程中出现错误，也不会对数据造成不一致性，那么就不需要使用事务
-   对数据一致性要求不高的操作：在某些场景下，例如日志记录、统计信息收集等，即使数据有一些小的不一致，也不会对系统产生严重影响，这种情况下可以不使用事务


##  0x01    基础例子
通过关于银行转账的例子演示如何使用事务来确保数据的完整性和一致性，假设有两个表：`accounts` 和 `transactions`，分别用于存储账户信息和交易记录：

```SQL
CREATE TABLE accounts (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(50),
  balance DECIMAL(10, 2)
);

CREATE TABLE transactions (
  id INT PRIMARY KEY AUTO_INCREMENT,
  from_account INT,
  to_account INT,
  amount DECIMAL(10, 2),
  transaction_date DATETIME
);
```


现在需要在 A 账户和 B 账户之间进行转账操作。假设 A 账户 ID 为 `1`，B 账户 ID 为 `2`，转账金额为 `100`，使用事务处理转账操作的步骤如下：

```SQL
开始一个新事务：
START TRANSACTION;
从 A 账户扣除转账金额：
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
向 B 账户添加转账金额：
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
记录交易信息：
INSERT INTO transactions (from_account, to_account, amount, transaction_date) VALUES (1, 2, 100, NOW());
检查 A 账户余额是否充足。如果余额不足，回滚事务；否则，提交事务
SELECT balance FROM accounts WHERE id = 1;
根据查询结果判断 A 账户余额是否充足。如果充足，执行以下命令提交事务：
COMMIT;
如果余额不足，执行以下命令回滚事务：
ROLLBACK;
```

使用 xorm 实现上述逻辑如下：

```GO
type Account struct {
	Id      int64   `xorm:"pk autoincr"`
	Name    string  `xorm:"varchar(50)"`
	Balance float64 `xorm:"decimal(10,2)"`
}

type Transaction struct {
	Id             int64   `xorm:"pk autoincr"`
	FromAccount    int64   `xorm:"int"`
	ToAccount      int64   `xorm:"int"`
	Amount         float64 `xorm:"decimal(10,2)"`
	TransactionDate time.Time
}

func main() {
	engine, err := xorm.NewEngine("mysql", "username:password@tcp(localhost:3306)/dbname?charset=utf8")
	if err != nil {
		fmt.Println("Failed to create engine:", err)
		return
	}

	if err = engine.Sync2(new(Account), new(Transaction)); err != nil {
		fmt.Println("Failed to sync tables:", err)
		return
	}

	fromAccountId := int64(1)
	toAccountId := int64(2)
	transferAmount := 100.0

	err = Transfer(engine, fromAccountId, toAccountId, transferAmount)
	if err != nil {
		fmt.Println("Transfer failed:", err)
	} else {
		fmt.Println("Transfer succeeded")
	}
}

// 在 Transfer 函数中，首先获取并检查 A 账户的余额，如果余额充足，我们更新 A 和 B 账户的余额，并插入一条新的交易记录。最后，我们根据操作结果提交或回滚事务
func Transfer(engine *xorm.Engine, fromAccountId, toAccountId int64, amount float64) error {
	session := engine.NewSession()
	defer session.Close()

	if err := session.Begin(); err != nil {
		return err
	}

	var fromAccount Account
	if has, err := session.ID(fromAccountId).Get(&fromAccount); err != nil {
		session.Rollback()
		return err
	} else if !has {
		session.Rollback()
		return fmt.Errorf("account %d not found", fromAccountId)
	}

	if fromAccount.Balance < amount {
		session.Rollback()
		return fmt.Errorf("account %d has insufficient balance", fromAccountId)
	}

	fromAccount.Balance -= amount
	if _, err := session.ID(fromAccountId).Update(&fromAccount); err != nil {
		session.Rollback()
		return err
	}

	var toAccount Account
	if has, err := session.ID(toAccountId).Get(&toAccount); err != nil {
		session.Rollback()
		return err
	} else if !has {
		session.Rollback()
		return fmt.Errorf("account %d not found", toAccountId)
	}

	toAccount.Balance += amount
	if _, err := session.ID(toAccountId).Update(&toAccount); err != nil {
		session.Rollback()
		return err
	}

	transaction := Transaction{
		FromAccount:    fromAccountId,
		ToAccount:      toAccountId,
		Amount:         amount,
		TransactionDate: time.Now(),
	}

	if _, err := session.Insert(&transaction); err != nil {
		session.Rollback()
		return err
	}

	return session.Commit()
}
```

如果只使用 session 实现，不使用事务，那么下面的代码存在什么问题呢？不推荐用于处理关键业务逻辑，因为它可能导致数据不一致。在上面的银行转账示例中，如果在更新 A 账户余额和 B 账户余额之间发生异常错误（比如 crash，进程重启等），可能会导致资金丢失或错误。使用事务可以确保所有操作要么全部成功，要么全部失败，从而保持数据的一致性


```GO
func TransferWithoutTransaction(engine *xorm.Engine, fromAccountId, toAccountId int64, amount float64) error {
	session := engine.NewSession()
	defer session.Close()

	var fromAccount Account
	if has, err := session.ID(fromAccountId).Get(&fromAccount); err != nil {
		return err
	} else if !has {
		return fmt.Errorf("account %d not found", fromAccountId)
	}

	if fromAccount.Balance < amount {
		return fmt.Errorf("account %d has insufficient balance", fromAccountId)
	}

	fromAccount.Balance -= amount
	if _, err := session.ID(fromAccountId).Update(&fromAccount); err != nil {
		return err
	}

	var toAccount Account
	if has, err := session.ID(toAccountId).Get(&toAccount); err != nil {
		return err
	} else if !has {
		return fmt.Errorf("account %d not found", toAccountId)
	}

	toAccount.Balance += amount
	if _, err := session.ID(toAccountId).Update(&toAccount); err != nil {
		return err
	}

	transaction := Transaction{
		FromAccount:    fromAccountId,
		ToAccount:      toAccountId,
		Amount:         amount,
		TransactionDate: time.Now(),
	}

	if _, err := session.Insert(&transaction); err != nil {
		return err
	}

	return nil
}
```

##	0x01 MySQL 体系结构

####	SQL 语句执行流程

![mysql-sql-execute-flow]()

1.	MySQL 客户端向 MySQL server 发起请求，获取到一个连接请求
2.	在 connector（连接管理器）创建连接，分配线程，并验证用户名、密码和库表权限等
3.	判断是否打开了 query_cache（查询缓存），如果打开了，则进入第 `4` 步，否则进入第 `5` 步
4.	根据当前 SQL 语句，获取到 hash 值，通过 hash 查询 cache&buffer 中是否有在过期时间范围内的数据。有则直接返回。否则进入第 `5` 步
5.	SQL 连接组件接收 SQL 语句，并将 SQL 语句分解成 MySQL 能够识别的数据结构（抽象语法树）
6.	Optimizer（查询优化器）根据 SQL 语句的特征，生产查询路径树，并选举一条最优的查询路径。
7.	调用存储引擎接口，打开表，执行查询。同时检查存储引擎中是否有对应的缓存记录，如果有，则直接返回。如果没有就继续往下执行
8.	到数据文件中寻找数据（如果有索引，会先通过索引查找）
9.	当查询到所需要的数据之后，先写入存储引擎缓存中。如果打开了 query_cache，也会同时写进去
10.	返回数据给客户端，同时关闭表，线程和连接

####	InnoDB 存储结构
一个记录的检索过程分为如下两步：

1.	检索记录所在的 Page：Page 的检索是通过索引构建的 B+Tree 进行检索的
2.	在 Page 中检索记录：Page 内的记录的检索是通过 Slot 槽位构建的 HashMap（跳表）进行检索的

##	0x02	MySQL 事务

####	事务的并发问题
1. 脏读：一个事务中访问到了另外一个事务未提交的数据，如果会话 `2` 更新 age 为 `10`，但是在 commit 之前，会话 `1` 此时获取 age，那么会获得的值就是会话 `2` 中尚未 commit 的值。如果会话 `2` 再次更新 age 或执行 rollback，而会话 `1` 已经拿到了 `age=10` 的值，这就是脏读

![dirty]()

2. 不可重复读：是一个事务读取同一条记录 `2` 次，得到的结果不一致，由于在读取中间变更了数据，所以会话 `1` 事务查询期间的得到的结果就不一样了

![read]()

3. 幻读：指一个事务读取 `2` 次，得到的记录条数不一致。由于在会话 `1` 之间插入了一个新的数据，所以得到的两次数据就不一样了

![phantom]()

####	事务隔离级别
为了解决上面的事务并发问题，SQL 规定了四个隔离水平：

-	读未提交（Read Uncommited）：该隔离级别安全最低，允许脏读
-	读已提交（Read Commited）：存在不可重复读取，但不允许脏读。读已提交只允许获取已经提交的数据
-	可重复读（Repeatable Read）：保证在事务处理过程中，多次读取同一个数据时，其值都和事务开始时刻是一致的，因此该事务级别禁止不可重复读取和脏读，但是有可能出现幻读
-	串行化（Serializable）：最严格的事务隔离级别，它要求所有事务被串行执行，即事务只能一个接一个的进行处理，不能并发执行


| 隔离级别 | 脏读 | 不可重复读 | 幻读 |
| :-----:| :----: | :----: | :----: |
| 读未提交 | 存在 | 存在 | 存在 |
| 读已提交 | 不存在 | 存在 | 存在 |
| 可重复读 | 不存在 | 不存在 | 存在 |
| 串行化 | 不存在 | 不存在 | 不存在 |

隔离级别越高，越能保证数据的完整性和一致性，但是对并发性能的影响也越大。对于多数应用程序，可以优先考虑把数据库系统的隔离级别设为 Read Committed。如何解决事务并发问题？MySQL、Oracle 中，为了性能的考虑并不是完全按照上面的 SQL 标准来实现的。数据库实现事务隔离的方式，基本可以分为以下两种：

1.	在读取数据前，对其加锁，阻止其他事务对数据进行修改
2.	不用加任何锁，通过一定机制生成一个数据请求时间点的一致性数据快照（Snapshot），并用这个快照来提供一定级别（语句级或事务级）的一致性读取。从用户的角度来看，数据库可以提供同一数据的多个版本，因此，这种技术叫做数据多版本并发控制（ＭultiVersion Concurrency Control）

####	MVCC
MySQL 通过 MVCC 机制，实现了事务的多版本控制。最早的数据库系统，只有读读之间可以并发，读写 / 写读 / 写写都要阻塞，引入多版本之后，只有写写之间相互阻塞，其他三种操作都可以并行，这样大幅度提高了 InnoDB 的并发度。

![mvcc-1]()

在 MySQL 中，主要通过这 `3` 种机制协同工作实现了 MVCC：

1、隐藏字段

在插入一条记录时，除了用户指定的数据字段外，MySQL 会默认额外添加如下三个隐藏字段：
-	`db_row_id`：随着新行插入而单调递增的占 `6` 字节的行 ID（并非顺序递增），主要作用是当表没有主键或唯一非空索引时，innodb 就会使用这个行 ID 自动产生聚簇索引。如果表有主键或唯一非空索引，聚簇索引就不会包含这个行 ID 了。`DB_ROW_ID` 跟 MVCC 关系不大。如果先没有指定主键，在表运行一段时间后，创建主键，那么会 MySQL 会进行聚簇索引的重建，会将新创建的主键或唯一索引作为聚簇索引，构建表
-	`db_trx_id`：事务 id，表示最近一次对本记录行作修改（`INSERT` 或 `UPDATE`）的事务 ID。至于 `DELETE` 操作，InnoDB 认为是一个 `UPDATE` 操作，不过会更新一个另外的删除位，将行表示为 `deleted`，并非真正删除
-	`db_roll_ptr`：回滚指针，指向该版本记录的前一个版本，实际指向的是当前记录行的 ** 上一版本 ** 的 undo log 信息

2、Read View

Read View 主要是用来做可见性分析的, 里面保存了 ** 对本事务不可见的其他活跃事务 **

1. 当一个事务被创建时，会创建该事务的 Read View（读视图），主要保存如下三个信息：

-	当前活跃事件列表（即当前还未提交的事务 id）
-	列表中最小事务 ID：小于当前事务 ID 的数据都是可见的，因为这些事务肯定已经 commit 或者 rollback
-	列表中最大事务 ID：大于当前事务 ID 的数据都是不可见的，需要通过回滚指针找到下一个版本

2. 一个事务通过 Read View 做可见性分析时，遵循如下规则：

![]()

-	小于活跃事件列表中最小事务 id：该事务已提交，可见
-	大于活跃事件列表中最大事务 id：该事务 id 对应的事务在当前事务启动之后才开启，不可见
-	在事务 id 处于 `(min_trx_id, max_trx_id)` 时，就需要校验是否在当前活跃事件列表中：因为事务的执行时间是不确定的，在 `min_trx_id` 后面启动的事务，可能已经提交，这时也是可见的：
	-	不在活跃事件列表中：说明该事务在当前事务启动时，已经提交，可见
	-	在活跃事件列表中：说明该事务在当前事务启动时，还没有提交，处于活跃状态，哪怕在当前事务 启动之后提交，也是不可见的，不可见


3、Undo log

回滚日志，Undo log 中存储的是老版本数据，当一个事务需要读取记录行时，** 如果当前记录行不可见 **，可以顺着 undo log 链找到满足其可见性条件的记录行版本。

1. Undo log 分类：在 InnoDB 里，undo log 分为如下两类：
-	insert undo log：事务对 insert 新记录时产生的 undo log, 只在事务回滚时需要, 并且在事务提交后就可以立即丢弃
-	update undo log：事务对记录进行 delete 和 update 操作时产生的 undo log，不仅在事务回滚时需要，快照读也需要，只有当数据库所使用的快照中不涉及该日志记录，对应的回滚日志才会被 purge 线程删除

2. Undo log 的作用是：a）保证事务原子性，即可以通过 undo log 做回滚操作；b）实现数据多版本：即实现 MVCC 机制。结合上面的 Read View 来看，一个事务会在上述的链表中，最终找到一个在当前事务中可见的记录版本

一条记录在经过多个事务修改后，在数据库中，大致的一个数据存储结构如下图所示：

![]()

####	当前读 VS 快照读
1.	在 innodb 中创建一个新事务后，执行第一个 `select` 语句的时候，innodb 会创建一个快照（Read View），快照中会保存系统当前不应该被本事务看到的其他活跃事务 id 列表（即 trx_ids）
2.	当用户在这个事务中要读取某个记录行的时候，innodb 会将该记录行的 `DB_TRX_ID` 与该 Read View 中的一些变量进行比较，判断是否满足可见性条件（见 Read View 可见性分析）

-	快照读（snapshot read）：也叫普通读，即通过回滚指针构成的链表，读取到当前事务对应的事务 id** 可见的数据版本 **。普通的 `select` 语句（但是不包括 `select ... lock in share mode`，`select ... for update`）就是快照读；该方法是利用 MVCC 机制读取快照中的数据（不加锁的简单的 SELECT 都属于快照读）
-	当前读（current read）：即直接读取聚簇索引中存储的最新数据，** 不会通过记录中的回滚指针向下寻找 **，如 `select ... lock in share mode`，`select ... for update`，`insert`，`update`，`delete` 语句都是属于当前读
-	快照读是基于 MVCC 实现的，提高了并发的性能，降低开销
-	大部分业务代码中的读取都属于快照读
-	当前读读取的是记录的最新版本，读取时会对读取的记录进行加锁, 其他事务就有可能阻塞。加锁的 SELECT，或者对数据进行增删改都会进行当前读，update、delete、insert 语句虽然没有 select, 但是它们也会先进行读取，而且只能读取最新版本

```SQL
SELECT * FROM user LOCK IN SHARE MODE; # 共享锁
SELECT * FROM user FOR UPDATE; # 排他锁
INSERT INTO user values ... # 排他锁
DELETE FROM user WHERE ... # 排他锁
UPDATE user SET ... # 排他锁
```

注意：只靠 MVCC 实现 RR（Repeatable Read）隔离级别，** 可以保证可重复读，还能防止部分幻读，但并不是完全防止 **。因此，InnoDB 在实现 RR 隔离级别时，不仅使用了 MVCC，还会对当前读语句读取的记录行加记录锁（record lock）和间隙锁（gap lock），禁止其他事务在间隙间插入记录行，来防止幻读。也就是行级锁 + MVCC，这样就可以防止幻读问题

此外，再讨论下 RR 和 RC 的 Read View 产生区别：

1.	在 innodb 中的 Repeatable Read 级别, 只有事务在 begin 之后，执行第一条 `select`（读操作）时, 才会创建一个快照 (read view)，将当前系统中活跃的其他事务记录起来；并且事务之后都是使用的这个快照，不会重新创建，直到事务结束
2.	在 innodb 中的 Read Committed 级别, 事务在 begin 之后，执行每条 `select`（读操作）语句时，快照会被重置，即会重新创建一个快照 (read view)


####	Mysql是如何避免幻读的？
在 RR 级别中，如果仅仅通过 MVCC 的话，** 只能防止快照读的幻读发生，而不能防止当前读产生的幻读问题 **。因此 MySQL 通过 **MVCC + 行锁 ** 实现的 RR 事务隔离级别

1、普通读是如何避免幻读的？

MySQL在可重复读隔离级别下，是通过MVCC机制避免幻读的。
MVCC机制，可以简单理解成在事务启动的时候对数据库拍了个“快照”，它保留了那个时刻数据库的数据状态，那么这个事务后续的读取都可以从这个“快照”中获取，哪怕其他事务新加了数据，也不会影响到“快照”中的数据，也就不会出现幻读了。

关于MVCC的详细机制强烈你阅读本文看完这篇还不懂MySQL的MVCC机制算我输



事务A在启动的时候创建了一个“快照”，查询出结果“小红，小蓝”
后续事务B插入一条记录“小飞”，提交
然后事务A再次同样查询条件查询，它会使用“快照”读取，所以还是“小红，小蓝”

小结： 针对快照读（普通 select 语句），是通过 MVCC 方式解决了幻读。
当前读是如何避免幻读的？
普通读（快照读）实际上读取的是历史版本中的数据，但一直用这种方式读取在某些场景下是有问题的。
假设你要 update 一个记录，但是另一个事务已经 delete 这条记录并且提交事务了，这样不是会产生冲突吗，所以 update 的时候肯定要知道最新的数据。也就是要做当前读。
那么针对当前读，MySQL在可重复读隔离级别下是如何避免幻读的呢？
也就是说不能读取“快照”了，因为你要最新状态的数据，那么能不能在当前读的时候，对这段区间都加上锁，让别的事务阻塞，无法插入。因此，MySQLInnoDB引擎为了解决可重复读隔离级别使用当前读而造成的幻读问题，引入了间隙锁。

表中有一个范围 id 为（3，5）间隙锁，那么其他事务就无法插入 id = 4 这条记录了，这样就有效的防止幻读现象的发生。
举个例子：


事务A的for_update是属于当前读，它会对锁定 id 范围 (2, +∞] ，相当于理解是间隙锁。
事务B插入了id=5的数据，(2, +∞]范围被锁定了，所以无法插入，阻塞。
通过这种加锁阻塞的方式，也可以避免幻读。

小结： 针对当前读（select ... for update 等语句），是通过 next-key lock（记录锁+间隙锁）方式解决了幻读。

关于MySQL锁的更多内容查看盘点MySQL中的各种锁

总结
MySQL默认采用的隔离级别是可重复读，在这种隔离级别下不同的读模式，针对幻读问题采用了不同解决方案：

针对快照读（普通 select 语句），是通过 MVCC 方式解决了幻读。
针对当前读（select ... for update 等语句），是通过 next-key lock（记录锁+间隙锁）方式解决了幻读。




#####


通过 MVCC 机制，虽然让数据变得可重复读，但读到的数据可能是历史数据，是不及时的数据，不是数据库当前的数据。对于数据的时效特别敏感业务可能出问题，对于这种读取历史数据的方式称为快照读（snapshot read），而读取数据库当前版本数据的方式称为当前读（current read）



在内部实现中，InnoDB 通过 undo log 保存每条数据的多个版本，并且能够找回数据历史版本提供给用户读，每个事务读到的数据版本可能是不一样的。

在同一个事务中，用户只能看到该事务创建快照之前已经提交的修改和该事务本身做的修改。

 MVCC 实现方式


Read View

Undo log
回滚日志。Undo log 中存储的是老版本数据，当一个事务需要读取记录行时，如果当前记录行不可见，可以顺着 undo log 链找到满足其可见性条件的记录行版本。

1. Undo log 分类：在 InnoDB 里，undo log 分为如下两类：

insert undo log : 事务对 insert 新记录时产生的 undo log, 只在事务回滚时需要, 并且在事务提交后就可以立即丢弃。
update undo log : 事务对记录进行 delete 和 update 操作时产生的 undo log，不仅在事务回滚时需要，快照读也需要，只有当数据库所使用的快照中不涉及该日志记录，对应的回滚日志才会被 purge 线程删除。
2. Undo log 的作用：Undo log 在 MySQL 中主要有一下两个用处：
一条记录在经过多个事务修改后，在数据库中，大致的一个数据存储结构如下图所示：


保证事务原子性：即可以通过 undo log 做回滚操作。
实现数据多版本：即实现 MVCC 机制。结合上面的 Read View 来看，一个事务会在上述的链表中，最终找到一个在当前事务中可见的记录版本。
当前读与快照读
在 innodb 中，创建一个新事务后，执行第一个 select 语句的时候，innodb 会创建一个快照（read view），快照中会保存系统当前不应该被本事务看到的其他活跃事务 id 列表（即 trx_ids）。

当用户在这个事务中要读取某个记录行的时候，innodb 会将该记录行的 DB_TRX_ID 与该 Read View 中的一些变量进行比较，判断是否满足可见性条件（Read View 可见性分析）。

快照读 (snapshot read)：即通过回滚指针构成的链表，读取到当前事务对应的事务 id` 可见 ` 的数据版本。普通的 select 语句 (不包括 select ... lock in share mode, select ... for update) 就是快照读。
当前读 (current read) ：即直接读取聚餐索引中存储的最新数据，不会通过记录中的 ` 回滚指针 ` 向下寻找。`select ... lock in share mode`，`select ... for update`，`insert`，`update`，`delete` 语句都是属于当前读。
注意：只靠 MVCC 实现 RR 隔离级别，可以保证可重复读，还能防止 ` 部分幻读 `，但并不是完全防止。因此，InnoDB 在实现 RR 隔离级别时，不仅使用了 MVCC，还会对 “当前读语句” 读取的记录行加记录锁（record lock）和间隙锁（gap lock），禁止其他事务在间隙间插入记录行，来防止幻读。也就是前文说的 "行级锁 + MVCC"。

RR 和 RC 的 Read View 产生区别
在 innodb 中的 Repeatable Read 级别, 只有事务在 begin 之后，执行第一条 select（读操作）时, 才会创建一个快照 (read view)，将当前系统中活跃的其他事务记录起来；并且事务之后都是使用的这个快照，不会重新创建，直到事务结束。
在 innodb 中的 Read Committed 级别, 事务在 begin 之后，执行每条 select（读操作）语句时，快照会被重置，即会重新创建一个快照 (read view)。

我们来看看不同操作时 MVCC 机制的体现：

Insert：InnoDB 在 MVCC 机制中，会在创建时间隐藏列更新为当前的事务 ID，删除时间列是 undefined
Select：MVCC 中只会返回创建时间小于或等于当前事务 ID 且删除时间要么是 undefined 要么大于当前事务 ID 的记录，两个条件都满足的记录才是正确的
Delete：MVCC 会在删除时间列更新当前事务 ID，然后真正删除是由 MySQL 后台运行的独立线程去定时清理的
Update：MVCC 中的 Update 被拆分成了 Insert 和 Delete 操作，显示插入一个更新后的记录（主键可能重复），然后标记原始记录，等待其被删除

关于 MySQL 对于 MVCC 的实现想了解可以参考这篇文章，这里不做过多的赘述
通过 MVCC 机制，虽然让数据变得可重复读，但读到的数据可能是历史数据，是不及时的数据，不是数据库当前的数据！这在一些对于数据的时效特别敏感的业务中，就很可能出问题。
对于这种读取历史数据的方式，我们叫它快照读 (snapshot read)，而读取数据库当前版本数据的方式，叫当前读 (current read)

快照读和当前读

快照读：简单的 Select 操作，不加锁（在隔离级别为序列化时，MySQL 会取消快照读，所有的读操作也会加读锁）, 也不会被其他加锁阻塞

sql 复制代码例：select * from table where ?;

其中：
    Read Committed（读已提交）隔离级别：每次 select 都生成一个快照读。
    Read Repeatable（可重复读）隔离级别：开启事务后第一个 select 语句才是快照读的地方，
    而不是一开启事务就快照读，之后每次读取为同一个快照。


当前读：特殊的读操作，Insert、Update、Delete 操作，处理的都是当前的数据，需要加锁

sql 复制代码例：
    select * from table where ? lock in share mode;（共享读锁）
    select * from table where ? for update(排他锁);
    insert into table values (…)（插入意向锁）;
    update table set ? where ?（排他锁）;
    delete from table where ?（排他锁）;

为什么将插入 / 更新 / 删除操作，都归为当前读？我们看下一个 Update 操作的具体流程。

当 Update SQL 被发给 MySQL 后，MySQL Server 会根据 where 条件，读取第一条满足条件的记录，然后 InnoDB 引擎会将第一条记录返回，并加锁 (共享锁)。待 MySQL Server 收到这条加锁的记录之后，会再发起一个 Update 请求，申请升级为排他锁，更新这条记录。一条记录操作完成，再读取下一条记录，直至没有满足条件的记录为止。因此，Update 操作内部，就包含了一个当前读。同理，Delete 操作也一样。Insert 操作会稍微有些不同，简单来说，就是 Insert 操作可能会触发 Unique Key 的冲突检查（插入意向锁），也会进行一个当前读。



####	事务的隔离级别

二、事务的隔离级别
SQL 标准定义了以下几种事务隔离级别
复制代码 READ_UNCOMMITTED 读未提交：最低级别，一个事务可以读取另一个未提交事务的数据。幻想读、不可重复读和脏读都允许。

READ_COMMITTED 读已提交：一个事务要等另一个事务提交后才能读取数据。允许幻想读、不可重复读，不允许脏读。

REPEATABLE_READ 可重复读：在开始读取数据（事务开启）时，不再允许修改操作。允许幻想读，不允许不可重复读和脏读。

SERIALIZABLE 可串行化：最高级别，在该级别下，事务串行化顺序执行。幻想读、不可重复读和脏读都不允许。

事务不同隔离级别引发的问题
复制代码 1  脏读，脏读是指在一个事务处理过程里读取了另一个未提交的事务中的数据。
2  不可重复读，不可重复读是指在对于数据库中的某个数据，一个事务范围内多次查询却返回了不同的数据值，这是由于在查询间隔，被另一个事务修改并提交了。
3  幻读，幻读和不可重复读都是读取了另一条已经提交的事务（这点就脏读不同），所不同的是不可重复读查询的都是同一个数据项，而幻读针对的是一批数据整体。


MySQL 数据库中，默认的隔离级别为 Repeatable read (可重复读)；

####	业界数据库的默认隔离级别

| Database | Default isolation Level |
| :-----:| :----: |
| Oracle | READ_COMMITTED |
| MySQL | REPETABLE_READ |
|Microsoft SQL Server|READ_COMMITTED|
|PostgreSQL|READ_COMMITTED|
|DB2|CURSOR STABILITY (a.k.a READ_COMMITTED)|

MySQL 数据库中，默认的隔离级别为 Repeatable read（可重复读）

隔离级别越高，越能保证数据的完整性和一致性，但是对并发性能的影响也越大。对于多数应用程序，可以优先考虑把数据库系统的隔离级别设为 Read Committed。它能够避免脏读取，而且具有较好的并发性能。尽管它会导致不可重复读、幻读和第二类丢失更新这些并发问题，在可能出现这类问题的业务场景，可以采用悲观锁或乐观锁来控制。

####	事务的实现方式


####	小结：事务 && MVCC && 锁
小结下：

1、不同的隔离级别能解决的并发问题，隔离级别越高，性能也会越差，现网应用需要取舍

2、隔离性的实现机制，根据事务的读写场景不一致，大致可以分为两类：

- 事务 A 写操作对事务 B 写操作的影响：InnoDB 通过锁机制保证隔离性，隔离性要求同一时刻只能有一个事务对数据进行写操作。事务在修改数据之前，需要先获得相应的锁；获得锁之后，事务便可以修改数据。该事务操作期间，这部分数据是锁定的，其他事务如果需要修改数据，需要等待当前事务提交或回滚后释放锁
- 事务 A 写操作对事务 B 读操作的影响：InnoDB 通过 MVCC 保证隔离性，在同一时刻，不同的事务读取到的数据可能是不同的（多版本），每个事务会有一个自己的 `ReadView`, 所谓 `ReadView`，是指事务在某一时刻给整个事务系统打快照，之后再进行读操作时，会将读取到的数据中的事务 `id` 与 `trx_sys` 快照比较，从而判断数据对该 `ReadView` 是否可见


##	0x03	Mysql 的锁



FROM: 深入浅出 mysql 锁

不使用事务的话，默认的 SQL 语句会用到锁机制吗？答案是肯定的，即使不显式使用事务，MySQL 默认的 SQL 语句也会使用锁机制。原因是为了保证数据的一致性和完整性。在 MySQL 中，锁是自动管理的，当执行一个 `UPDATE` 或 `DELETE` 等修改数据的语句时，MySQL 会自动给涉及的数据行加上排他锁（写锁），执行完毕后再自动释放。同样当执行一个 `SELECT ... FOR UPDATE` 语句时，MySQL 也会自动给涉及的数据行加上共享锁（读锁），执行完毕后再自动释放。即使没有显式使用事务，MySQL 也会在必要时使用锁来保证数据的一臀性和完整性

####	分类

1、悲观锁

悲观锁指的是对数据被外界（包括本系统当前的其他事务，以及来自外部系统的事务处理）修改持保守态度，因此，在整个数据处理过程中，将数据处于锁定状态。例如 `select...for update` 是 MySQL 提供的实现悲观锁的方式。在悲观锁的情况下，为了保证事务的隔离性，其它事务无法修改这些数据

2、乐观锁

相对悲观锁而言，乐观锁机制采取了更加宽松的加锁机制。悲观锁依靠数据库的锁机制实现，以保证操作最大程度的独占性。但悲观锁随之而来的就是数据库性能的大量开销，而乐观锁机制在一定程度上解决了这个问题。乐观锁大多是基于数据版本（`version`）记录机制实现，即通过为数据库表增加一个数字类型的 `version` 字段，当读取数据时，将 `version` 字段的值一同读出，数据每更新一次，对此 `version` 值 `+1`。当提交更新的时候，判断数据库表对应记录的当前版本信息与第一次取出来的 `version` 值进行比对，如果数据库表当前版本号与第一次取出来的 `version` 值相等，则予以更新，否则认为是过期数据，返回更新失败

#####	锁的粒度
MySQL 定义了两种锁的粒度：表级、行级锁，对比如下：

-	表锁：由 MySQL Server 控制，分为读锁和写锁。优点是开销小，加锁快；不会出现死锁；缺点是锁定粒度大，发生锁冲突的概率最高，并发度最低。表锁适合查询多、更新少的场景；当对表加了读锁，则会话只能读取当前被加锁的表，其它会话仍然可以对表进行读取但不能写入；当对表加了写锁，则会话可以读取或写入被加锁的表，其它会话不能对加锁的表进行读取或写入
-	行锁：由存储引擎实现（InnoDB）。优点是开销大，加锁慢；会出现死锁；缺点是锁定粒度最小，发生锁冲突的概率最低，并发度也最高。InnoDB 引擎下默认使用行级锁。行级锁适合按索引更新频率高的场景

####	行锁分类
1、记录锁（Record lock）：在唯一索引列或主键列记录上加锁，且该值存在，否则加锁类型为间隙锁。例如 `SELECT a FROM t WHERE a = 12 FOR UPDATE`，对值为 `12` 的索引进行锁定，防止其它事务插入、删除、更新值为 `a = 12` 的记录行

2、间隙锁（Gap Lock）：只有在可重复读、串行化隔离级别才有，在索引记录之间的间隙中加锁，或者是在某一条索引之前或者之后加锁，并不包括该索引本身，例如：`SELECT a FROM t WHERE a > 15 and a <20 FOR UPDATE`，且 `a` 存在的值为 `1`、`2`、`5`、`10`、`15`、`20`，则将 `(15,20)` 的间隙锁住

间歇锁的范围：
-	对主键或唯一索引当前读时，`WHERE` 条件全部精确命中 (`=` 或者 `in`)，这种场景本身就不会出现幻读，所以只会加记录锁
-	没有索引的列当前读操作时，会加全表 gap 锁，生产环境要注意。（所有主键 x 锁，所有主键间隙 gap 锁）
-	非唯一索引列，如果 where 条件部分命中 (>、<、like 等) 或者全未命中，则会加附近 Gap 间隙锁。例如，某表数据如下，非唯一索引 2,6,9,9,11,15。delete from table where another_id = 9 要操作非唯一索引列 9 的数据，gap 锁将会锁定的列是 (6,11)，该区间内无法插入数据。

注意：在使用范围条件检索并锁定记录时，间歇锁机制会阻塞符合条件范围内键值的并发插入，这往往会造成严重的锁等待。因此，在实际应用开发中，尤其是并发插入比较多的应用，要尽量优化业务逻辑，尽量使用相等条件来访问更新数据，避免使用范围条件

间隙锁和间隙锁之间是互不冲突的，间隙锁唯一的作用就是为了防止其他事务的插入，在 RR（可重复读）级别下解决了幻读的问题。例如 id 有 3,4,5，间隙锁锁定 id>3 的数据，是指的 4 及后面的数字都会被锁定。这样的话加入新的数据 id=6，就会被阻塞，从而避免了幻读。

3、临键锁（Next-Key Lock）：是记录锁和间隙锁的合集。只有在可重复读、串行化隔离级别才有。

![]()

例如一个索引有 10,11,13,20 这四个值。InnoDB 可以根据需要使用记录锁将 10，11，13，20 四个索引锁住，也可以使用间隙锁将 (-∞,10)，(10,11)，(11,13)，(13,20)，(20,+∞) 五个范围区间锁住。而临键锁是记录锁和间隙锁的合集。

4、插入意向锁（Insert Intention Locks）：是一种特殊的间隙锁，只有在执行 INSERT 操作时才会加锁，插入意向锁之间不冲突，可以向一个间隙中同时插入多行数据，但插入意向锁与间隙锁是冲突的，当有间隙锁存在时，插入语句将被阻塞，正是这个特性解决了幻读的问题。

假设有一个记录索引包含键值 4 和 7，不同的事务分别插入 5 和 6，每个事务都会产生一个加在 4-7 之间的插入意向锁，获取在插入行上的排它锁，但是不会被互相锁住，因为数据行并不冲突。

意向锁
innodb 的意向锁主要用户多粒度的锁并存的情况。比如事务 A 要在一个表上加 S 锁，如果表中的一行已被事务 B 加了 X 锁，那么该锁的申请也应被阻塞。如果表中的数据很多，逐行检查锁标志的开销将很大，系统的性能将会受到影响。为了解决这个问题，可以在表级上引入新的锁类型来表示其所属行的加锁情况，这就引出了 “意向锁” 的概念。

举个例子，如果表中记录 1 亿，事务 A 把其中有几条记录上了行锁了，这时事务 B 需要给这个表加表级锁，如果没有意向锁的话，那就要去表中查找这一亿条记录是否上锁了。如果存在意向锁，那么假如事务 A 在更新一条记录之前，先加意向锁，再加 X 锁，事务 B 先检查该表上是否存在意向锁，存在的意向锁是否与自己准备加的锁冲突，如果有冲突，则等待直到事务 A 释放，而无须逐条记录去检测。事务 B 更新表时，其实无须知道到底哪一行被锁了，它只要知道反正有一行被锁了就行了。

意向锁的主要作用是处理行锁和表锁之间的矛盾，能够显示 “某个事务正在某一行上持有了锁，或者准备去持有锁”。

####	锁的模式
共享锁和排它锁都是行级锁。意向共享锁和意向排他锁是表级锁。意向共享锁和意向排他锁都是系统自动添加和自动释放的，整个过程无需人工干预。

1. 共享锁
共享锁（S 锁，Shared Lock）：又称读锁，允许一个事务去读数据集，阻止其他事务获得该数据集的排他锁。共享锁与共享锁可以同时使用。举例：若事务 T 对数据对象 A 加上 S 锁，则事务 T 可以读 A 但不能修改 A，其他事务只能再对 A 加 S 锁，而不能加 X 锁，直到 T 释放 A 上的 S 锁。这保证了其他事务可以读 A，但在 T 释放 A 上的 S 锁之前不能对 A 做任何修改。

2. 排他锁
排他锁（X 锁，Exclusive Lock）：又称写锁，允许获取排他锁的事务更新数据，阻止其他事务获得相同的数据集的共享锁和排他锁。排它锁与排它锁、共享锁都不兼容。举例：若事务 T 对数据对象 A 加上 X 锁，事务 T 可以读 A 也可以修改 A，其他事务不能再对 A 加任何锁，直到 T 释放 A 上的锁。

3. 意向共享锁
意向共享锁（IS 锁，Intention Shared Lock）：事务打算给数据行共享锁，事务在给一个数据行加共享锁前必须先取得该表的 IS 锁。

4. 意向排他锁
意向排他锁（IX 锁，Intention Exclusive Lock）：事务打算给数据行加排他锁，事务在给一个数据行加排他锁前必须先取得该表的 IX 锁。

IS 锁和 IX 锁的提出仅仅为了在之后加表级别的 S 锁和 X 锁时可以快速判断表中的记录是否被上锁，以避免用遍历的方式来查看表中有没有上锁的记录，也就是说其实 IS 锁和 IX 锁是兼容的，IX 锁和 IX 锁是兼容的。

意向锁之间不会发生冲突，但共享锁、排它锁、意向锁之间会发生冲突，表级别各种锁的兼容性如下表所示。

兼容性	X	IX	S	IS
X	不兼容	不兼容	不兼容	不兼容
IX	不兼容	兼容	不兼容	兼容
S	不兼容	不兼容	兼容	兼容
IS	不兼容	兼容	兼容	兼容
5. 自增锁
AUTO-INC Locks，自增锁，是一种特殊的表锁。当表有设置自增 auto_increment 列，在插入数据时会先获取自增锁，其它事务将会被阻塞插入操作，自增列 + 1 后释放锁，如果事务回滚，自增值也不会回退，所以自增列并不一定是连续自增的。（MySQL 从 5.1.22 版本开始，引入了一种可选的轻量级锁（mutex）机制来代替 AUTOINC 锁。见于参考文档 3）

6. 元数据锁
元数据锁（metadata lock），MySQL Server 控制，表级锁，是维护表元数据的数据一致性，保证在表上有活动事务（显式或隐式）的时候，不可以对元数据进行写入操作。从 MySQL5.5 版本开始引入了 MDL 锁，来保护表的元数据信息，用于解决或者保证 DDL 操作与 DML 操作之间的一致性。

对于引入 MDL，其主要解决了 2 个问题：

事务隔离问题，比如在可重复隔离级别下，会话 A 在 2 次查询期间，会话 B 对表结构做了修改，两次查询结果就会不一致，无法满足可重复读的要求。

数据复制的问题，比如会话 A 执行了多条更新语句期间，另外一个会话 B 做了表结构变更并且先提交，就会导致 slave 在重做时，先重做 alter，再重做 update 时就会出现复制错误的现象。

每执行一条 DML、DDL 语句时都会申请 MDL 锁，DML 操作需要 MDL 读锁，DDL 操作需要 MDL 写锁（MDL 加锁过程是系统自动控制，无法直接干预，读读共享，读写互斥，写写互斥），申请 MDL 锁的操作会形成一个队列，队列中写锁获取优先级高于读锁。

一旦出现 MDL 写锁等待，不但当前操作会被阻塞，同时还会阻塞后续该表的所有操作（不过在 MySQL5.6 的时候推出了 online ddl 机制，使得排队的 MDL 写锁进行降级，防止对 MDL 读锁的阻塞）。

加锁时机
SELECT xxx 查询语句正常情况下为快照读，只加元数据读锁，直到事务结束。

SELECT xxx LOCK IN SHARE MODE 语句为当前读，加 S 锁和元数据读锁，直到事务结束。

SELECT xxx FOR UPDATE 语句为当前读，加 X 锁和元数据读锁，直到事务结束。

DML 语句（INSERT、DELETE、UPDATE）为当前读，加 X 锁和元数据读锁，直到事务结束。

DDL 语句（ALTER、CREATE 等）加元数据写锁，且是隐式提交不能回滚，直到事务结束。

为什么 DDL 语句会隐式提交？因为 DDL 是数据定义语言，在数据库中承担着创建、删除和修改的重要的职责。一旦发生问题，带来的后果很可能是不可估量的。于是在每执行完一次后就进行提交，可以保证流畅性，数据不会发生阻塞，同时也会提高数据库的整体性能。

线上踩坑举例：由于 DDL 语句存在隐式提交，所以如果会话 A 开始了事务，进行了 DML 操作，然后进行了 DDL 操作，然后会话 A 回滚事务。此时会话 A 回滚的事务是一个空事务，因为 DDL 操作执行的时候会进行一次隐式提交

行锁锁住整表的场景
SQL 语句没有使用索引会把整张表锁住。例如事务里进行整表 update；用到前缀 like；字段没有加索引；数据库优化将索引查询转全表扫描等等。

Mysql 在 5.6 版本之前，直接修改表结构的过程中会锁表。

“查询每个表索引，并使用最佳索引，除非优化程序认为使用表扫描更有效。一次使用扫描是基于最佳索引是否跨越了表的 30％以上，但是固定百分比不再决定使用索引还是扫描。现在，优化器更加复杂，并且根据附加因素（如表大小，行数和 I / O 块大小）进行估计。” 见于参考文档 1。

实验操作下的加锁情况分析
以下结论基于 MySQL5.6，以 InnoDB 默认的 RR 级别来实验，只用来方便理解本文提到的锁机制。恐有纰漏，敬请谅解。

图片

死锁排查
INFORMATION_SCHEMA 提供对数据库元数据的访问、关于 MySQL 服务器的信息，如数据库或表的名称、列的数据类型或访问权限。其中有一个关于 InnoDB 数据库引擎表的集合，里面有记录数据库事务和锁的相关表。

MySQL 有关事务和锁的四条命令：

SELECT * FROM information_schema.INNODB_TRX; 命令是用来查看当前运行的所有事务。

SELECT * FROM information_schema.INNODB_LOCKs; 命令是用来查看当前出现的锁。

SELECT * FROM information_schema.INNODB_LOCK_waits; 命令是用来查看锁等待的对应关系。

show engine innodb status \G; 命令是用来获取最近一次的死锁信息。

在查询结果中可以看到是否有表锁等待或者死锁。如果有死锁发生，可以通过 KILL trx_mysql_thread_id 来杀掉当前运行的事务。

查询事务与锁的命令行
死锁是并发系统中常见的问题，同样也会出现在数据库 MySQL 的并发读写请求场景中。当两个及以上的事务，双方都在等待对方释放已经持有的锁或因为加锁顺序不一致造成循环等待锁资源，就会出现 “死锁”。常见的报错信息为 "Deadlock found when trying to get lock..."

MySQL 死锁问题排查的常见思路：

通过多终端模拟并发事务，复现死锁。

通过上面四条命令，查看事务与锁的信息。

通过 explain 可以查看执行计划。

发生死锁异常后，通过开启 InnoDB 的监控机制来获取实时的死锁信息，它会周期性（每隔 15 秒）打印 InnoDb 的运行状态到 mysqld 服务的错误日志文件中。

InnoDB 的监控较为重要的有标准监控（Standard InnoDB Monitor）和锁监控（InnoDB Lock Monitor），通过对应的系统参数可以将其开启。

set GLOBAL innodb_status_output=ON; 开启标准监控
set GLOBAL innodb_status_output_locks; 开启所监控
另外，MySQL 提供了一个系统参数 innodb_print_all_deadlocks 专门用于记录死锁日志，当发生死锁时，死锁日志会记录到 MySQL 的错误日志文件中。

另外，MySQL 提供了一个系统参数 innodb_print_all_deadlocks 专门用于记录死锁日志，当发生死锁时，死锁日志会记录到 MySQL 的错误日志文件中。

如何尽可能避免死锁
合理的设计索引，区分度高的列放到组合索引前面，使业务 SQL 尽可能通过索引定位更少的行，减少锁竞争。

尽量按主键 / 索引去查找记录，范围查找增加了锁冲突的可能性，也不要利用数据库做一些额外额度计算工作。比如有的程序会用到 select … where … order by rand(); 这样的语句，类似这样的语句用不到索引，因此将导致整个表的数据都被锁住。

大事务拆小。大事务更倾向于死锁，如果业务允许，将大事务拆小。

以固定的顺序访问表和行。比如两个更新数据的事务，事务 A 更新数据的顺序为 1，2; 事务 B 更新数据的顺序为 2，1。这样更可能会造成死锁。

降低隔离级别。如果业务允许，将隔离级别调低也是较好的选择，比如将隔离级别从 RR 调整为 RC，可以避免掉很多因为 gap 锁造成的死锁。



##  0x04	参考
-   [XORM 的七种武器](https://xorm.io/zh/blog/xorm%E7%9A%84%E4%B8%83%E7%A7%8D%E6%AD%A6%E5%99%A8/)
-	[『浅入深出』MySQL 中事务的实现](https://draveness.me/mysql-transaction/)
-	[一文理解 MySQL 的锁机制与死锁排查](https://mp.weixin.qq.com/s?__biz=MzUyNzgyNzAwNg==&amp;mid=2247484376&amp;idx=1&amp;sn=fa93e7481e03ba1cd60720586122a8d4&amp;scene=21#wechat_redirect)
-	[MySQL(十三)：小一万字 + 14 张图读懂锁机制](https://juejin.cn/post/7100060213297807368)
-	[一文理解 MySQL 的事务原则与事务隔离](https://cloud.tencent.com/developer/article/1839603)
-	[让你彻底搞懂 MySQL 中的锁机制与 MVCC]()
-	[MySQL 的锁和多版本并发控制 (MVCC) 的学习与分析]()
-	[深入浅出 mysql 锁]()
-	[【mysql】关于 innodb 中 MVCC 的一些理解](https://www.cnblogs.com/chenpingzhao/p/5065316.html)
-
一文理解 MySQL 的事务原则与事务隔离
-        [Innodb 中的事务隔离级别和锁的关系](https://tech.meituan.com/2014/08/20/innodb-lock.html)
-        [深入理解 MySQL：锁、事务与并发控制](https://cloud.tencent.com/developer/article/1401196)
-	[一文带你理解 MySQL 事务核心知识点](https://juejin.cn/post/7165896352797294623)
-	[MySQL 中的锁机制与 MVCC]()