---
title: postgreSQL建表流程分析
date: 2024-06-27 20:47:24
tags:
categories:
    - openGauss
---

# 表的种类

<!--more-->

```c++
#define       RELKIND_RELATION        'r'   /* ordinary table */
#define       RELKIND_INDEX           'i'   /* secondary index */
#define       RELKIND_SEQUENCE        'S'   /* sequence object */
#define       RELKIND_TOASTVALUE      't'   /* for out-of-line values */
#define       RELKIND_VIEW            'v'   /* view */
#define       RELKIND_MATVIEW         'm'   /* materialized view */
#define       RELKIND_COMPOSITE_TYPE  'c'   /* composite type */
#define       RELKIND_FOREIGN_TABLE   'f'   /* foreign table */
#define       RELKIND_PARTITIONED_TABLE 'p' /* partitioned table */
#define       RELKIND_PARTITIONED_INDEX 'I' /* partitioned index */
#define       RELPERSISTENCE_PERMANENT  'p' /* regular table */
#define       RELPERSISTENCE_UNLOGGED   'u' /* unlogged permanent table */
#define       RELPERSISTENCE_TEMP       't' /* temporary table */
/* default selection for replica identity (primary key or nothing) */
#define       REPLICA_IDENTITY_DEFAULT  'd'
/* no replica identity is logged for this relation */
#define       REPLICA_IDENTITY_NOTHING  'n'
/* all columns are logged as replica identity */
#define       REPLICA_IDENTITY_FULL     'f'
/*
 * an explicitly chosen candidate key's columns are used as replica identity.
 * Note this will still be set if the index has been dropped; in that case it
 * has the same meaning as 'n'.
 */
#define       REPLICA_IDENTITY_INDEX    'i'
```



# CreateStmt

```c++
typedef struct CreateStmt
{
	NodeTag		type;
	RangeVar   *relation;		/* relation to create */	
	List	   *tableElts;		/* column definitions (list of ColumnDef) */
	List	   *inhRelations;	/* relations to inherit from (list of
								 * RangeVar) */
	PartitionBoundSpec *partbound;	/* FOR VALUES clause */ 
	PartitionSpec *partspec;	/* PARTITION BY clause */  
	TypeName   *ofTypename;		/* OF typename  类型名 */ 
	List	   *constraints;	/* constraints (list of Constraint nodes)  约束*/
	List	   *options;		/* options from WITH clause    with 语句参数 */
	OnCommitAction oncommit;	/* what do we do at COMMIT? */
	char	   *tablespacename; /* table space to use, or NULL 表空间  */
	char	   *accessMethod;	/* table access method 访问方法  */
	bool		if_not_exists;	/* just do nothing if it already exists? 表是否存在*/
} CreateStmt;
```

该结构用于保存对Create Table Statement语句查询解析生成的相关信息，如表信息、column列信息列表，访问方法（heap,btree）等

# RangeVar

该结构保存SQL语句中from 子句信息，如catalogname/ relname、是否采用别名（alias）和继承关系。

```c++
typedef struct RangeVar
{
	NodeTag		type;
	char	   *catalogname;	/* the catalog (database) name, or NULL */
	char	   *schemaname;		/* the schema name, or NULL */
	char	   *relname;		/* the relation/sequence name */
	bool		inh;			/* expand rel by inheritance? recursively act
								 * on children? */
	char		relpersistence; /* see RELPERSISTENCE_* in pg_class.h */
	Alias	   *alias;			/* table alias & optional column aliases */
	int			location;		/* token location, or -1 if unknown */
} RangeVar;
```



# Alias

为RangeVar指定别名.别名可能同时重命名了表列.

```c++
/*
 * Alias -
 *    specifies an alias for a range variable; the alias might also
 *    specify renaming of columns within the table.
 *    为RangeVar指定别名.别名可能同时重命名了表列.
 *
 * Note: colnames is a list of Value nodes (always strings).  In Alias structs
 * associated with RTEs, there may be entries corresponding to dropped
 * columns; these are normally empty strings ("").  See parsenodes.h for info.
 * 注意:colnames是Value节点(通常是字符串)链表.
 *      在与RTEs相关的Alias结构体中,可能有跟已删除的列对应的条目.
 *    这些通常是空字符串("").详细可参考parsenodes.h
 */
typedef struct Alias
{
  NodeTag   type;
  //别名
  char     *aliasname;    /* aliased rel name (never qualified) */
  //列别名链表
  List     *colnames;   /* optional list of column aliases */
} Alias;
```



# ColumnDef

该结构体保存创建表中的列定义信息，如列名、是否为空、压缩方法、是否定义默认值

```c++
/*
 * ColumnDef - column definition (used in various creates)
 *
 * If the column has a default value, we may have the value expression
 * in either "raw" form (an untransformed parse tree) or "cooked" form
 * (a post-parse-analysis, executable expression tree), depending on
 * how this ColumnDef node was created (by parsing, or by inheritance
 * from an existing relation).  We should never have both in the same node!
 *
 * Similarly, we may have a COLLATE specification in either raw form
 * (represented as a CollateClause with arg==NULL) or cooked form
 * (the collation's OID).
 *
 * The constraints list may contain a CONSTR_DEFAULT item in a raw
 * parsetree produced by gram.y, but transformCreateStmt will remove
 * the item and set raw_default instead.  CONSTR_DEFAULT items
 * should not appear in any subsequent processing.
 */
typedef struct ColumnDef
{
	NodeTag		type;
	char	   *colname;		/* name of column */
	TypeName   *typeName;		/* type of column */
	char	   *compression;	/* compression method for column */
	int			inhcount;		/* number of times column is inherited */
	bool		is_local;		/* column has local (non-inherited) def'n */
	bool		is_not_null;	/* NOT NULL constraint specified? */
	bool		is_from_type;	/* column definition came from table type */
	char		storage;		/* attstorage setting, or 0 for default */
	Node	   *raw_default;	/* default value (untransformed parse tree) */
	Node	   *cooked_default; /* default value (transformed expr tree) */
	char		identity;		/* attidentity setting */
	RangeVar   *identitySequence;	/* to store identity sequence name for
									 * ALTER TABLE ... ADD COLUMN */
	char		generated;		/* attgenerated setting */
	CollateClause *collClause;	/* untransformed COLLATE spec, if any */
	Oid			collOid;		/* collation OID (InvalidOid if not set) */
	List	   *constraints;	/* other constraints on column */
	List	   *fdwoptions;		/* per-column FDW options */
	int			location;		/* parse location, or -1 if none/unknown */
} ColumnDef;
```

# Constraint

该结构体用于保存约束信息，如主键、唯一索引、非空、外键和排他约束等信息。

```c++
typedef struct Constraint
{
	NodeTag		type;
	ConstrType	contype;		/* see above  约束类型 */

	/* Fields used for most/all constraint types: */
	char	   *conname;		/* Constraint name, or NULL if unnamed 约束名  */
	bool		deferrable;		/* DEFERRABLE?  可延迟 */
	bool		initdeferred;	/* INITIALLY DEFERRED? */
	int			location;		/* token location, or -1 if unknown */

	/* Fields used for constraints with expressions (CHECK and DEFAULT): */
	bool		is_no_inherit;	/* is constraint non-inheritable? 不可继承的约束*/
	Node	   *raw_expr;		/* expr, as untransformed parse tree 表达式，作为未转化解析树*/
	char	   *cooked_expr;	/* expr, as nodeToString representation */
	char		generated_when; /* ALWAYS or BY DEFAULT */

	/* Fields used for unique constraints (UNIQUE and PRIMARY KEY): */
	List	   *keys;			/* String nodes naming referenced key	 key列
								 * column(s) */
	List	   *including;		/* String nodes naming referenced nonkey  nonkey列
								 * column(s) */

	/* Fields used for EXCLUSION constraints: */
	List	   *exclusions;		/* list of (IndexElem, operator name) pairs */ 

	/* Fields used for index constraints (UNIQUE, PRIMARY KEY, EXCLUSION): */
	List	   *options;		/* options from WITH clause */   
	char	   *indexname;		/* existing index to use; otherwise NULL  索引名 */
	char	   *indexspace;		/* index tablespace; NULL for default  索引对应的表空间  */
	bool		reset_default_tblspc;	/* reset default_tablespace prior to
										 * creating the index */
	/* These could be, but currently are not, used for UNIQUE/PKEY: */
	char	   *access_method;	/* index access method; NULL for default  访问方法 */
	Node	   *where_clause;	/* partial index predicate      where子句信息	*/

	/* Fields used for FOREIGN KEY constraints: */
	RangeVar   *pktable;		/* Primary key table  主键信息  */
	List	   *fk_attrs;		/* Attributes of foreign key    外键属性列表*/
	List	   *pk_attrs;		/* Corresponding attrs in PK table  对应的主键属性*/
	char		fk_matchtype;	/* FULL, PARTIAL, SIMPLE */
	char		fk_upd_action;	/* ON UPDATE action */
	char		fk_del_action;	/* ON DELETE action */
	List	   *old_conpfeqop;	/* pg_constraint.conpfeqop of my former self */
	Oid			old_pktable_oid;	/* pg_constraint.confrelid of my former
									 * self */

	/* Fields used for constraints that allow a NOT VALID specification */
	bool		skip_validation;	/* skip validation of existing rows? */
	bool		initially_valid;	/* mark the new constraint as valid? */
} Constraint;
-------------------------------------------------------------------------------------
typedef enum ConstrType			/* types of constraints */
{
	CONSTR_NULL,				/* not standard SQL, but a lot of people
								 * expect it */
	CONSTR_NOTNULL,
	CONSTR_DEFAULT,
	CONSTR_IDENTITY,
	CONSTR_GENERATED,
	CONSTR_CHECK,
	CONSTR_PRIMARY,
	CONSTR_UNIQUE,
	CONSTR_EXCLUSION,
	CONSTR_FOREIGN,
	CONSTR_ATTR_DEFERRABLE,		/* attributes for previous constraint node */
	CONSTR_ATTR_NOT_DEFERRABLE,
	CONSTR_ATTR_DEFERRED,
	CONSTR_ATTR_IMMEDIATE
} ConstrType;
```

# Query

SQL语句完成词法、语法解析生成解析树，后进行查询分析与重写生成查询树，其元素为Query结构体

```c++
/*
 * Query -
 *	  Parse analysis turns all statements into a Query tree
 *	  for further processing by the rewriter and planner.
 * 对解析树进行分析生成查询树，继而供后续重写器和计划器处理
 *	  Utility statements (i.e. non-optimizable statements) have the
 *	  utilityStmt field set, and the rest of the Query is mostly dummy.
 *
 *	  Planning converts a Query tree into a Plan tree headed by a PlannedStmt
 *	  node --- the Query structure is not used by the executor.
 * 
 *  计划器将查询树转变成计划树，其head为 PlannedStmt节点
 */
typedef struct Query
{
	NodeTag		type;

	CmdType		commandType;	/* select|insert|update|delete|utility */

	QuerySource querySource;	/* where did I come from? */		

	uint64		queryId;		/* query identifier (can be set by plugins)  query 标识符*/ 

	bool		canSetTag;		/* do I set the command result tag?  */ 

	Node	   *utilityStmt;	/* non-null if commandType == CMD_UTILITY */  

	int			resultRelation; /* rtable index of target relation for 
								 * INSERT/UPDATE/DELETE; 0 for SELECT */ 

	bool		hasAggs;		/* has aggregates in tlist or havingQual  agg */ 
	bool		hasWindowFuncs; /* has window functions in tlist  是否有窗口函数 */
	bool		hasTargetSRFs;	/* has set-returning functions in tlist   是否设有returning functions */
	bool		hasSubLinks;	/* has subquery SubLink 子查询链  */
	bool		hasDistinctOn;	/* distinctClause is from DISTINCT ON  是否有distinct子句 */
	bool		hasRecursive;	/* WITH RECURSIVE was specified */
	bool		hasModifyingCTE;	/* has INSERT/UPDATE/DELETE in WITH */
	bool		hasForUpdate;	/* FOR [KEY] UPDATE/SHARE was specified 是否指定for update */
	bool		hasRowSecurity; /* rewriter has applied some RLS policy */

	bool		isReturn;		/* is a RETURN statement  return 查询*/

	List	   *cteList;		/* WITH list (of CommonTableExpr's) */

	List	   *rtable;			/* list of range table entries 范围表项 */
	FromExpr   *jointree;		/* table join tree (FROM and WHERE clauses) join tree */

	List	   *targetList;		/* target list (of TargetEntry) 投影列表*/

	OverridingKind override;	/* OVERRIDING clause */

	OnConflictExpr *onConflict; /* ON CONFLICT DO [NOTHING | UPDATE] 冲突*/

	List	   *returningList;	/* return-values list (of TargetEntry) 返回链表*/

	List	   *groupClause;	/* a list of SortGroupClause's */
	bool		groupDistinct;	/* is the group by clause distinct? */

	List	   *groupingSets;	/* a list of GroupingSet's if present */

	Node	   *havingQual;		/* qualifications applied to groups */

	List	   *windowClause;	/* a list of WindowClause's */

	List	   *distinctClause; /* a list of SortGroupClause's */

	List	   *sortClause;		/* a list of SortGroupClause's */

	Node	   *limitOffset;	/* # of result tuples to skip (int8 expr)  偏移*/
	Node	   *limitCount;		/* # of result tuples to return (int8 expr) 计数*/
	LimitOption limitOption;	/* limit type */

	List	   *rowMarks;		/* a list of RowMarkClause's */

	Node	   *setOperations;	/* set-operation tree if this is top level of
								 * a UNION/INTERSECT/EXCEPT query */

	List	   *constraintDeps; /* a list of pg_constraint OIDs that the query
								 * depends on to be semantically valid */

	List	   *withCheckOptions;	/* a list of WithCheckOption's (added
									 * during rewrite) */

	/*
	 * The following two fields identify the portion of the source text string
	 * containing this query.  They are typically only populated in top-level
	 * Queries, not in sub-queries.  When not set, they might both be zero, or
	 * both be -1 meaning "unknown".
	 */
	int			stmt_location;	/* start location, or -1 if unknown */
	int			stmt_len;		/* length in bytes; 0 means "rest of string" */
} Query;

```

# PlannedStmt

计划器会对上述的查询树进一步处理生成计划树

```c++
/* ----------------
 *		PlannedStmt node
 *
 * The output of the planner is a Plan tree headed by a PlannedStmt node.
 * PlannedStmt holds the "one time" information needed by the executor.
 * 
 * 计划器对此处理生成一个头部为 PlannedStmt node 计划树  
 * DDL语句其 commandType == CMD_UTILITY
 * For simplicity in APIs, we also wrap utility statements in PlannedStmt
 * nodes; in such cases, commandType == CMD_UTILITY, the statement itself
 * is in the utilityStmt field, and the rest of the struct is mostly dummy.
 * (We do use canSetTag, stmt_location, stmt_len, and possibly queryId.)
 * ----------------
 */
typedef struct PlannedStmt
{
	NodeTag		type;

	CmdType		commandType;	/* select|insert|update|delete|utility */ 

	uint64		queryId;		/* query identifier (copied from Query) */

	bool		hasReturning;	/* is it insert|update|delete RETURNING? */ 

	bool		hasModifyingCTE;	/* has insert|update|delete in WITH? */
 
	bool		canSetTag;		/* do I set the command result tag? */

	bool		transientPlan;	/* redo plan when TransactionXmin changes? */

	bool		dependsOnRole;	/* is plan specific to current role? */

	bool		parallelModeNeeded; /* parallel mode required to execute?  是否为并行模式 */

	int			jitFlags;		/* which forms of JIT should be performed  JIT 执行形式  */

	struct Plan *planTree;		/* tree of Plan nodes */  // plan nodes树

	List	   *rtable;			/* list of RangeTblEntry nodes */  // 范围链表

	/* rtable indexes of target relations for INSERT/UPDATE/DELETE */
	List	   *resultRelations;	/* integer list of RT indexes, or NIL */  // 范围表索引

	List	   *appendRelations;	/* list of AppendRelInfo nodes */ 

	List	   *subplans;		/* Plan trees for SubPlan expressions; note
								 * that some could be NULL */

	Bitmapset  *rewindPlanIDs;	/* indices of subplans that require REWIND */

	List	   *rowMarks;		/* a list of PlanRowMark's */

	List	   *relationOids;	/* OIDs of relations the plan depends on */ // relation oid

	List	   *invalItems;		/* other dependencies, as PlanInvalItems */ 

	List	   *paramExecTypes; /* type OIDs for PARAM_EXEC Params */ 

	Node	   *utilityStmt;	/* non-null if this is utility stmt */ 
 
	/* statement location in source string (copied from Query) */
	int			stmt_location;	/* start location, or -1 if unknown */
	int			stmt_len;		/* length in bytes; 0 means "rest of string" */
} PlannedStmt;
```

# 流程图及讲解

![建表流程图](https://img-blog.csdnimg.cn/54613b5533824e7eb4fd077c9e81fd65.png#pic_center)

transformCreateStmt函数是表创建真正的入口函数，其执行流程如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/bee6a79d047344b980c9f7e776528112.png#pic_center)

物理文件的创建以及系统表元数据等更新由DefineRelation函数实现：

![在这里插入图片描述](https://img-blog.csdnimg.cn/9731ba160532430eb761c1861afde93c.png#pic_center)

## transformCreateStmt

该函数对生成的计划树解析分析，返回操作节点链表

执行流程：
1）获取并检查命名空间权限；
2）构建并初始化CreateStmtContext上下文，在后续执行过程中进一步更新relation、约束、列属性等信息。
3）遍历表中所有的列，调用相应的处理函数获取列的属性、约束等信息，填充CreateStmtContext对应字段信息<普通列调用 transformColumnDefinition， 含有约束的调用 transformTableConstraint >。
4）后续进行预处理，检查约束合理性，最后返回 utility命令的操作节点链表。



## DefineRelation

该函数的功能是创建新的relation，包括物理文件、相应的内存relcache Entry和系统表元数据的更新
1）首先进行权限检查，确定当前用户是否有权限创建表；
2）对创建表语句中的WITH子句进行解析（transformRelOptions）；
3）调用heap_reloptions对参数进行合法验证。
4）调用 MergeAttributes 将继承的属性合并到表属性定义中；
5）根据表列信息调用 BuildDescForRelation函数生成元组描述符TupleDesc，该结构体记录了元组每列字段的详细信息（pg_attribute）
6）遍历定义链表中的每一个属性查看是否有默认值、压缩等信息；
7）在上述条件准备完善下调用 heap_create_with_catalog创建物理文件并在系统表中注册；
8）调用 AddRelationNewConstraints 处理表中新增的约束与默认值



## heap_create_with_catalog

1）首先进行参数校验检查，在同一命名空间是否存在相同名、pg_type系统表是否存在相同typename等；
2）调用 GetNewRelFileNode为此表分配一个全局唯一对象标识符Oid;

3) 结合表名、命名空间、对象标识符OID以及元组描述符等信息调用 heap_create 创建一个Relation 结构放入RelCache,后续根据此信息 table_relation_set_new_filenode（Relation）/ RelationCreateStorage(Index)创建物理文件。
4）紧接着调用 AddNewRelationType向pg_type系统表中注册该表的记录；
5）AddNewRelationTuple向pg_class 系统表中插入该表的相关信息；
6）AddNewAttributeTuples 将该表每个字段信息填充值 pg_attribute系统表；
7）最后通过 StoreConstraints 将约束和默认值等信息存储至 pg_constraint和pg_attrdef系统表中。



## heap_create

![在这里插入图片描述](https://img-blog.csdnimg.cn/aa22e495290b48cca4397b04558b98c8.png#pic_center)

1）首先进行安全性检查，不允许在系统表中创建relations，判断是否需要创建持久化文件等；
2）根据表名、表空间、表对象标识符和文件节点relfilenode等信息调用 RelationBuildLocalRelation在内存中构建Relation，并插入全局relcache 哈希表中；
3）结合relation类型调用相应的接口函数进行relation的创建，[普通表/TOAST/物化视图: table_relation_set_new_filenode，索引/序列：RelationCreateStorage];

  对于无需创建持久化的relation且用户指定表空间，则需要在 pg_tablespace 中注册对应的信息。



## RelationBuildLocalRelation

该函数目的是在内存中构建创建表的relcache Entry，并插入全局Relcache 哈希表中，用于加速后续对此表的访问。
1）如果不存在CacheMemoryContext，则创建此上下文，后续操作均在此上下文进行；
2）分配并初始化Relation结构体，结合入参的TueDesc填充Relation结构体中rd_att字段：字段属性的详细信息；
3）分配并根据入参填充Relation结构体中rd_att字段的Form_pg_class字段：表名、命名空间、字段属性/数目等；
4）调用 RelationInitLockInfo初始化relation描述符锁信息；
5）调用 RelationInitPhysicalAddr 初始化relation描述符对应的物理地址：spcNode/dbNode//RelNode [表空间/数据库/表]
6）将上述构建好的RelCache Entry插入全局ralcache 哈希表中，并增加该条目的引用计数

![在这里插入图片描述](https://img-blog.csdnimg.cn/f9bf4f64d38c421fb3278cc251816008.png#pic_center)

## RelationCreateStorage

物理文件的创建由磁盘管理器负责，pg中所有文件系统均调用这统一接口，而RelationCreateStorage 函数的实现就是通过调用这些函数进一步封装而成，期执行流程如下：
1）对于持久化的relation，设置字段表示need_wal，表明需要写WAL日志，对于临时relation或者unlogged relation无需此操作；
2）根据输入的RelFileNode调用 smgropen返回 SMgrRelation对象，不存在会创建一个；
3）结合上述返回的 SMgrRelation和ForkNumber号调用 smgrcreate创建relation的物理文件；
4）如需写WAL日志，调用 log_smgrcreate函数记录下此relation的实际物理信息；
5）最后将其添加至PendingRelDelete链表尾，在事务真正提交的时候如需回滚则可通过此信息将创建的文件删除，并返回 SMgrRelation对象。
