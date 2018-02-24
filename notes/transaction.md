1. SERIALIZABLE 隔离级别下
最先开始执行SQL语句的获得锁









## wiki

### X锁和S锁的目的、原理

InnoDB 实现了以下两种类型的行锁： 

共享锁（S）：允许一个事务去读一行，阻止其他事务获得相同数据集的排他锁。 
排他锁（X）：允许获得排他锁的事务更新数据，阻止其他事务取得相同数据集的共享读锁和排他写锁。

### 意向锁

为了允许行锁和表锁共存，实现多粒度锁机制，InnoDB 还有两种内部使用的意向锁（Intention Locks），这两种意向锁都是表锁：意向共享锁（IS）：事务打算给数据行加行共享锁，事务在给一个数据行加共享锁前必须先取得该表的 IS 锁。
意向排他锁（IX）：事务打算给数据行加行排他锁，事务在给一个数据行加排他锁前必须先取得该表的 IX 锁。

意向锁产生的主要目的是为了处理行锁和表锁之间的冲突，用于表明“某个事务正在某一行上持有了锁，或者准备去持有锁”。比如，表中的某一行上加了X锁，就不能对这张表加X锁。

如果不在表上加意向锁，对表加锁的时候，都要去检查表中的某一行上是否加有行锁，多麻烦。


什么时候获取什么锁？
已知serializable的select会获取共享锁，写会获取排他锁


怎么查看mysql当前加的锁



事物的传播行为



只有通过索引条件检索数据，InnoDB 才使用行级锁，否则，InnoDB 将使用表锁！
上面这句话是对的吗？


使用了间隙锁防止幻读（next-key locking）

阅读官方文档：InnoDB事物模型和锁


### 死锁
1. 哪些情况会发生死锁？
2. 写代码时要注意处理死锁的问题


### 事物日志
mysql写入时使用追加的方式写入事务日志，然后提交。然后异步持久化到磁盘。（对吗？）


### 自动提交
如果autocommit为0，在没有主动开启事务的情况下，会把每条SQL语句当成一个事务。
某些语句（比如ALTER TABLE）会自动提交已执行的语句，请查询官方文档。


### 锁相关语句
> 不属于SQL标准
* SELECT ... LOCK IN SHARE MODE
* SELECT ... FOR UPDATE


### MVCC
1. 适用于那些范围（隔离级别等）
只适用于READ COMMITTED 和 REPEATABLE READ， SERIALIZABLE会对所有读取的行都加S锁。
2. 目的是什么
行锁的升级，在许多情况下避免了加锁，提升性能。（需要更具体，严谨的表述）
3. 原理

READ COMMITTED 和 REPEATABLE READ时使用Consistent Nonlocking Reads, 不加锁


## 问题

什么时候使用行锁？
官方文档https://dev.mysql.com/doc/refman/5.7/en/innodb-transaction-isolation-levels.html说，unique index时使用行锁，其他gap锁，怎么回事？如果没有索引呢？


mvcc查询的时候，快照是什么时候生成的？
官方文档说第一次读时
https://dev.mysql.com/doc/refman/5.7/en/innodb-consistent-read.html
` all consistent reads within the same transaction read the snapshot established by the first such read in that transaction. `
验证：
T1:
BEGIN;
T2:
BEGIN;
DELETE FROM xx WHERE ID=3;
COMMIT;
T1:
SELECT * FROM xx;（查询结果没有id为3的行。说明，即使T1比T2先开始，T1读到的快照也是T2commit后的。）

MVCC模式下什么时候建立快照？
If the transaction isolation level is REPEATABLE READ (the default level), all consistent reads within the same transaction read the snapshot established by the first such read in that transaction.
With READ COMMITTED isolation level, each consistent read within a transaction sets and reads its own fresh snapshot.

所以，READ COMMITED隔离级别下会出现幻读。REPEATABLE READ隔离级别下不会出现幻读，但是会出现"幻写"，即写入时主键冲突。

来源：https://dev.mysql.com/doc/refman/5.7/en/innodb-consistent-read.html


## 文章

[Spring五个事务隔离级别和七个事务传播行为](https://yq.aliyun.com/articles/48893)