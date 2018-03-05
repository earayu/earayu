# InnoDB存储引擎与事务模型FAQ

### 概述

>  本文将持续更新，内容包括mysql innodb存储引擎下的事务模型。本来想写一篇文章，但又感觉想表达的东西比较零碎，所以变成了一篇FAQ把问题记录下来。如非特别说明，本文讨论的语境为mysql下的InnoDB存储引擎。



### FAQ

#### 基础

1. mysql有哪些隔离级别？分别解决了什么问题？给出场景。

   这篇文章解释的还不错: [单机事务不同隔离级别的并发问题整理](https://zhuanlan.zhihu.com/p/33767823)

#### 锁

1. 什么是共享锁(S)、排他锁(X)？

   共享锁: 允许一个事务进行读**一行**，阻止其他事务获得相同数据集的**排他锁**。

   排他锁: 允许获得排他锁的事务更新**一行**，阻止其他事务取得相同数据集的**共享锁**和**排他锁**。

2. 什么时候会加行锁（S锁和X锁）？

   查询条件为唯一索引，并且查询结果唯一时使用行锁。

   参见[官方文档](https://dev.mysql.com/doc/refman/5.7/en/innodb-transaction-isolation-levels.html)： `locking depends on whether the statement uses a unique index with a unique search condition, or a range-type search condition.` 

3. 什么是意向共享锁(IS)、意向排他锁(IX)？

   为了允许行锁和表锁共存，实现多粒度锁机制，InnoDB 还有两种内部使用的意向锁（Intention Locks），这两种意向锁都是表锁。

   意向共享锁（IS）：事务打算给数据行加行共享锁，事务在给一个数据行加共享锁前必须先取得该表的 IS 锁。
   意向排他锁（IX）：事务打算给数据行加行排他锁，事务在给一个数据行加排他锁前必须先取得该表的 IX 锁。

   意向锁产生的主要目的是为了处理行锁和表锁之间的冲突，用于表明“某个事务正在某一行上持有了锁，或者准备去持有锁”。比如，表中的某一行上加了X锁，就不能对这张表加X锁。

   如果不在表上加意向锁，对表加锁的时候，都要去检查表中的某一行上是否加有行锁，多麻烦。

4. 在X隔离级别下进行Y操作，会按什么顺序加哪些锁？

5. ​

   ​

#### MVCC

1. MVCC是什么，解决了什么问题？

   MVCC：Multiversion concurrency control

   多版本并发控制（MVCC）是一种用来**解决读-写冲突**的无锁并发控制。并发的读-写操作会导致脏读和不可重复读问题，为了解决问题可以使用2PL（S锁和X锁），但这样会导致较多的锁争用。而MVCC作为一种无锁的方式，可以提高读-写操作的效率。

2. MVCC适用范围是什么？

   Read Committed和Repeatable Read隔离级别下使用MVCC

3. MVCC的原理是什么？

   innodb 只对读无锁，写操作仍是上锁的悲观并发控制，这也意味着，innodb 中只能见到因死锁和不变性约束而回滚，而见不到因为写冲突而回滚。

4. MVCC的快照什么时候生成？

   如果是Read Committed, 则每次读开始时建立新的快照。

   如果是Repeatable Read, 则第一次读时建立快照。

   所以，READ COMMITED隔离级别下会出现幻读。REPEATABLE READ隔离级别下不会出现幻读，但是会出现"幻写"，即写入时主键冲突。
   REPEATABLE READ隔离级别如果想查看到（其他事务提交的）最新数据，可以使用S锁：SELECT * FROM t FOR SHARE; 

   参见[官方文档](https://dev.mysql.com/doc/refman/5.7/en/innodb-consistent-read.html): `If the transaction isolation level is REPEATABLE READ (the default level), all consistent reads within the same transaction read the snapshot established by the first such read in that transaction.With READ COMMITTED isolation level, each consistent read within a transaction sets and reads its own fresh snapshot.`

   验证：
   T1:
   BEGIN;
   T2:
   BEGIN;
   DELETE FROM xx WHERE ID=3;
   COMMIT;
   T1:
   SELECT * FROM xx;（查询结果没有id为3的行。说明，即使T1比T2先开始，T1读到的快照也是T2commit后的。）

5. MVCC和乐观锁的区别是什么？







## 参考资料

mysql官方文档

[浅谈数据库并发控制 - 锁和 MVCC](https://draveness.me/database-concurrency-control) 好像没什么干货

https://liuzhengyang.github.io/2017/04/18/innodb-mvcc/