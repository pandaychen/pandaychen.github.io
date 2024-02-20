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
-   事务（Transaction）：事务是数据库操作的一个逻辑单元，它是由一系列的 SQL 语句组成。事务具有 ACID 特性（原子性 / 一致性 / 隔离性 / 持久性），这些特性保证了数据库在并发操作和系统故障的情况下，仍然能够保持数据的一致性和完整性。事务可以在一个会话中执行，也可以跨多个会话执行


####    事务 && 会话 的区别
Session 是客户端与 MySQL 服务器之间的一个连接，它是数据库操作的基本单位；而 Transaction 是数据库操作的一个逻辑单元，它具有 ACID 特性，保证了数据库的一致性和完整性
-   生命周期：Session 的生命周期从客户端连接到服务器开始，直到客户端断开连接；而 Transaction 的生命周期从事务开始（如：START TRANSACTION）到事务结束（如：COMMIT 或 ROLLBACK）
-   范围：Session 是数据库操作的基本单位，可以包含多个事务；而 Transaction 是数据库操作的一个逻辑单元，由一系列的 SQL 语句组成
-   隔离性：Session 之间是相互隔离的，一个会话中的操作不会影响到其他会话；而 Transaction 之间的隔离性取决于事务隔离级别，不同的隔离级别会对并发事务产生不同的影响
-   上下文：Session 具有自己的上下文，例如变量、临时表等；而 Transaction 的上下文是在会话的基础上进行操作，例如锁、回滚段等

####	事务 && 锁的区别
事务和锁是数据库中两个非常重要的概念，它们在数据库的并发控制中起着重要的作用。

事务：事务是数据库管理系统执行过程中的一个逻辑单位，由一个有限的数据库操作序列构成。事务应该具有 4 个属性：原子性、一致性、隔离性、持久性。这四个属性通常被称为 ACID 特性。事务是用户定义的一个数据库操作序列，这些操作要么全做，要么全不做，是一个不可分割的工作单位。例如，银行转账工作：从一个账户扣款并把相应金额的款项加到另一个账户，这两个操作要么全做要么全不做。

锁：锁是用来解决并发问题的一种机制，主要是为了保证数据的一致性和完整性。锁的类型分为共享锁（读锁）和排他锁（写锁）。共享锁是指多个事务对同一数据可以共享一把锁，可以同时读取数据，但是不能修改数据。排他锁是指一旦一个事务获得了对数据的排他锁，其他事务就不能再对该数据加任何类型的锁。锁的粒度分为行锁和表锁，行锁锁定的是某一行数据，表锁锁定的是整个数据表。

总的来说，事务和锁的主要区别在于：

事务是数据库操作的一个执行单元，而锁是并发控制的一种手段。
事务是逻辑上的概念，主要解决操作的原子性问题；而锁是物理上的概念，主要解决资源的并发访问问题。
事务的触发是用户行为，而锁的触发是系统行为

事务和锁配合使用的主要原因是为了更好地解决并发访问下的数据一致性、完整性和隔离性问题。虽然锁机制可以解决并发访问的问题，但在某些场景下，仅仅依靠锁还不足以保证数据的完整性和一致性，事务可以保证一组操作的原子性，这意味着这组操作要么全部成功，要么全部失败。在事务中，我们可以对多个 SQL 语句进行组合，使它们在一个原子操作中执行。这样，即使在并发访问的情况下，也能确保数据的一致性和完整性


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

####	SQL语句执行流程
MySQL客户端向MySQL server发起请求，获取到一个连接请求。
在connector（连接管理器）创建连接，分配线程，并验证用户名、密码和库表权限等。
判断是否打开了query_cache（查询缓存），如果打开了，则进入第四步，否则进入第五步
根据当前SQL语句，获取到hash值，通过hash查询cache&buffer中是否有在过期时间范围内的数据。有则直接返回。否则进入第五步。
SQL连接组件接收SQL语句，并将SQL语句分解成MySQL能够识别的数据结构（抽象语法树）。
Optimizer（查询优化器）根据SQL语句的特征，生产查询路径树，并选举一条最优的查询路径。
调用存储引擎接口，打开表，执行查询。同时检查存储引擎中是否有对应的缓存记录，如果有，则直接返回。如果没有就继续往下执行。
到数据文件中寻找数据（如果有索引，会先通过索引查找）
当查询到所需要的数据之后，先写入存储引擎缓存中。如果打开了query_cache，也会同时写进去。
返回数据给客户端，同时关闭表，线程和连接。

####	MVCC机制


##  0x0	参考
-   [XORM 的七种武器](https://xorm.io/zh/blog/xorm%E7%9A%84%E4%B8%83%E7%A7%8D%E6%AD%A6%E5%99%A8/)
-	[『浅入深出』MySQL 中事务的实现](https://draveness.me/mysql-transaction/)
-	[一文理解 MySQL 的锁机制与死锁排查](https://mp.weixin.qq.com/s?__biz=MzUyNzgyNzAwNg==&amp;mid=2247484376&amp;idx=1&amp;sn=fa93e7481e03ba1cd60720586122a8d4&amp;scene=21#wechat_redirect)
-	[MySQL(十三)：小一万字 + 14 张图读懂锁机制](https://juejin.cn/post/7100060213297807368)
-	[一文理解MySQL的事务原则与事务隔离](https://cloud.tencent.com/developer/article/1839603)
-	[让你彻底搞懂 MySQL 中的锁机制与 MVCC]()
-	[MySQL的锁和多版本并发控制(MVCC)的学习与分析]()
-	[深入浅出mysql锁]()