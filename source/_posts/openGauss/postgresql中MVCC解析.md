---
title: postgreSQL中MVCC解析
date: 2024-06-27 20:47:24
tags:
categories:
    - openGauss

---

> 参考：
>
> - [MVCC PostgreSQL 事务模型 多版本并发控制](jasongj.com)
> - [PostgreSQL如何实现MVCC (基于xmin、xmax、cmin、cmax)_postgresql cmin xmin-CSDN博客](https://blog.csdn.net/qq_31156277/article/details/90551978)

# 0x01 PostgreSQL中MVCC原理

PostgreSQL中，对于每一行数据（称为一个tuple），包含有4个隐藏字段。这四个字段是隐藏的，但可直接访问。

- xmin 在创建（insert）记录（tuple）时，记录此值为插入tuple的事务ID
- xmax 默认值为0.在删除tuple时，记录此值
- cmin和cmax 标识在同一个事务中多个语句命令的序列值，从0开始，用于同一个事务中实现版本可见性判断

下面通过实验具体看看这些标记如何工作。在此之前，先创建测试表

```sql
CREATE TABLE test 
(
  id INTEGER,
  value TEXT
);
```

<!--more-->

开启一个事务，查询当前事务ID（值为3277），并插入一条数据，xmin为3277，与当前事务ID相等。符合上文所述——插入tuple时记录xmin，记录未被删除时xmax为0

```shell
postgres=> BEGIN;
BEGIN
postgres=> SELECT TXID_CURRENT();
 txid_current 
--------------
         3277
(1 row)

postgres=> INSERT INTO test VALUES(1, 'a');
INSERT 0 1
postgres=> SELECT *, xmin, xmax, cmin, cmax FROM test;
 id | value | xmin | xmax | cmin | cmax 
----+-------+------+------+------+------
  1 | a     | 3277 |    0 |    0 |    0
(1 row)
```



继续通过一条语句插入2条记录，xmin仍然为当前事务ID，即3277，xmax仍然为0，同时cmin和cmax为1，符合上文所述cmin/cmax在事务内随着所执行的语句递增。虽然此步骤插入了两条数据，但因为是在同一条语句中插入，故其cmin/cmax都为1，在上一条语句的基础上加一。

```shell
INSERT INTO test VALUES(2, 'b'), (3, 'c');
INSERT 0 2
postgres=> SELECT *, xmin, xmax, cmin, cmax FROM test;
 id | value | xmin | xmax | cmin | cmax 
----+-------+------+------+------+------
  1 | a     | 3277 |    0 |    0 |    0
  2 | b     | 3277 |    0 |    1 |    1
  3 | c     | 3277 |    0 |    1 |    1
(3 rows)
```



将id为1的记录的value字段更新为’d’，其xmin和xmax均未变，而cmin和cmax变为2，在上一条语句的基础之上增加一。此时提交事务。

```shell
UPDATE test SET value = 'd' WHERE id = 1;
UPDATE 1
postgres=> SELECT *, xmin, xmax, cmin, cmax FROM test;
 id | value | xmin | xmax | cmin | cmax 
----+-------+------+------+------+------
  2 | b     | 3277 |    0 |    1 |    1
  3 | c     | 3277 |    0 |    1 |    1
  1 | d     | 3277 |    0 |    2 |    2
(3 rows)

postgres=> COMMIT;
COMMIT
```



开启一个新事务，通过2条语句分别插入2条id为4和5的tuple。

```shell
BEGIN;
BEGIN
postgres=> INSERT INTO test VALUES (4, 'x');
INSERT 0 1
postgres=> INSERT INTO test VALUES (5, 'y'); 
INSERT 0 1
postgres=> SELECT *, xmin, xmax, cmin, cmax FROM test;
 id | value | xmin | xmax | cmin | cmax 
----+-------+------+------+------+------
  2 | b     | 3277 |    0 |    1 |    1
  3 | c     | 3277 |    0 |    1 |    1
  1 | d     | 3277 |    0 |    2 |    2
  4 | x     | 3278 |    0 |    0 |    0
  5 | y     | 3278 |    0 |    1 |    1
(5 rows)
```



此时，将id为2的tuple的value更新为’e’，其对应的cmin/cmax被设置为2，且其xmin被设置为当前事务ID，即3278

```shell
UPDATE test SET value = 'e' WHERE id = 2;
UPDATE 1
postgres=> SELECT *, xmin, xmax, cmin, cmax FROM test;
 id | value | xmin | xmax | cmin | cmax 
----+-------+------+------+------+------
  3 | c     | 3277 |    0 |    1 |    1
  1 | d     | 3277 |    0 |    2 |    2
  4 | x     | 3278 |    0 |    0 |    0
  5 | y     | 3278 |    0 |    1 |    1
  2 | e     | 3278 |    0 |    2 |    2
```



在另外一个窗口中开启一个事务，可以发现id为2的tuple，xin仍然为3277，但其xmax被设置为3278，而cmin和cmax均为2。符合上文所述——若tuple被删除，则xmax被设置为删除tuple的事务的ID。

```shell
BEGIN;
BEGIN
postgres=> SELECT *, xmin, xmax, cmin, cmax FROM test;
 id | value | xmin | xmax | cmin | cmax 
----+-------+------+------+------+------
  2 | b     | 3277 | 3278 |    2 |    2
  3 | c     | 3277 |    0 |    1 |    1
  1 | d     | 3277 |    0 |    2 |    2
(3 rows)
```



这里有几点要注意

- 新旧窗口中id为2的tuple对应的value和xmin、xmax、cmin/cmax均不相同，实际上它们是该tuple的2个不同版本
- 在旧窗口中，更新之前，数据的顺序是2，3，1，4，5，更新后变为3，1，4，5，2。因为在PostgreSQL中更新实际上是将旧tuple标记为删除，并插入更新后的新数据，所以更新后id为2的tuple从原来最前面变成了最后面
- 在新窗口中，id为2的tuple仍然如旧窗口中更新之前一样，排在最前面。这是因为旧窗口中的事务未提交，更新对新窗口不可见，新窗口看到的仍然是旧版本的数据

提交旧窗口中的事务后，新旧窗口中看到数据完全一致——id为2的tuple排在了最后，xmin变为3278，xmax为0，cmin/cmax为2。前文定义中，xmin是tuple创建时的事务ID，并没有提及更新的事务ID，但因为PostgreSQL的更新操作并非真正更新数据，而是将旧数据标记为删除，并插入新数据，所以“更新的事务ID”也就是“创建记录的事务ID”。

```shell
 SELECT *, xmin, xmax, cmin, cmax FROM test;
 id | value | xmin | xmax | cmin | cmax 
----+-------+------+------+------+------
  3 | c     | 3277 |    0 |    1 |    1
  1 | d     | 3277 |    0 |    2 |    2
  4 | x     | 3278 |    0 |    0 |    0
  5 | y     | 3278 |    0 |    1 |    1
  2 | e     | 3278 |    0 |    2 |    2
(5 rows)
```

## MVCC保证原子性

原子性（Atomicity）指得是一个事务是一个不可分割的工作单位，事务中包括的所有操作要么都做，要么都不做。

对于插入操作，PostgreSQL会将当前事务ID存于xmin中。对于删除操作，其事务ID会存于xmax中。对于更新操作，PostgreSQL会将当前事务ID存于旧数据的xmax中，并存于新数据的xin中。换句话说，事务对增、删和改所操作的数据上都留有其事务ID，可以很方便的提交该批操作或者完全撤销操作，从而实现了事务的原子性。

## MVCC保证事物的隔离性

隔离性（Isolation）指一个事务的执行不能被其他事务干扰。即一个事务内部的操作及使用的数据对并发的其他事务是隔离的，并发执行的各个事务之间不能互相干扰。

标准SQL的事务隔离级别分为如下四个级别

| 隔离级别                     | 脏读   | 不可重复读 | 幻读   |
| :--------------------------- | :----- | :--------- | :----- |
| 未提交读（read uncommitted） | 可能   | 可能       | 可能   |
| 提交读（read committed）     | 不可能 | 可能       | 可能   |
| 可重复读（repeatable read）  | 不可能 | 不可能     | 可能   |
| 串行读（serializable）       | 不可能 | 不可能     | 不可能 |

从上表中可以看出，从未提交读到串行读，要求越来越严格。

注意，SQL标准规定，具体数据库实现时，对于标准规定不允许发生的，绝不可发生；对于可能发生的，并不要求一定能发生。换句话说，具体数据库实现时，对应的隔离级别只可更严格，不可更宽松。

事实中，PostgreSQL实现了三种隔离级别——未提交读和提交读实际上都被实现为提交读。

下面将讨论提交读和可重复读的实现方式

## MVCC隔离级别探索

无论提交成功或回滚的事务，xid 都会递增，对于repeatable read 和 serializable 隔离级别的事务，如果它的xid 小于另外一个事务的xid 。也就是xmin小于另外一个事务的xmin，那么另外一个事务对这个事务是不可见的。而read committed 则不会。

**设置一个读已提交隔离级别**

```shell
technology=# begin transaction isolation level read committed;
BEGIN
technology=# SELECT ival FROM tb_mvcc WHERE id = 1;
 id | ival 
----+------
 1 |    1
(1 row)
```

**另外一个事务对id=1 的进行修改，并commit**

```shell
technology=# BEGIN;
BEGIN
technology=# update tb_mvcc set ival = 11 where id = 1;
UPDATE 1
technology=# commit;
```

**在从第一个事务进行读取时，发现数据已经被修改，即在同一个事务中两次读取结果不一致。发现ival 被修改成了11**

```shell
technology=# SELECT id,ival FROM tb_mvcc WHERE id = 1;
id | ival 
----+------
 1 |   11
(1 row)
```

**完整信息如下所示：**

```shell
technology=# begin transaction isolation level read committed;
BEGIN
technology=# SELECT xmin,xmax,cmin,xmax,id,ival FROM tb_mvcc WHERE id = 1;
 xmin | xmax | cmin | xmax | id | ival 
------+------+------+------+----+------
  630 |    0 |    0 |    0 |  1 |    1
(1 row)

technology=# SELECT xmin,xmax,cmin,xmax,id,ival FROM tb_mvcc WHERE id = 1;
 xmin | xmax | cmin | xmax | id | ival 
------+------+------+------+----+------
  635 |    0 |    0 |    0 |  1 |   11
(1 row)

technology=# END;
COMMIT

```

### MVCC提交读

提交读只可读取其它已提交事务的结果。PostgreSQL中通过pg_clog来记录哪些事务已经被提交，哪些未被提交。具体实现方式将在下一篇文章《SQL优化（七） WAL PostgreSQL实现事务和高并发的重要技术》中讲述。

### MVCC可重复读

相对于提交读，重复读要求在同一事务中，前后两次带条件查询所得到的结果集相同。实际中，PostgreSQL的实现更严格，不紧要求可重复读，还不允许出现幻读。它是通过只读取在当前事务开启之前已经提交的数据实现的。结合上文的四个隐藏系统字段来讲，PostgreSQL的可重复读是通过只读取xmin小于当前事务ID且已提交的事务的结果来实现的。

# 0x02 PostgreSQL中的MVCC优势

- 使用MVCC，读操作不会阻塞写，写操作也不会阻塞读，提高了并发访问下的性能
- 事务的回滚可立即完成，无论事务进行了多少操作
- 数据可以进行大量更新，不像MySQL和Innodb引擎和Oracle那样需要保证回滚段不会被耗尽

# 0x03 PostgreSQL中的MVCC缺点

## 事务ID个数有限制

事务ID由32位数保存，而事务ID递增，当事务ID用完时，会出现wraparound问题。

PostgreSQL通过VACUUM机制来解决该问题。对于事务ID，PostgreSQL有三个事务ID有特殊意义：

- 0代表invalid事务号
- 1代表bootstrap事务号
- 2代表frozon事务。frozon transaction id比任何事务都要老

可用的有效最小事务ID为3。VACUUM时将所有已提交的事务ID均设置为2，即frozon。之后所有的事务都比frozon事务新，因此VACUUM之前的所有已提交的数据都对之后的事务可见。PostgreSQL通过这种方式实现了事务ID的循环利用。

## 大量过期数据占用磁盘并降低查询性能

由于上文提到的，PostgreSQL更新数据并非真正更改记录值，而是通过将旧数据标记为删除，再插入新的数据来实现。对于更新或删除频繁的表，会累积大量过期数据，占用大量磁盘，并且由于需要扫描更多数据，使得查询性能降低。

PostgreSQL解决该问题的方式也是VACUUM机制。从释放磁盘的角度，VACUUM分为两种

- VACUUM 该操作并不要求获得排它锁，因此它可以和其它的读写表操作并行进行。同时它只是简单的将dead tuple对应的磁盘空间标记为可用状态，新的数据可以重用这部分磁盘空间。但是这部分磁盘并不会被真正释放，也即不会被交还给操作系统，因此不能被系统中其它程序所使用，并且可能会产生磁盘碎片。
- VACUUM FULL 需要获得排它锁，它通过“标记-复制”的方式将所有有效数据（非dead tuple）复制到新的磁盘文件中，并将原数据文件全部删除，并将未使用的磁盘空间还给操作系统，因此系统中其它进程可使用该空间，并且不会因此产生磁盘碎片。