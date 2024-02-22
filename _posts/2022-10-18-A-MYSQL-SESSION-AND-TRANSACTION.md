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

事务和锁配合使用的主要原因是为了更好地解决并发访问下的数据一致性、完整性和隔离性问题。虽然锁机制可以解决并发访问的问题，但在某些场景下，仅仅依靠锁还不足以保证数据的完整性和一致性，事务可以保证一组操作的原子性，这意味着这组操作要么全部成功，要么全部失败。在事务中可以对多个 SQL 语句进行组合，使它们在一个原子操作中执行。这样，即使在并发访问的情况下，也能确保数据的一致性和完整性


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

##	0x01

####	SQL 语句执行流程

![]()

1.	MySQL 客户端向 MySQL server 发起请求，获取到一个连接请求。
2.	在 connector（连接管理器）创建连接，分配线程，并验证用户名、密码和库表权限等。
3.	判断是否打开了 query_cache（查询缓存），如果打开了，则进入第四步，否则进入第五步
4.	根据当前 SQL 语句，获取到 hash 值，通过 hash 查询 cache&buffer 中是否有在过期时间范围内的数据。有则直接返回。否则进入第五步。
5.	SQL 连接组件接收 SQL 语句，并将 SQL 语句分解成 MySQL 能够识别的数据结构（抽象语法树）。
6.	Optimizer（查询优化器）根据 SQL 语句的特征，生产查询路径树，并选举一条最优的查询路径。
7.	调用存储引擎接口，打开表，执行查询。同时检查存储引擎中是否有对应的缓存记录，如果有，则直接返回。如果没有就继续往下执行。
8.	到数据文件中寻找数据（如果有索引，会先通过索引查找）
9.	当查询到所需要的数据之后，先写入存储引擎缓存中。如果打开了 query_cache，也会同时写进去。
10.	返回数据给客户端，同时关闭表，线程和连接。

####	MVCC
FROM：MySQL 之事务的实现原理


##	0x0	Mysql 的锁

FROM:深入浅出mysql锁

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

##	0x	事务


##  0x0	参考
-   [XORM 的七种武器](https://xorm.io/zh/blog/xorm%E7%9A%84%E4%B8%83%E7%A7%8D%E6%AD%A6%E5%99%A8/)
-	[『浅入深出』MySQL 中事务的实现](https://draveness.me/mysql-transaction/)
-	[一文理解 MySQL 的锁机制与死锁排查](https://mp.weixin.qq.com/s?__biz=MzUyNzgyNzAwNg==&amp;mid=2247484376&amp;idx=1&amp;sn=fa93e7481e03ba1cd60720586122a8d4&amp;scene=21#wechat_redirect)
-	[MySQL(十三)：小一万字 + 14 张图读懂锁机制](https://juejin.cn/post/7100060213297807368)
-	[一文理解 MySQL 的事务原则与事务隔离](https://cloud.tencent.com/developer/article/1839603)
-	[让你彻底搞懂 MySQL 中的锁机制与 MVCC]()
-	[MySQL 的锁和多版本并发控制 (MVCC) 的学习与分析]()
-	[深入浅出 mysql 锁]()