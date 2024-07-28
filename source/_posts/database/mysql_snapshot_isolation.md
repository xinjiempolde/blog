---
title: mysql上验证快照隔离
date: 2022-05-31 18:41:25
tags:
    - database
    - mysql
categories:
    - database
---

> 本文在mysql上验证快照隔离的特性。参考[文章](http://t.zoukankan.com/xwdreamer-p-2297081.html)。

<!--more-->
# 创建数据库

```sql
create database demo;
use demo;
CREATE TABLE test
(
tid INT NOT NULL primary key,
tname VARCHAR(50) NOT NULL
);
INSERT INTO test VALUES(1,'version1');
INSERT INTO test VALUES(2,'version2');
```

![image-20220715113204595](http://img.singhe.art/image-20220715113204595.png)



# 连接一

```sql
USE demo;
START TRANSACTION WITH CONSISTENT SNAPSHOT;
UPDATE test SET tname='version3' WHERE tid=2;
SELECT * FROM test;
```

![image-20220715113440161](http://img.singhe.art/image-20220715113440161.png)

可以看到连接一中数据已被修改。



# 连接二

```sql
USE demo;
START TRANSACTION WITH CONSISTENT SNAPSHOT;
SELECT * FROM test;
```

![image-20220715113625028](http://img.singhe.art/image-20220715113625028.png)

由于事务一(连接一)还没有提交，因此连接二看到的是version2而不是version3。



# commit

![image-20220715114056294](http://img.singhe.art/image-20220715114056294.png)

即使连接一commit之后，连接二在当前事务中也不能看到最新的版本version3。



![image-20220715114239913](http://img.singhe.art/image-20220715114239913.png)

而当连接二commit退出当前事务后，便能够看到version3了。
