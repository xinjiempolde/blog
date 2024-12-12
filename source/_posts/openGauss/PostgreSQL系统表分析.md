---
title: postgreSQL系统表分析
date: 2024-06-27 20:47:24
tags:
categories:
    - openGauss

---

```shell
test=# select pg_relation_filenode('pg_attribute'); 
pg_relation_filenode    
----------------------
24621
(1 row)
```

这个命令可以找到表文件在哪里存着

> 转载自
>
> - [PostgreSQL 系统表体系 (syscache & recache) - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/623283855)

# 前言

系统表是整个 PostgreSQL 数据库存储体系中最重要的一部分数据，它们用来组织管理PostgreSQL 的数据空间，将用户自己定义的数据集合更好得以一个或者多个表组织起来。它们本质也是一个个表对象，相比于普通表是存储的元数据。

这里的元数据可以理解为描述数据的数据。比如，用户创建的表有 `(c1 int, c2 text)`两种列类型，这一些类型 `int, text` 会被单独存放在 `pg_type`的系统表中，同时 `c1, c2` 列名字则会被存放在 `pg_attribute`的系统表中，并和 pg_type 形成关联。这一些 `pg_type`, `pg_attribute` 类型的表可以建立对用户表的关系描述，所以它们可以被称为元数据。

<!--more-->

# OID

Oid 在 PostgreSQL 中被用来描述一个个数据表的逻辑对象，比如 `Relation`, `type`, `attr`, `namespace`等等，每创建一个对象都会为其分配一个属于自己的标识(Oid)。PG 也会通过 Oid 来在不同的数据表之间建立关联，也就是说有一些 对象是全局唯一的(pg_class 表的oid)。但是因为 Oid 是 `unsigned int`，所以当对象的数量超过42亿之后可能会有回卷，所以PG 对Oid的划分有一些自己的定义，比如预留16383 个Oid 作为全局唯一的对象标识，其他的都是给用户表使用，允许发生回卷。

```sql
postgres=# select oid,relname,relnamespace from pg_class where relname='pg_class';
 oid  | relname  | relnamespace
------+----------+--------------
 1259 | pg_class |           11
 
 test=# SELECT pg_relation_filepath('student');
 pg_relation_filepath 
----------------------
 base/16384/16387
(1 行记录)

test=# select oid,relname,relnamespace from pg_class where relname='student';
  oid  | relname | relnamespace 
-------+---------+--------------
 16387 | student |         2200
(1 行记录)


-- 三个 预定义好的Oid 类型
#define FirstGenbkiObjectId     10000
#define FirstUnpinnedObjectId   12000
#define FirstNormalObjectId     16384
```

可以看到student表的OID和存储tuple的文件名是一样的

接下来看看 PostgreSQL 内部非常重要的一些系统表，以及它们之间的关系模型，从而更好的帮助我们理解创建的一个用户表是如何被组织管理的。

# pg_database

```shell
postgres=# select oid,datname,datdba,encoding from pg_database;
  oid  |  datname  | datdba | encoding 
-------+-----------+--------+----------
     1 | template1 |     10 |        6
 12921 | template0 |     10 |        6
 12926 | postgres  |     10 |        6
 16385 | learn     |     10 |        6
(4 rows)
```

pg_database系统表用来管理一个database的对象属性

# pg_class

pg_class 系统表用来管理一个表的对象属性，就是存储在当前数据库的所有表在此时的固有属性信息都会被统一放在pg_class 系统表中，直接看其列属性的定义 `pg_class.h`：

> 因为过多，简单挑几个关键信息如下

```c
// pg_class 的唯一标识 
CATALOG(pg_class,1259,RelationRelationId)..

Oid         oid; // 当前表对象在 pg_class 的唯一标识，pg_class会以oid 为主键建索引，方便查找
NameData    relname; // relation 名字
Oid         relnamespace; // 所处的 pg_namespace oid，用来和 pg_namespace系统表建立关联
Oid         reltype; // 对象类型，用于和pg_type系统表建立关联
...
Oid         relam; // am 类型，比如是heap or 其他的，也是和 pg_amthod 建立管理
...
Oid         relfilenode; // 当前对象的物理文件名，pg 内部文件名都是以数字存在。
...
char        relpersistence; // 该对象的存储类型， 'p' 表示永久，即基本持久化类型; 
                            //'u' 表示 unlogged，不写wal.
                            // 't' 表示临时表，session 级别的生命周期
char        relkind; // 该对象的类型，'r'=普通表，'i'=索引，'v'=视图, 't'=toast 大value， 'c'=符合类型 等
int16       relnatts; // 该对象的属性列的个数
...
```

可以看到通过 **create type map as (string varchar, int_1 int);** **create table map_test (id int, value map);** 创建的表在 pg_class 中存储的属性信息 有两个，一个是 类型 `map` 的属性信息， 一个是表 `map_test`的属性信息。

```sql
-- 复合类型 map 的属性信息
postgres=# select oid,relname,relnamespace,reltype,relam,relfilenode,relpersistence,relkind,relnatts from pg_class where relname='map';
  oid  | relname | relnamespace | reltype | relam | relfilenode | relpersistence | relkind | relnatts
-------+---------+--------------+---------+-------+-------------+----------------+---------+----------
 27737 | map     |         2200 |   27739 |     0 |           0 | p              | c       |        2
-- 表map_test的属性信息
postgres=# select oid,relname,relnamespace,reltype,relam,relfilenode,relpersistence,relkind,relnatts from pg_class where relname='map_test';
  oid  | relname  | relnamespace | reltype | relam | relfilenode | relpersistence | relkind | relnatts
-------+----------+--------------+---------+-------+-------------+----------------+---------+----------
 27740 | map_test |         2200 |   27742 |     2 |       27740 | p              | r       |        2
```

当然，pg_class 本身也是一个对象，所以在 pg_class 中也会存储自己的对象属性信息。

```sql
postgres=# select oid,relname,relnamespace,reltype,relam,relfilenode,relpersistence,relkind,relnatts from pg_class where relname='pg_class';
 oid  | relname  | relnamespace | reltype | relam | relfilenode | relpersistence | relkind | relnatts
------+----------+--------------+---------+-------+-------------+----------------+---------+----------
 1259 | pg_class |           11 |      83 |     2 |           0 | p              | r       |       33
```

# pg_type

该系统表用于记录管理所有的类型定义，比如上面的 `create table map_test (id int, value map);` 建表过程中用到的类型 `int` 以及 复合类型 `map` 都会被存储到 `pg_type`中，而列名字 `id` 以及 `value` 则会被存储到的 `pg_attribute` 系统表中，这个后面会说。

PG 通过 `pg_class` 的对象属性描述的系统表 以及 `pg_type` 和 `pg_attribute` 两种对列属性描述的系统表 共同构造一个基本表的列信息。

接下来看看pg_type 的定义 `pg_type.h`，挑选几个简略定义如下:

```c
// pg_type的 固有对象 标识是1247
CATALOG(pg_type,1247,TypeRelationId)
Oid         oid; // 类型oid
NameData    typname; // 类型名字
...
int16       typlen; // 该类型的长度，对于变长类型则一直是-1，如果是-2则是以null 终止的c字符串。
...
char        typtype; // 该类型的基础类型。 'b'=基本类型，'c'=复合类型, 'd'=域类型, 'e'=枚举类型等
```

比如对于我们前面通过 `create type map as (string varchar, int_1 int);` 创建的类型，可以从 pg_type中看到其信息如下：

```sql
postgres=# select oid,typname,typlen,typtype from pg_type where typname='map';
  oid  | typname | typlen | typtype
-------+---------+--------+---------
 27739 | map     |     -1 | c
```

因为 `map` 是我们自己创建的类型，其在 PG 内部的Oid 会从 16384 之后开始生成，标识其属于用户对象。 对于普通的 `int`类型，其在数据库初始化的时候 以及在 pg_type 中预先创建好了，且 Oid也是提前分配好的，保证全局唯一：

```sql
postgres=# select oid,typname,typlen,typtype from pg_type where typname='int4';
 oid | typname | typlen | typtype
-----+---------+--------+---------
  23 | int4    |      4 | b
```

# pg_attribute

这个系统表描述的是一个表（对象）的每一个列属性的定义。在 pg_class中我们看到的是这个表对象的列的个数，但是具体每一个列 都是什么类型，名字是什么，长度是多少，是第几列等这样的列的描述信息则是会存储在 pg_attribute 系统表中。 其基本类型定义如下`pg_attribute.h`：

```c
CATALOG(pg_attribute,1249
Oid         attrelid; // 该列属于哪一个关系对象，关系对象的oid （一个数据库只能有一个关系对象的名字）
NameData    attname; // 该列的名称
Oid         atttypid; // 该列的类型， 指向 pg_type的一条类型
...
int16       attlen; // 该列的长度，同 pg_type中的 typlen，加速读取attr信息。
int16       attnum; // 该列的index，是 attrelid 的第几列。
...
```

比如我们查看 前面创建的 `test_map` 表的列描述信息如下：

```sql
postgres=# select oid from pg_class where relname='map_test';
  oid
-------
 27740
postgres=# select attrelid,attname,atttypid,attlen,attnum  from pg_attribute  where attrelid=27740;
 attrelid | attname  | atttypid | attlen | attnum
----------+----------+----------+--------+--------
    27740 | tableoid |       26 |      4 |     -6
    27740 | cmax     |       29 |      4 |     -5
    27740 | xmax     |       28 |      4 |     -4
    27740 | cmin     |       29 |      4 |     -3
    27740 | xmin     |       28 |      4 |     -2
    27740 | ctid     |       27 |      6 |     -1
    27740 | id       |       23 |      4 |      1
    27740 | value    |    27739 |     -1 |      2
```

可以看到 pg_attribute 还为 map_test 默认分配了一些默认不可见的属性列，用作 extension 时查看更细粒度的tuple信息。 用户自己创建的两列 `id` 和 `value` 则是有自己的typeid信息，可以从 `pg_type` 中看到其定义。

# 系统表关系

接下来还是通过上面两个简单语句：

```sql
create type map as (string varchar, int_1 int);
create table map_test (id int, value map);
```

看看最后创建的 `map_test`表信息 是如何由不同的系统表中的数据共同描述的。

![img](https://pic4.zhimg.com/80/v2-11941cecfb4e7a3892a2a997aa32755b_1440w.webp)

如上图，描述了整个创建过程中涉及到的 系统表信息（并不全面），主要的几个系统表如上。 当我们执行第一条语句 `create type map as (string varchar, int_1 int);` 按照上图给出的系统表，发生的事情如下： - 在pg_attribute 增加map的两个列属性，一个是 string，一个是int_1，并标识各自的 pg_type relid；创建好的 string和int_1 行各自的 `attrelid` 都会保存下来，用于指向pg_class 中的 map 对象对应的 oid。 - 在 pg_type 中创建 map类型，因为其是由两个基本类型 `int4` 和 `varchar` 组合而成，所以其类型是组合类型。其 `typerelid` 也是指向 pg_class 中 map 对象的 oid。 - 在pg_class中创建一行信息， 记录其指向的 pg_type 中的行oid 以及所属的 namespace信息。因为 map对象是 type类型，并不是relation，所以其内部不需要存储数据，也就不需要`relam`的智齿了。

当我们执行第二条语句 `create table map_test (id int, value map);` 就是继续在 pg_type 以及 pg_attribute 系统表中添加对应的行。 - 因为复合类型 map已经存在，则 map_test 表中的行只需要增加对应的 pg_attribute信息了，不需要额外创建pg_type行。增加的 `id` 和 `value`行只需要让 `atttypid` 指定 pg_type中的类型即可，id 是基本类型，value 则是已经创建好的复合类型 map。 - pg_class 中增加属于 `map_test`的行。因为其拥有符合类型的列，且是是一个普通表；所以拥有 toast 以及 am。

可以看到创建表的过程中需要有较多的系统表的读写，上图仅仅展示了写入的系统表 以及很小部分需要读区的系统表，实际执行 SQL 语句的过程中会有更多的对系统表的访问。

接下来我们看看 系统表的初始化以及访问链路。

# pg_filenode.map

```c++
#include <iostream>
typedef int int32;
typedef unsigned int uint32; /* == 32 bits */
typedef unsigned char uint8;   /* == 8 bits */
typedef unsigned short uint16; /* == 16 bits */
typedef unsigned long int uint64;
typedef unsigned int Oid;
#define MAX_MAPPINGS 62

typedef struct RelMapping {
    Oid         mapoid;         /* OID of a catalog */
    Oid         mapfilenode;    /* its filenode number */
} RelMapping;

typedef struct RelMapFile {
    int32 magic;        /* always RELMAPPER_FILEMAGIC */
    int32 num_mappings; /* number of valid RelMapping entries */
    RelMapping mappings[MAX_MAPPINGS];
    int32 crc; /* CRC of all above */
    int32 pad; /* to make the struct size be 512 exactly */
} RelMapFile;

int main() {
  // 读取文件
  RelMapFile *mapfile = (RelMapFile *)malloc(sizeof(RelMapFile));
  FILE *file = fopen("/home/singheart/project/cmake/pg_page/pg_filenode.map", "r");
  fread(mapfile, sizeof(RelMapFile), 1, file);
  fclose(file);

  printf("magic: %d\n", mapfile->magic);
  printf("Oid    fileid\n");
  for (int i = 0; i < MAX_MAPPINGS; ++i) {
    printf("%d -> %d\n", mapfile->mappings[i].mapoid, mapfile->mappings[i].mapfilenode);
  }
}
```

