---
title: openGauss启动流程
date: 2022-04-18 10:28:40
tags:
    - openGauss
    - 数据库
categories:
    - openGauss
---

main(main.cpp) -> PostmasterMain(postmaster.cpp)

先执行MOTIterateForeignScan，然后进行XACT_EVENT_COMMIT，最后会XACT_EVENT_RECORD_COMMIT

 For MOT, XACT_EVENT_COMMIT will just do the OCC validation。Actual commit (write redo and apply changes) will be done during XACT_EVENT_RECORD_COMMIT event.

# 类说明

TxnAccess: Cache manager for current transaction

Access: Holds data for single row access.

Sentinel: Primary/Secondary index sentinel.好像被用作key

Row: 一行数据

RowHeader：Row里面有RowHeader，好像是用于事务并发控制的



# MOT流程

## 查找

1. mot_fdw.cpp的MOTIterateForeignScan
2. TxnManager的RowLookup
3. TxnManager的AccessLookup
4. AccessManager的AccessLookup
5. 使用AccessManager的RowLookup获取Access
6. 使用Access的GetTxnRow获取Row



# 创建表

``` 
CreateCommand -》 DefineRelation
```

首先我们知道DefineRelation此函数是最终创建表结构的函数，最主要的参数是CreateStmt这个结构，该结构核心信息如下

```c++
typedef struct CreateStmt
{
    NodeTag     type;
    RangeVar   *relation;       /* relation to create */
    List       *tableElts;      /* column definitions (list of ColumnDef) */
    List       *inhRelations;   /* relations to inherit from (list of
                                 * inhRelation) */
    TypeName   *ofTypename;     /* OF typename */
    List       *constraints;    /* constraints (list of Constraint nodes) */
    List       *options;        /* options from WITH clause */
    OnCommitAction oncommit;    /* what do we do at COMMIT? */
    char       *tablespacename; /* table space to use, or NULL */
    bool        if_not_exists;  /* just do nothing if it already exists? */
} CreateStmt;
```

transformColumnDefination(parse_utilcmd.cpp)



```c++
typedef struct HeapTupleFields {
    ShortTransactionId t_xmin; /* inserting xact ID */
    ShortTransactionId t_xmax; /* deleting or locking xact ID */

    union {
        CommandId t_cid;           /* inserting or deleting command ID, or both */
        ShortTransactionId t_xvac; /* old-style VACUUM FULL xact ID */
    } t_field3;
} HeapTupleFields;

typedef struct DatumTupleFields {
    int32 datum_len_; /* varlena header (do not touch directly!) */

    int32 datum_typmod; /* -1, or identifier of a record type */

    Oid datum_typeid; /* composite type OID, or RECORDOID */

    /*
     * Note: field ordering is chosen with thought that Oid might someday
     * widen to 64 bits.
     */
} DatumTupleFields;

typedef struct HeapTupleHeaderData {
    union {
        HeapTupleFields t_heap;
        DatumTupleFields t_datum;
    } t_choice;

    ItemPointerData t_ctid; /* current TID of this or newer tuple */

    /* Fields below here must match MinimalTupleData! */

    uint16 t_infomask2; /* number of attributes + various flags */

    uint16 t_infomask; /* various flag bits, see below */

    uint8 t_hoff; /* sizeof header incl. bitmap, padding */

    /* ^ - 23 bytes - ^ */

    bits8 t_bits[FLEXIBLE_ARRAY_MEMBER]; /* bitmap of NULLs -- VARIABLE LENGTH */

    /* MORE DATA FOLLOWS AT END OF STRUCT */
} HeapTupleHeaderData;

typedef struct HeapTupleData {
    uint32 t_len;           /* length of *t_data */
    uint1 tupTableType = HEAP_TUPLE;
    int2   t_bucketId;
    ItemPointerData t_self; /* SelfItemPointer */
    Oid t_tableOid;         /* table the tuple came from */
    TransactionId t_xid_base;
    TransactionId t_multi_base;
#ifdef PGXC
    uint32 t_xc_node_id; /* Data node the tuple came from */
#endif
    HeapTupleHeader t_data; /* -> tuple header and data */
} HeapTupleData;

typedef HeapTupleData* HeapTuple;

typedef struct {
    /* XXX LSN is member of *any* block, not only page-organized ones */
    PageXLogRecPtr pd_lsn;    /* LSN: next byte after last byte of xlog
                               * record for last change to this page */
    uint16 pd_checksum;       /* checksum */
    uint16 pd_flags;          /* flag bits, see below */
    LocationIndex pd_lower;   /* offset to start of free space */
    LocationIndex pd_upper;   /* offset to end of free space */
    LocationIndex pd_special; /* offset to start of special space */
    uint16 pd_pagesize_version;
    ShortTransactionId pd_prune_xid;           /* oldest prunable XID, or zero if none */
    ItemIdData pd_linp[FLEXIBLE_ARRAY_MEMBER]; /* beginning of line pointer array */
} PageHeaderData;

typedef PageHeaderData* PageHeader;
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191228113139310.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3hpYW9oYWk5Mjh3dw==,size_16,color_FFFFFF,t_70)