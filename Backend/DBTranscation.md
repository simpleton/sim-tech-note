!-- TOC depth:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [Mysql 事务](#mysql-)
	- [结论](#)
	- [概念](#)
	- [原子性](#)
	- [隔离性（Ioslation Level）](#ioslation-level)
		- [锁机制](#)
			- [排他锁例子](#)
			- [表加锁解锁](#)
		- [事务所用到的表](#)
<!-- /TOC -->

# Mysql 事务
上一次的抢购出现了库存是 -1 的问题，说明我们lock ticket的逻辑是非排他的，临界区没有被锁住。今天仔细研究了下DB的事务，发现以前的理解有一些偏差。

## 结论
在查询库存的语句加入 `for update` 来让这一行加入悲观锁，其余同事在查询transcation会被hold住，有效fix读到脏库存的问题。

## 概念
事务的特征：原子性（Atomiocity）、一致性（Consistency）、隔离性（Isolation）和持久性（Durability），这四个特性简称ACID特性。

* 原子性：事务是数据库的逻辑工作单位，事务中包括的所有操作要么都做，要么都不做。

* 一致性：事务执行的结果必须是使数据库从一个一致性的状态变到另外一个一致性状态。

* 隔离性：一个事务的执行不能被其他事务干扰。即一个事务内部的操作及使用的数据对其他事务是隔离的，并发执行的各个事务之间互相不干扰。

* 持久性：一个事务一旦成功提交，对数据库中数据的修改就是持久性的。接下来其他的其他操作或故障不应该对其执行结果有任何影响。

## 原子性
首先被begin和commit（或rollback）包含的逻辑是原子的，即要不一起成功，要不一起失败。这一点我们的理解没有错误。

## 隔离性（Ioslation Level）
这一点是我们以前理解错的，对于我们使用的RDS的InnoDB默R认的Ioslation Level为READ COMMITED.

```doc
1) READ UNCOMMITED
SELECT的时候允许脏读，即SELECT会读取其他事务修改而还没有提交的数据。

2) READ COMMITED
SELECT的时候无法重复读，即同一个事务中两次执行同样的查询语句，若在第一次与第二次查询之间时间段，其他事务又刚好修改了其查询的数据且提交了，则两次读到的数据不一致。

3) REPEATABLE READ
SELECT的时候可以重复读，即同一个事务中两次执行同样的查询语句，得到的数据始终都是一致的。实现的原理是，在一个事务对数据行执行读取或写入操作时锁定了这些数据行。
但是这种方式又引发了幻想读的问题。因为只能锁定读取或写入的行，不能阻止另一个事务插入数据，后期执行同样的查询会产生更多的结果。

4) SERIALIZABLE
与可重复读的唯一区别是，默认把普通的SELECT语句改成SELECT …. LOCK IN SHARE MODE。即为查询语句涉及到的数据加上共享琐，阻塞其他事务修改真实数据。serializable模式中，事务被强制为依次执行。这是SQL标准建议的默认行为。
```

我们原先一直认为应为SERIALIZABLE, 导致我们读到的数据是不正确的。

### 锁机制
+ 共享锁（乐观锁）：由读表操作加上的锁，加锁后其他用户只能获取该表或行的共享锁，不能获取排它锁，也就是说只能读不能写
+ 排他锁（悲观锁）：由写表操作加上的锁，加锁后其他用户不能获取该表或行的任何锁，典型是mysql事务中的

其命令分别为：共享锁(share mode), 排他锁(for update)

这里我们需要的是排他锁，即当一个操作在lock的时候，其他事务应当被hold住，而不是读到以前的旧数据。

#### 排他锁例子
我们做一个小实现，开2个sql的terminal(分别为T1和T2).

*T1 RUN*:
```sql
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from t1 where id='3' for update;
[result]
```

*T2 RUN*:
```sql
mysql> select * from t1;
```
这一句查询是OK的

*T2 查询非锁定记录*
```sql
select * from t1 where id=2 for update;
```

*T2 查询锁定记录*
```sql
select * from t1 for update;
```
这样会被hold住

#### 表加锁解锁
+ LOCK TABLES tablename WRITE;
+ LOCK TABLES tablename READ;
+ UNLOCK TABLE;S

###事务所用到的表
information_schema
```sql
select * from innodb_trx;
select * from innodb_lock_waits;
select * from innodb_locks;
```
