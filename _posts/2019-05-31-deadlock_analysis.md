---
title: InnoDB死锁案例分析
author: Yibin Yang
tag: mysql
---

InnoDB的锁机制是一个比较复杂的问题，要透彻理解得深入源码，这里先留个坑以后慢慢填。各种锁类型的简单介绍，可以参见[官方文档](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)(罗列的锁类型不完整)。**表级**锁类型的兼容矩阵参见[lock0priv.h](https://github.com/flyingice/mysql-server/blob/8.0/storage/innobase/include/lock0priv.h)。

- 案例一：自己最近碰到的线上MySQL数据库出现的一个死锁场景。

```wiki
2019-02-04 13:36:18 7f032aad8b00
*** (1) TRANSACTION:
TRANSACTION 5432527494, ACTIVE 0 sec inserting
mysql tables in use 98, locked 98
LOCK WAIT 4 lock struct(s), heap size 1184, 2 row lock(s), undo log entries 7
MySQL thread id 378824, OS thread handle 0x7f14d408eb00, query id 12692046831 ec2-15-188-239-93.eu-west-3.compute.amazonaws.com 172.31.21.192 db update
INSERT IGNORE INTO DATA_INDEX(NUMBER_A, DATE_A, NUMBER_B, DATE_B) VALUES('8217342438', '2019-02-04 13:36:00', '2100427091', '2018-06-07 11:26:00'),('8217342438', '2019-02-04 13:36:00', '2109469649', '2019-02-04 13:36:00'),('8217342438', '2019-02-04 13:36:00', '8216181543', '2018-09-29 10:54:00'),('8217342438', '2019-02-04 13:36:00', '2105084140', '2018-09-29 10:54:00'),('8217342438', '2019-02-04 13:36:00', '8216181542', '2018-09-29 10:54:00'),('8217342438', '2019-02-04 13:36:00', '8217342438', '2019-02-04 13:36:00')
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 1926 page no 10117 n bits 368 index `PRIMARY` of table `dbname`.`data_index` /* Partition `p51` */ trx table locks 2 total table locks 2  trx id 5432527494 lock mode S locks rec but not gap waiting lock hold time 0 wait time before grant 0
*** (2) TRANSACTION:
TRANSACTION 5432527493, ACTIVE 0 sec inserting
mysql tables in use 98, locked 98
5 lock struct(s), heap size 1184, 2 row lock(s), undo log entries 9
MySQL thread id 378862, OS thread handle 0x7f032aad8b00, query id 12692046815 ec2-15-188-239-93.eu-west-3.compute.amazonaws.com 172.31.21.192 db update
INSERT IGNORE INTO DATA_INDEX(NUMBER_A, DATE_A, NUMBER_B, DATE_B) VALUES('2109469649', '2019-02-04 13:36:00', '2100427091', '2018-06-07 11:26:00'),('2109469649', '2019-02-04 13:36:00', '8216181543', '2018-09-29 10:54:00'),('2109469649', '2019-02-04 13:36:00', '8217342438', '2019-02-04 13:36:00'),('2109469649', '2019-02-04 13:36:00', '8217342439', '2019-02-04 13:36:00'),('2109469649', '2019-02-04 13:36:00', '2105084140', '2018-09-29 10:54:00'),('2109469649', '2019-02-04 13:36:00', '8216181542', '2018-09-29 10:54:00'),('2109469649', '2019-02-04 13:36:00', '2109469649', '2019-02-04 13:36:00')
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 1926 page no 10117 n bits 368 index `PRIMARY` of table `dbname`.`data_index` /* Partition `p51` */ trx table locks 3 total table locks 2  trx id 5432527493 lock_mode X locks rec but not gap lock hold time 0 wait time before grant 0
*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 1926 page no 10933 n bits 112 index `PRIMARY` of table `dbname`.`data_index` /* Partition `p51` */ trx table locks 3 total table locks 2  trx id 5432527493 lock mode S locks rec but not gap waiting lock hold time 0 wait time before grant 0
*** WE ROLL BACK TRANSACTION (1)
```

分析死锁可以从死锁日志入手，但是日志信息非常有限，体现在下面几点：

1） `show engine innodb status`的结果只保留最近一次死锁的记录。

2）锁只会在产生冲突的时候被显示。

3）日志只显示被阻塞的语句。如果事务包含多个SQL操作，已经成功执行的语句不会记录在日志中。所以要知道事务包含哪些操作，需要对业务有所了解或者根据发生死锁的时间戳和线程id去binlog里找对应日志。

4）日志只显示两个事务，但是导致死锁发生的依赖关系环可能包含三个或者更多的事务。

data_index这张表有一个包含所有四列的联合主键，另外(NUMBER_B, DATE_B)是非唯一二级索引。线上所有业务均默认RR事务隔离级别。

为了简便，以下a表示NUMBER_A, DATE_A两列，b表示NUMBER_B, DATE_B两列。已知事务1需要插入数据(a, b) (b, a)，事务2插入(b, a), (a, b)的情况下，死锁日志能为我们提供一些线索。事务2占有一个REC_NOT_GAP的X锁，等待另一个REC_NOT_GAP的S锁。普通的INSERT操作一般不加锁（[官方文档](https://dev.mysql.com/doc/refman/8.0/en/innodb-locks-set.html)说*INSERT sets an exclusive lock on the inserted row. This lock is an index-record lock, not a next-key lock*，这里涉及到隐式锁的问题，坑先留着），如果检测到duplicate key，会加S锁。一个可能的场景是，事务2在insert(a, b)的时候发现(a, b)已经被事务1插入，于是**事务2为事务1**创建(a, b)上的X锁，自己则开始在(a, b)记录上等待S锁。对称地，事务1首先插入(a, b)，不加锁。紧接着插入(b, a)发现duplicate key，于是**事务1为事务2**创建(b, a)上的X锁，自己在(b, a)上等待S锁，所以日志中显示等待REC_NOT_GAP的S锁。

这是死锁产生最经典也是最容易分析的场景，两个事务各自持有一部分互斥资源，然后互相等待对方释放另外一部分资源。解决上面的问题，只需要改变数据插入顺序即可，即让两个事务都以(a, b) (b, a)的顺序进行更新，保证大家上锁的顺序一致从而避免死锁发生。

- 案例二：来自Percona官方技术博客：[One more InnoDB gap lock to avoid](https://www.percona.com/blog/2013/12/12/one-more-innodb-gap-lock-to-avoid/)

文章里只简要列举了场景，没有具体分析。以下是在本地重现问题之后的死锁日志：

```wiki
2019-06-02 16:50:15 0x70000591a000
*** (1) TRANSACTION:
TRANSACTION 1027249, ACTIVE 452 sec inserting
mysql tables in use 1, locked 1
LOCK WAIT 3 lock struct(s), heap size 1136, 2 row lock(s)
MySQL thread id 25, OS thread handle 123145395437568, query id 440 localhost root update
INSERT INTO preferences (numericId, receiveNotifications) VALUES ('1', '1')
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 30 page no 4 n bits 72 index PRIMARY of table `demo`.`preferences` trx id 1027249 lock_mode X insert intention waiting
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;

*** (2) TRANSACTION:
TRANSACTION 1027250, ACTIVE 242 sec inserting
mysql tables in use 1, locked 1
3 lock struct(s), heap size 1136, 2 row lock(s)
MySQL thread id 26, OS thread handle 123145395740672, query id 450 localhost root update
INSERT INTO preferences (numericId, receiveNotifications) VALUES ('2', '1')
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 30 page no 4 n bits 72 index PRIMARY of table `demo`.`preferences` trx id 1027250 lock_mode X
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 30 page no 4 n bits 72 index PRIMARY of table `demo`.`preferences` trx id 1027250 lock_mode X insert intention waiting
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;

*** WE ROLL BACK TRANSACTION (2)
```

两个session均在事务开始的时候试图UPDATE一个不存在的记录，于是两个事务都加上了对应GAP的X锁，这里范围重叠GAP锁不冲突。如果目标记录存在，则会给对应主键加上REC_NOT_GAP的X锁。接着session 1试着insert(1, 1), 发现与session 2的GAP锁冲突，于是给主键加上INSERT_INTENTION的X锁表明插入意图然后进入等待。对应地，如果session 2此时insert(2, 2)也会和session 1的GAP锁冲突。Innodb检测到循环等待，回滚权重较小的事务。

- 案例三：来自[官方文档](https://dev.mysql.com/doc/refman/8.0/en/innodb-locks-set.html)

```sql
CREATE TABLE t1 (i INT, PRIMARY KEY (i)) ENGINE = InnoDB;

# session 1
START TRANSACTION;
INSERT INTO t1 VALUES(1);

# session 2
START TRANSACTION;
INSERT INTO t1 VALUES(1);

# session 3
START TRANSACTION;
INSERT INTO t1 VALUES(1);

# session 1
ROLLBACK;
```

```wiki
2019-06-02 18:44:42 0x700005886000
*** (1) TRANSACTION:
TRANSACTION 1027283, ACTIVE 11 sec inserting
mysql tables in use 1, locked 1
LOCK WAIT 4 lock struct(s), heap size 1136, 2 row lock(s)
MySQL thread id 26, OS thread handle 123145395740672, query id 526 localhost root update
INSERT INTO t1 VALUES(1)
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 31 page no 4 n bits 72 index PRIMARY of table `demo`.`t1` trx id 1027283 lock_mode X insert intention waiting
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;

*** (2) TRANSACTION:
TRANSACTION 1027284, ACTIVE 5 sec inserting
mysql tables in use 1, locked 1
4 lock struct(s), heap size 1136, 2 row lock(s)
MySQL thread id 27, OS thread handle 123145395134464, query id 528 localhost root update
INSERT INTO t1 VALUES(1)
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 31 page no 4 n bits 72 index PRIMARY of table `demo`.`t1` trx id 1027284 lock mode S
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 31 page no 4 n bits 72 index PRIMARY of table `demo`.`t1` trx id 1027284 lock_mode X insert intention waiting
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;

*** WE ROLL BACK TRANSACTION (2)
```

session 2为session 1创建REC_NOT_GAP的X锁，自己等待REC_NOT_GAP的S锁。同理，session 3等待REC_NOT_GAP的S锁。session 1回滚之后释放X锁，session 2和session 3均获得S锁。session 1回滚后对应主键标记删除。session 2发现插入位置已经被session 1挂上了S锁，于是创建INSERT_INTENTION的X锁进入等待。由于session 3也会采取相同动作，死锁被检测到，其中一个失败回滚。

假设上面的案例中session 1最终提交事务，session 2和session 3都会检测到duplicate key而出错回滚，不会产生死锁的问题。

- 更多的上锁案例

[InnoDB locks and deadlocks with or without index for different isolation level](https://www.percona.com/blog/2015/04/09/innodb-locks-deadlocks-without-index-different-isolation-level/)

- 结论

死锁即使在表结构和事务逻辑非常简单的情况下也可以发生。我们可以通过最佳实践减少死锁发生的几率，但是要完全避免是不可能的，所以业务层的设计必须考虑该场景。
