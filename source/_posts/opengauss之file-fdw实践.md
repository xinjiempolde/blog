---
title: opengauss之file-fdw实践
date: 2022-04-18 10:28:40
tags:
    - openGauss
    - 数据库
categories:
    - openGauss
---

> 本文转载自[CSDN-架构师忠哥](https://blog.csdn.net/penriver/article/details/122260524)

# file_fdw介绍
`fdw (Foreign Data Wrapper) `是一种外部访问接口，被用来访问存储在数据库外部的数据，这些数据可以是外部的pg数据库，也可以`oracle`、`mysql`等数据库，甚至可以是文件。

本文讲解如何通过`file_fdw`，访问外部的数据文件

<!--more-->

使用时注意如下：

- 数据文件必须是COPY FROM可读的格式；具体可参照COPY语句的介绍。

- 访问这样的数据文件当前只是可读的。当前不支持对该数据文件的写入操作。

- CREATE FOREIGN TABLE 外表的表结构需要与指定的文件的数据保持一致。

- 当前openGauss会默认编译file_fdw，在initdb的时候会在pg_catalog schema中创建该插件。

  

# file_fdw实践
## 创建外部server

```sql
select * from pg_foreign_server ;
--默认已经存在，不用创建
--create extension file_fdw;
create server pgcsv foreign data wrapper file_fdw;
```

## 创建表
使用`file_fdw`创建的外部表的选项参见 `file_fdw`
通过filename选项指定要读取的文件，是必需的参数，且必须是一个绝对路径名。

```sql
create foreign table person (
    id integer,
    name varchar(20),
    departno integer,
    age integer
) SERVER pgcsv
OPTIONS ( filename '/tmp/persons.csv', format 'csv',header 'true', delimiter '#');
```

`persons.csv`文件的内容如下：

```
id#name#departno#age
1#张三#22#56
2#李四#22#56
2#王五#22#51
2#赵六#22#51
2#张三丰#22# 51
3#张无敌#1#111
```


## 数据查询
```sql
explain select * from person;
```


![在这里插入图片描述](img.singhe.art/0dad6640addc46b2890ad7541ad407ae.png)



```sql
select * from person;
```

![](img.singhe.art/select.png)

## 表删除

```sql
drop foreign table person
```

