---
title: 关于InnoDB行锁的一点思考
tag: mysql
---

最近看到一篇详细总结InnoDB加锁机制的[文章](http://hedengcheng.com/?p=771)，写得非常漂亮，评论区也有一些很有参考价值的问题。刚好翻Stack Overflow看到一个关于MySQL锁的[问题](https://stackoverflow.com/questions/6066205/)。复现如下：

```shell
CREATE TABLE `t1` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `notid` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1;

INSERT INTO t1 (notid) VALUES(1), (2), (3), (4), (5), (6), (7), (8), (9), (10);

# transaction 1
BEGIN;
SELECT * FROM t1 WHERE id=5 FOR UPDATE;

# transaction 2
# case 1
SELECT * FROM t1 WHERE id!=5 FOR UPDATE;
# case 2
SELECT * FROM t1 WHERE id<5 FOR UPDATE;
# case 3
SELECT * FROM t1 WHERE notid!=5 FOR UPDATE;
# case 4
SELECT * FROM t1 WHERE notid<5 FOR UPDATE;
# case 5
SELECT * FROM t1 WHERE id<=4 FOR UPDATE;
```

事务1在未提交或回滚的情况下，事务2五种情况下的SELECT语句都会阻塞。下面先以case 2为例，查看事务状态和锁状态：

```shell
mysql> show engine innodb status;
------------
TRANSACTIONS
------------
Trx id counter 2152
Purge done for trx's n:o < 2116 undo n:o < 0 state: running but idle
History list length 8
LIST OF TRANSACTIONS FOR EACH SESSION:
---TRANSACTION 281479735847608, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 2151, ACTIVE 5 sec fetching rows
mysql tables in use 1, locked 1
LOCK WAIT 3 lock struct(s), heap size 1136, 5 row lock(s)
MySQL thread id 24, OS thread handle 123145559453696, query id 652 localhost root Sending data
select * from t1 where id < 5 for update
------- TRX HAS BEEN WAITING 5 SEC FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 2 page no 4 n bits 80 index PRIMARY of table `demo`.`t1` trx id 2151 lock_mode X waiting
Record lock, heap no 6 PHYSICAL RECORD: n_fields 4; compact format; info bits 0
 0: len 4; hex 80000005; asc     ;;
 1: len 6; hex 00000000080d; asc       ;;
 2: len 7; hex 82000001030144; asc       D;;
 3: len 4; hex 80000005; asc     ;;

------------------
---TRANSACTION 2150, ACTIVE 52 sec
2 lock struct(s), heap size 1136, 1 row lock(s)
MySQL thread id 22, OS thread handle 123145559756800, query id 649 localhost root
```

```shell
mysql> select * from performance_schema.data_locks;
+--------+----------------------------------+-----------------------+-----------+----------+---------------+-------------+----------------+-------------------+------------+-----------------------+-----------+---------------+-------------+-----------+
| ENGINE | ENGINE_LOCK_ID                   | ENGINE_TRANSACTION_ID | THREAD_ID | EVENT_ID | OBJECT_SCHEMA | OBJECT_NAME | PARTITION_NAME | SUBPARTITION_NAME | INDEX_NAME | OBJECT_INSTANCE_BEGIN | LOCK_TYPE | LOCK_MODE     | LOCK_STATUS | LOCK_DATA |
+--------+----------------------------------+-----------------------+-----------+----------+---------------+-------------+----------------+-------------------+------------+-----------------------+-----------+---------------+-------------+-----------+
| INNODB | 4759136952:1059:140424070172856  |                  2155 |        68 |       12 | demo          | t1          | NULL           | NULL              | NULL       |       140424070172856 | TABLE     | IX            | GRANTED     | NULL      |
| INNODB | 4759136952:2:4:2:140424090159640 |                  2155 |        68 |       12 | demo          | t1          | NULL           | NULL              | PRIMARY    |       140424090159640 | RECORD    | X             | GRANTED     | 1         |
| INNODB | 4759136952:2:4:3:140424090159640 |                  2155 |        68 |       12 | demo          | t1          | NULL           | NULL              | PRIMARY    |       140424090159640 | RECORD    | X             | GRANTED     | 2         |
| INNODB | 4759136952:2:4:4:140424090159640 |                  2155 |        68 |       12 | demo          | t1          | NULL           | NULL              | PRIMARY    |       140424090159640 | RECORD    | X             | GRANTED     | 3         |
| INNODB | 4759136952:2:4:5:140424090159640 |                  2155 |        68 |       12 | demo          | t1          | NULL           | NULL              | PRIMARY    |       140424090159640 | RECORD    | X             | GRANTED     | 4         |
| INNODB | 4759136952:2:4:6:140424090159984 |                  2155 |        68 |       12 | demo          | t1          | NULL           | NULL              | PRIMARY    |       140424090159984 | RECORD    | X             | WAITING     | 5         |
| INNODB | 4759136048:1059:140424070170952  |                  2154 |        67 |       12 | demo          | t1          | NULL           | NULL              | NULL       |       140424070170952 | TABLE     | IX            | GRANTED     | NULL      |
| INNODB | 4759136048:2:4:6:140424090155032 |                  2154 |        67 |       12 | demo          | t1          | NULL           | NULL              | PRIMARY    |       140424090155032 | RECORD    | X,REC_NOT_GAP | GRANTED     | 5         |
+--------+----------------------------------+-----------------------+-----------+----------+---------------+-------------+----------------+-------------------+------------+-----------------------+-----------+---------------+-------------+-----------+
8 rows in set (0.00 sec)
```

可以看到，在InnoDB默认的RR事务隔离级别下，事务2的最近读操作在id=5这行数据的X锁上等待直到超时。问题来了，为什么两个WHERE条件中的id=5会和id<5产生冲突？

根据开始提到的那篇博文中介绍的加锁机制，简单来说，两个事务中的SELECT都属于主键查询，理论上都只会给符合各自条件的行加锁，两者没有重叠区域，不应该会出现事务1阻塞事务2的情况。

会不会是因为表t1中的试验数据量太小导致优化器选择采用全索引扫描的方式查询结果，这样使得所有行都被上锁？

```shell
mysql> explain select * from t1 where id = 5 for update;
+----+-------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
| id | select_type | table | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra |
+----+-------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | t1    | NULL       | const | PRIMARY       | PRIMARY | 4       | const |    1 |   100.00 | NULL  |
+----+-------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.01 sec)

mysql> explain select * from t1 where id < 5 for update;
+----+-------------+-------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | t1    | NULL       | range | PRIMARY       | PRIMARY | 4       | NULL |    4 |   100.00 | Using where |
+----+-------------+-------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```

可以看到事务1用的index unique scan，事务2用的index range scan，因此可以排除是index full scan的原因。这个问题让我头疼了很久，最后才找到原因。

虽然事务2中的SQL查询非常简单，不过还是先来分析它的结构，方法具体可以参考[此文](http://hedengcheng.com/?p=577)。表t1中只有一个索引，即主键id，WHERE clause中只有唯一条件id<5。分解这个SQL可以知道，id<5属于index last key，该查询中不存在index first key，index filter和table filter。

首先明确一点，MySQL server层和引擎层之间的交互是逐行进行的，也就是row by row的方式。不考虑特殊情况，引擎找到下一条符合条件的记录后上锁立马返回，然后继续等待server层的指令。

对于上面事务2中的查询，id<5作为index last key，是搜索的终止条件；由于index first key不存在，所以server层第一次请求InnoDB引擎的时候，引擎首先定位到B+树叶子结点中的第一条记录，也就是id=1的那行数据，给这条索引记录加X锁然后返回。之后server每次调用引擎，引擎继续线性扫描定位到下一条记录(超出当前结点页面范围的情况下根据双向链表指针找到相邻叶子结点)，然后判断当前位置是否还落在目标搜索范围内（id<5)。由于索引是有序的，所以一旦碰到一条超出范围的index记录，搜索就可以立马结束。

上述过程的关键在于对结束条件的判断。C++中我们可以这样遍历STL容器：

```c++
vector<int> v = {1, 2, 3, 4, 5};
for(auto it = v.begin(); it != v.end(); it++) ....
```

迭代器在做++操作的时候，先移动到下一个相邻位置，判断是否到达it.end()这个sentinel。如果没有它可以继续往后移动，否则终止循环。然而要判断是否终止循环，是需要比较当前元素的。

回到刚才事务2中case 2查询的例子，当引擎定位到id=4这条index时id<5为真，上锁返回，没有任何问题；接下来server再次调用[row_search_for_mysql](https://github.com/flyingice/mysql-server/blob/8.0/storage/innobase/include/row0sel.h)，引擎定位到下条记录并尝试读取，然后判断id是否还小于5；如若为假，那么可以告知server层搜索结束。然而，引擎想要读取的当前记录（id=5）已经被transaction 1挂上了X锁。由于X锁具有排它性，于是transaction 2只能等待，这就解释了case 2阻塞的原因。

下面也简单解释一下其余四种情况挂起的原因：

case 1: id<>5这个条件是index fiter，无法确定搜索范围。引擎搜索到id=5这条索引时阻塞。

case 3: notid列无索引，引擎只能全表扫描，导致所有被扫描过的行所对应的primary index都被加X锁，到id=5时产生冲突。

case 4: 原因同case 3。

case 5: id<=4作为index last key，本质上和id<5没有区别，都需要读取id=5的index记录来判断是否终止扫描，因此冲突原因与case 2完全相同。
