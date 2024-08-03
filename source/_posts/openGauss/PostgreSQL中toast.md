---
title: postgreSQL中TOAST类型
date: 2024-06-27 20:47:24
tags:
categories:
    - openGauss
---

本文参考：

- [PostgreSQL TOAST 技术理解](https://link.zhihu.com/?target=https%3A//cloud.tencent.com/developer/article/1004455)
- 《PostgreSQL修炼之道》

## **一、TOAST是什么？**

TOAST是**“The Oversized-Attribute Storage Technique”（超尺寸属性存储技术）**的缩写，主要用于存储一个大字段的值。

要理解TOAST，我们要先理解**页（BLOCK）**的概念。在PG中，页是数据在文件存储中的基本单位，其大小是固定的且只能在编译期指定，之后无法修改，默认的大小为8KB。同时，PG不允许一行数据跨页存储。那么对于超长的行数据，PG就会启动TOAST，将大的字段压缩或切片成多个物理行存到另一张系统表中（**TOAST表**），这种存储方式叫**行外存储**。

<!--more-->

## **二、使用TOAST**

只有特定的数据类型支持TOAST，因为那些整数、浮点数等不太长的数据类型是没有必要使用TOAST的。

另外，支持TOAST的数据类型必须是变长的。在变长类型中：

- 前4字节（32bit）称为**长度字**，长度字后面存储具体的内容或一个指针。
- 长度字的高2bit位是**标志位**，后面的30bit是**长度值**（表示值的总长度，包括长度字本身，以字节计）。
- 由长度值可知TOAST数据类型的逻辑长度最多是30bit，即1GB(2^30-1字节）之内。
- 前2bit的标志位，一个表示**压缩标志位**，一个表示是否**行外存储**，如果两个都是零，那么表示既未压缩也未行外存储。
- 如果设置了压缩标志标志位，表示该数值被压缩过（使用的是非常简单且快速的LZ压缩方法），使用前必须先解压缩。
- 如果设置了行外存储标志位，则表示该数值是在行外存储的。此时，长度字后面的部分只是一个指针，指向存储实际数据的TOAST表中的位置。如果两个标志位都设置了，那么这个行外数据也会被压缩。不管是哪种情况，长度字里剩下的30bit的长度值都表示数据的实际尺寸，而不是压缩后的长度。

![img](https://pic4.zhimg.com/80/v2-701222e7e79fce6f6e85a2a735265bbb_1440w.webp)

在 PG 中每个表字段有四种 TOAST 的策略：

- **PLAIN** —— 避免压缩和行外存储。只有那些不需要 TOAST 策略就能存放的数据类型允许选择（例如 int 类型），而对于 text 这类要求存储长度超过页大小的类型，是不允许采用此策略的。
- **EXTENDED** —— 允许压缩和行外存储。一般会先压缩，如果还是太大，就会行外存储。这是大多数可以TOAST的数据类型的默认策略。
- **EXTERNAL** —— 允许行外存储，但不许压缩。这让在text类型和bytea类型字段上的子串操作更快。类似字符串这种会对数据的一部分进行操作的字段，采用此策略可能获得更高的性能，因为不需要读取出整行数据再解压。
- **MAIN** —— 允许压缩，但不许行外存储。不过实际上，为了保证过大数据的存储，行外存储在其它方式（例如压缩）都无法满足需求的情况下，作为最后手段还是会被启动。因此理解为：尽量不使用行外存储更贴切。

首先创建一张 blog 表，查看它的各字段的TOAST策略：

```sql
postgres=# create table blog(id int, title text, content text);
CREATE TABLE
postgres=# \d+ blog;
                          Table "public.blog"
 Column  |  Type   | Modifiers | Storage  | Stats target | Description 
---------+---------+-----------+----------+--------------+-------------
 id      | integer |           | plain    |              | 
 title   | text    |           | extended |              | 
 content | text    |           | extended |              |
```

可以看到，interger 默认 TOAST 策略为 **PLAIN** ，而 text 为 **EXTENDED** 。

另外可以修改某个字段系统默认分配的TOAST策略，假如要将上面blog表中的content字段的TOAST策略改成**EXTERNAL**，就可以这样：

```sql
ALTER TABLE blog ALTER content SET STORAGE EXTERNAL; 
```

## **三、TOAST表的结构**

如果一个表中有任何一个字段是可以TOAST的，那么PostgreSQL会自动为该表建一个相关联的TOAST表，其OID存储在**pg_class**系统表的**reltoastrelid**记录里，行外的内容保存在TOAST表里。

查看blog表对应的TOAST表的OID：

```sql
postgres=# select relname,relfilenode,reltoastrelid from pg_class where relname='blog';
 relname | relfilenode | reltoastrelid 
---------+-------------+---------------
 blog    |       16441 |         16444
(1 row)
```

通过上述语句，我们查到 blog 表的 oid 为16441，其对应 TOAST 表的 oid 为16444（关于 oid 和 pg_class 的概念，请参考[PG官方文档](https://link.zhihu.com/?target=https%3A//www.postgresql.org/docs/9.5/static/index.html)），那么其对应 TOAST 表名则为： **pg_toast.pg_toast_16441**（注意这里是 blog 表的 oid ）。

行外存储被切成了多个Chunk块，每个Chunk块大约是一个BLOCK的四分之一大小，如果块大小为8KB（默认就是8KB），则Chunk大约为2KB（比2KB略小一点），每个Chunk都作为独立的行存储在TOAST表中。

TOAST表有三个字段：

- **chunk_id** —— 用来表示特定 TOAST 值的 OID ，可以理解为具有同样 chunk_id 值的所有行组成原表（这里的 blog ）的 TOAST 字段的一行数据。
- **chunk_seq** —— 用来表示该行数据在整个数据中的位置。
- **chunk_data** —— 该Chunk实际的数据。

我们看下上面的TOAST表**pg_toast.pg_toast_16441**的定义：

```sql
postgres=# \d+ pg_toast.pg_toast_16441;
TOAST table "pg_toast.pg_toast_16441"
   Column   |  Type   | Storage 
------------+---------+---------
 chunk_id   | oid     | plain
 chunk_seq  | integer | plain
 chunk_data | bytea   | plain
```

在chunk_id和chunk_seq上有一个唯一的索引，提供对数值的快速检索。

因此，一个表示行外存储的指针数据中包括了要查询的TOAST表的OID和特定数值的chunk_id（也是一个OID类型）。为了方便，指针数据还存储了逻辑数据的尺寸（原始的未压缩的数据长度）及实际存储的尺寸（如果使用了压缩，则两者不同）。加上头部的长度字，一个TOAST指针数据的总尺寸是20字节。

## **四、TOAST技术实践**

现在我们来实际验证下TOAST:

```sql
postgres=# insert into blog values(1, 'title', '0123456789');
INSERT 0 1
postgres=# select * from blog;
 id | title |  content   
----+-------+------------
  1 | title | 0123456789
(1 row)

postgres=# select * from pg_toast.pg_toast_16441;
 chunk_id | chunk_seq | chunk_data 
----------+-----------+------------
(0 rows)
```

可以看到因为 content 只有10个字符，所以没有压缩，也没有行外存储。然后我们使用如下 SQL 语句增加 content 的长度，每次增长1倍，同时观察 content 的长度，看看会发生什么情况？

```sql
postgres=# update blog set content=content||content where id=1;
UPDATE 1
postgres=# select id,title,length(content) from blog;
 id | title | length 
----+-------+--------
  1 | title |     20
(1 row)
postgres=# select * from pg_toast.pg_toast_16441;
 chunk_id | chunk_seq | chunk_data 
----------+-----------+------------
(0 rows)
```

反复执行如上过程，直到 pg_toast_16441 表中有数据：

```sql
postgres=# select id,title,length(content) from blog;
 id | title | length 
----+-------+--------
  1 | title | 327680
(1 row)

postgres=# select chunk_id,chunk_seq,length(chunk_data) from pg_toast.pg_toast_16441;
 chunk_id | chunk_seq | length 
----------+-----------+--------
    16439 |         0 |   1996
    16439 |         1 |   1773
(2 rows)
```

可以看到，直到 content 的长度为327680时（已远远超过页大小 8K），对应 TOAST 表中才有了2行 数据，且长度都是略小于2K，这是因为 extended 策略下，先启用了压缩，然后才使用行外存储。

下面我们将 content 的 TOAST 策略改为 EXTERNAL ，以禁止压缩。

```sql
postgres=# alter table blog alter content set storage external;
ALTER TABLE
postgres=# \d+ blog;
                          Table "public.blog"
 Column  |  Type   | Modifiers | Storage  | Stats target | Description 
---------+---------+-----------+----------+--------------+-------------
 id      | integer |           | plain    |              | 
 title   | text    |           | extended |              | 
 content | text    |           | external |              |
```

然后我们再插入一条数据：

```sql
postgres=# insert into blog values(2, 'title', '0123456789');
INSERT 0 1
postgres=# select id,title,length(content) from blog;
 id | title | length 
----+-------+--------
  1 | title | 327680
  2 | title |     10
(2 rows)
```

然后重复以上步骤，直到 pg_toast_16441 表中有数据：

```sql
postgres=# update blog set content=content||content where id=2;
UPDATE 1
postgres=# select id,title,length(content) from blog;
 id | title | length 
----+-------+--------
  2 | title | 327680
  1 | title | 327680
(2 rows)

postgres=# select chunk_id,chunk_seq,length(chunk_data) from pg_toast.pg_toast_16441;
 chunk_id | chunk_seq | length 
----------+-----------+--------
    16447 |         0 |   1996
    16447 |         1 |   1773
    16448 |         0 |   1996
    16448 |         1 |   1996
    16448 |         2 |   1996
    ....（省略）
    16448 |       164 |   1996
(167 rows)
```

因为不允许压缩，所以新的操作在TOAST表中生成了更多Chunk块行记录。通过以上操作得出以下结论：

- 如果策略允许压缩，则TOAST优先选择压缩。
- 不管是否压缩，一旦数据超过2KB左右，就会启用行外存储。
- 修改TOAST策略，不会影响现有数据的存储方式。

## **五、TOAST技术总结**

TOAST比那些更直接的方法（比如允许行值跨越多个页面）有更多优点。 假设查询通常是用相对比较短的键值进行匹配的，那么执行器的大多数工作都将使用主行项完成。TOAST过的属性的大值只是在把结果集发送给客户端的时候才被抽出来（如果它被选中）。 因此，主表要小得多，并且它的能放入到共享缓冲区中的行要比没有任何行外存储的方案更多。 排序集也缩小了，并且排序将更多地在内存里完成。一个小测试表明，一个典型的保存 HTML 页面以及它们的 URL 的表占用的存储（包括TOAST表在内）大约只有裸数据的一半，而主表只包含全部数据的 10%（URL和一些小的 HTML 页面）。与在一个非TOAST的对照表里面存储（把全部 HTML 页面裁剪成 7Kb 以匹配页面大小）同样的数据相比，运行时没有任何区别。