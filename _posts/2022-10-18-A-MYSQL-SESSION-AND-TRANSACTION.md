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

##	0x02 MySQL 体系结构

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

####	redo log vs undo log

TODO

##	0x03	MySQL 事务

####	事务的并发问题
1、脏读：一个事务中访问到了另外一个事务未提交的数据，如果会话 `2` 更新 age 为 `10`，但是在 commit 之前，会话 `1` 此时获取 age，那么会获得的值就是会话 `2` 中尚未 commit 的值。如果会话 `2` 再次更新 age 或执行 rollback，而会话 `1` 已经拿到了 `age=10` 的值，这就是脏读

![dirty-read](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/mysql-practice/transaction/DirtyRead.jpeg)

2、不可重复读：是一个事务读取同一条记录 `2` 次，得到的结果不一致，由于在读取中间变更了数据，所以会话 `1` 事务查询期间的得到的结果就不一样了

![read](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/mysql-practice/transaction/Non-RepeatableRead.jpeg)

3、幻读：指一个事务读取 `2` 次，得到的记录条数不一致。由于在会话 `1` 之间插入了一个新的数据，所以得到的两次数据就不一样了

![phantom](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/mysql-practice/transaction/Phantom.jpeg)

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

隔离级别越高，越能保证数据的完整性和一致性，但是对并发性能的影响也越大。对于多数应用程序，可以优先考虑把数据库系统的隔离级别设为 Read Committed。如何解决事务并发问题？在MySQL、Oracle 中，为了性能的考虑并不是完全按照上面的 SQL 标准来实现的。数据库实现事务隔离的方式，基本可以分为以下两种：

1.	在读取数据前，对其加锁，阻止其他事务对数据进行修改
2.	不用加任何锁，通过一定机制生成一个数据请求时间点的一致性数据快照（Snapshot），并用这个快照来提供一定级别（语句级或事务级）的一致性读取。从用户的角度来看，数据库可以提供同一数据的多个版本，因此，这种技术叫做数据多版本并发控制（ＭultiVersion Concurrency Control）

####	MVCC
MySQL 通过 MVCC 机制，实现了事务的多版本控制。最早的数据库系统，只有读读之间可以并发，读写 / 写读 / 写写都要阻塞，引入多版本之后，只有写写之间相互阻塞，其他三种操作都可以并行，这样大幅度提高了 InnoDB 的并发度。

![mvcc-1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/mysql-practice/transaction/mvcc-1.png)

在 MySQL 中，主要通过这 `3` 种机制协同工作实现了 MVCC：

1、隐藏字段

在插入一条记录时，除了用户指定的数据字段外，MySQL 会默认额外添加如下三个隐藏字段：
-	`db_row_id`：随着新行插入而单调递增的占 `6` 字节的行 ID（并非顺序递增），主要作用是当表没有主键或唯一非空索引时，innodb 就会使用这个行 ID 自动产生聚簇索引。如果表有主键或唯一非空索引，聚簇索引就不会包含这个行 ID 了。`DB_ROW_ID` 跟 MVCC 关系不大。如果先没有指定主键，在表运行一段时间后，创建主键，那么会 MySQL 会进行聚簇索引的重建，会将新创建的主键或唯一索引作为聚簇索引，构建表
-	`db_trx_id`：事务 id，表示最近一次对本记录行作修改（`INSERT` 或 `UPDATE`）的事务 ID。至于 `DELETE` 操作，InnoDB 认为是一个 `UPDATE` 操作，不过会更新一个另外的删除位，将行表示为 `deleted`，并非真正删除
-	`db_roll_ptr`：回滚指针，指向该版本记录的前一个版本，实际指向的是当前记录行的 ** 上一版本 ** 的 undo log 信息

![]()

2、Read View

Read View 主要是用来做可见性分析的, 里面保存了 ** 对本事务不可见的其他活跃事务 **

1. 当一个事务被创建时，会创建该事务的 Read View（读视图），主要保存如下三个信息：

-	当前活跃事件列表（即当前还未提交的事务 id）
-	列表中最小事务 ID：小于当前事务 ID 的数据都是可见的，因为这些事务肯定已经 commit 或者 rollback
-	列表中最大事务 ID：大于当前事务 ID 的数据都是不可见的，需要通过回滚指针找到下一个版本

2. 一个事务通过 Read View 做可见性分析时，遵循如下规则：

![read-view](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/mysql-practice/transaction/readview.jpeg)

-	小于活跃事件列表中最小事务 id：该事务已提交，可见
-	大于活跃事件列表中最大事务 id：该事务 id 对应的事务在当前事务启动之后才开启，不可见
-	在事务 id 处于 `(min_trx_id, max_trx_id)` 时，就需要校验是否在当前活跃事件列表中：因为事务的执行时间是不确定的，在 `min_trx_id` 后面启动的事务，可能已经提交，这时也是可见的：
	-	不在活跃事件列表中：说明该事务在当前事务启动时，已经提交，可见
	-	在活跃事件列表中：说明该事务在当前事务启动时，还没有提交，处于活跃状态，哪怕在当前事务 启动之后提交，也是不可见的，不可见


3、Undo log

回滚日志，Undo log 中存储的是老版本数据，当一个事务需要读取记录行时，** 如果当前记录行不可见 **，可以顺着 undo log 链找到满足其可见性条件的记录行版本

Undo log 分类：在 InnoDB 里分为如下两类：
-	insert undo log：事务对 insert 新记录时产生的 undo log, 只在事务回滚时需要, 并且在事务提交后就可以立即丢弃
-	update undo log：事务对记录进行 delete 和 update 操作时产生的 undo log，不仅在事务回滚时需要，快照读也需要，只有当数据库所使用的快照中不涉及该日志记录，对应的回滚日志才会被 purge 线程删除

Undo log 的作用是：a）保证事务原子性，即可以通过 undo log 做回滚操作；b）实现数据多版本：即实现 MVCC 机制。结合上面的 Read View 来看，一个事务会在上述的链表中，最终找到一个在当前事务中可见的记录版本

一条记录在经过多个事务修改后，在数据库中，大致的一个数据存储结构如下图所示：

![]()

####	当前读 VS 快照读
1.	在 innodb 中创建一个新事务后，执行第一个 `select` 语句的时候，innodb 会创建一个快照（Read View），快照中会保存系统当前不应该被本事务看到的其他活跃事务 id 列表（即 trx_ids）
2.	当用户在这个事务中要读取某个记录行的时候，innodb 会将该记录行的 `DB_TRX_ID` 与该 Read View 中的一些变量进行比较，判断是否满足可见性条件（见 Read View 可见性分析）

-	快照读（snapshot read）：也叫普通读，即通过回滚指针构成的链表，读取到当前事务对应的事务 id** 可见的数据版本 **。普通的 `select` 语句（但是不包括 `select ... lock in share mode`，`select ... for update`）就是快照读；该方法是利用 MVCC 机制读取快照中的数据（不加锁的简单的 SELECT 都属于快照读）
-	当前读（current read）：即直接读取聚簇索引中存储的最新数据，** 不会通过记录中的回滚指针向下寻找 **，如 `select ... lock in share mode`，`select ... for update`，`insert`，`update`，`delete` 语句都是属于当前读
-	快照读是基于 MVCC 实现的，提高了并发的性能，降低开销
-	大部分业务代码中的读取都属于快照读
-	当前读读取的是记录的最新版本，读取时会对读取的记录进行加锁, 其他事务就有可能阻塞。加锁的 `SELECT`，或者对数据进行增删改都会进行当前读，`update`、`delete`、`insert` 语句虽然没有 `select`, 但是它们也会先进行读取，而且只能读取最新版本

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


####	重要：Mysql 是如何避免幻读的？
在 RR 级别中，如果仅仅通过 MVCC 的话，** 只能防止快照读的幻读发生，而不能防止当前读产生的幻读问题 **。因此 MySQL 通过 **MVCC + 行锁 ** 实现的 RR 事务隔离级别

1、快照读是如何避免幻读的？

根据前文描述，MySQL 在可重复读隔离级别下，是通过 MVCC 机制避免幻读的。MVCC 机制，通俗点可理解成在事务启动的时候对数据库拍了个快照 snapshot，它保留了那个时刻数据库的数据状态，那么该事务后续的读取都可以从此快照中获取，哪怕其他事务新加了数据，也不会影响到此快照中的数据，也就不会出现幻读了。结合前文的幻读 case，事务会话 `1` 在启动的时候创建了一个 snapshot，两次相同查询条件的 `SELECT` 结果是相同的，不会受到会话 `2` 的 `INSERT` 影响

2、当前读是如何避免幻读的？

普通读（快照读）实际上读取的是历史版本中的数据，但此方式读取在某些场景下是有问题的，假设需要 `update` 一个记录，但另一个事务已经 `delete` 这条记录并且提交事务了，这样会产生冲突，所以 `update` 的时候要知道最新的数据（当前读），那么MySQL 在可重复读隔离级别下是如何避免幻读的呢？由于需要最新状态的数据，那么试想能否在当前读的时候，对这段区间都加锁，让其他事务阻塞，无法插入？InnoDB 为了解决可重复读隔离级别使用当前读而造成的幻读问题，引入了间隙锁（gap lock）

![gap-lock-with-transaction.png](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/mysql-practice/transaction/gap-lock-with-transaction.png)


1.	事务 A SQL `SELECT ... for update` 是属于当前读，它会对锁定 id 范围  $$(2,+\infty)$$ 
2.	事务 B 插入了 `id=5` 的数据， $$(2,+\infty)$$  范围被锁定了，所以无法插入，阻塞

小结下：
-	针对快照读，InnoDB是通过 MVCC 方式解决了幻读
-	针对当前读，通过 next-key lock（记录锁）加 gap lock（间隙锁）方式解决了幻读





##  0x04	参考
-   [XORM 的七种武器](https://xorm.io/zh/blog/xorm%E7%9A%84%E4%B8%83%E7%A7%8D%E6%AD%A6%E5%99%A8/)
-	[『浅入深出』MySQL 中事务的实现](https://draveness.me/mysql-transaction/)
-	[一文理解 MySQL 的锁机制与死锁排查](https://mp.weixin.qq.com/s?__biz=MzUyNzgyNzAwNg==&amp;mid=2247484376&amp;idx=1&amp;sn=fa93e7481e03ba1cd60720586122a8d4&amp;scene=21#wechat_redirect)
-	[MySQL(十三)：小一万字 + 14 张图读懂锁机制](https://juejin.cn/post/7100060213297807368)
-	[一文理解 MySQL 的事务原则与事务隔离](https://cloud.tencent.com/developer/article/1839603)
-	[【mysql】关于 innodb 中 MVCC 的一些理解](https://www.cnblogs.com/chenpingzhao/p/5065316.html)
-   [Innodb 中的事务隔离级别和锁的关系](https://tech.meituan.com/2014/08/20/innodb-lock.html)
-   [深入理解 MySQL：锁、事务与并发控制](https://cloud.tencent.com/developer/article/1401196)
-	[一文带你理解 MySQL 事务核心知识点](https://juejin.cn/post/7165896352797294623)
-	[MySQL 日志：undo log、redo log、binlog 有什么用？](https://www.xiaolincoding.com/mysql/log/how_update.html)