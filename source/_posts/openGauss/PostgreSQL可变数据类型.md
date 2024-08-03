---
title: postgreSQL可变数据类型
date: 2024-06-27 20:47:24
tags:
categories:
    - openGauss
---

> 参考：
>
> - [postgreSQL可变数据类型 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/614890090)
> - [Postgresql Varlena 结构 | 学习笔记 (zhmin.github.io)](https://zhmin.github.io/posts/postgresql-varlena/)

# 0x00 可变长类型

## 0. Datum 的 typlen 的约定

如果Datum 类型是 “byVal”，则Datum表示一个值。如果Datum 类型不是”byVal“，则Datum 表示一个指针：

typlen > 0， Datum 就指向固定长度字节流，比如int类型
typlen == -1， Datum 指向一个变长 varlena 结构体，比如char，varchar类型
typlen == -2， Datum 指向一个C语言风格的字符串；

因此，查看所有的变长数据类型：

```sql
SELECT typname FROM pg_type WHERE typlen = -1
```

<!--more-->

## 1. 可变长类型varlena

```c++
struct varlena
{
	char		vl_len_[4];		/* Do not touch this field directly! */
	char		vl_dat[FLEXIBLE_ARRAY_MEMBER];	/* Data content is here */
};

#define VARHDRSZ		((int32) sizeof(int32))

/*
 * These widely-used datatypes are just a varlena header and the data bytes.
 * There is no terminating null or anything like that --- the data length is
 * always VARSIZE_ANY_EXHDR(ptr).
 */
typedef struct varlena bytea;
typedef struct varlena text;
typedef struct varlena BpChar;	/* blank-padded char, ie SQL char(n) */
typedef struct varlena VarChar; /* var-length char, ie SQL varchar(n) */
```

为了可以存储任意的数据，跳出CString中需要使用`'\0'`来作为终结符的弊端，PG采用了Varlena数据类型，由于不再有终结符表示数组的结束，所以必须存在一个长度字段指出当前的数组长度大小（Redis中的SDS也是类似的设计），以及是否经过了TOAST（是否压缩，是否行外存储等）。因此Varlena在数据开头引入了一个header。

注意`varlena`只是变长数据类型的基类，在具体使用中一般很少直接使用varlena类型。`varlena`还分为很多种子类，每种格式的定义都不相同。我们在使用之前，需要根据它的第一个字节，转换为它对应格式：

1. 第一个字节等于`000 00001`（小端序）, 那么就是`varattrib_1b_e`，用来存储和toast有关的 external 数据
2. 第一个字节的最高位等于1，且后7bit不全为0，那么就是`varattrib_1b`，用来存储小数据
3. 第一个字节的最高位等于0，那么就是`varattrib_4b`，可以存储不超过1GB的数据

另外需要注意的是，所有的Varlena类型中的vl_len_字段都是包括了header自身的大小，如果需要获得实际的数据大小需要减去header的长度。

### 1.1 varattrib_1b （small）

```c++
typedef struct
{
	uint8		va_header;
	char		va_data[FLEXIBLE_ARRAY_MEMBER]; /* Data begins here */
} varattrib_1b;
```

注意到`varattrib_1b`的`va_header`只有 8 位，最高位是标记位，值为 1。剩余的 7 位表示数据长度，所以`varattrib_1b`类型只是用于存储长度不超过127 byte 的小数据，`varattrib_1b`最常见的用法就是存放一个TOAST指针。

```text
-----------------------------------------
   tag   |         length               |   
-----------------------------------------
  1 bit  |        7 bit                 |
-----------------------------------------
```

### 1.2 varattrib_4b（flat）

这种数据形式一般被称为"flat"的形式，即**没有经过TOAST机制的行外存储**，也就是说其`va_data`中存储就是实际的变长数据。

对于未压缩的数据，使用`va_4byte`结构体存储，这是一个最原始的变长类型。观察`va_4byte`对象，其内部的定义和一个原始的`varlena`结构体一模一样，所以`varattrib_4b.va_4byte`是变长类型中唯一和TOAST没关系的类型。

```c++
/*
 * These structs describe the header of a varlena object that may have been
 * TOASTed.  Generally, don't reference these structs directly, but use the
 * macros below.
 *
 * We use separate structs for the aligned and unaligned cases because the
 * compiler might otherwise think it could generate code that assumes
 * alignment while touching fields of a 1-byte-header varlena.
 */
typedef union {
    struct /* Normal varlena (4-byte length) */
    {
        uint32 va_header;
        char va_data[FLEXIBLE_ARRAY_MEMBER];
    } va_4byte;
    struct /* Compressed-in-line format */
    {
        uint32 va_header;
        uint32 va_rawsize;                   /* Original data size (excludes header) */
        char va_data[FLEXIBLE_ARRAY_MEMBER]; /* Compressed data */
    } va_compressed;
} varattrib_4b;
```

对于压缩的数据，使用`va_compressed`结构体存储，其内部存储了数据压缩之前的大小。`varattrib_4b`使用`union`来表示这两种情况。`varattrib_4b`的头部`va_header`是一个32bit大小的类型，其第一位为0，用于与`varattrib_1b`类型进行区分，而其`va_header`中的第二高位用于区分数据是否被压缩，为1，则表示存储的数据是压缩的。为0，则表示存储的数据是未压缩过的。剩下的 30 位表示数据的长度，所以只能支持不超过 1GB ($2^{30} - 1$ bytes) 的数据。

```text
--------------------------------------------------
    tag    |    compress    |      length        |
--------------------------------------------------
   1 bit   |    1 bit       |      30 bit        |
-------------------------------------------------
```



### 1.3 varattrib_1b_e (toast)

`varattrib_1b_e`就是我们所说的“TOAST 指针”的父类，它并不存储数据，他的`va_data`数据段存放的是TOAST指针，可以是三种不同类型： `varatt_external`， `varatt_indirect`， `varatt_expanded`。这三种不同的TOAST指针具体在下一小节中会详细介绍。

varattrib_1b_e结构的首部header的第一个字节永远是 0x80（大端序） or 0x01 （小端序）

`varattrib_1b_e` 是 `varattrib_1b`类型的一个子集，其和 `varattrib_1b` 类型的唯一区别就是多了一个 `va_tag` 字段，其可以指出在`va_data`数据段中到底存放了哪一种的TOAST指针。同时需要注意的是varattrib_1b_e 类型和`varattrib_1b`一样，内部的字段都是未对齐的（因为va_data是一个char数组），因此如果需要访问对应的va_data字段，只能使用memcpy的方法，将其va_data范围内的数据copy到`varatt_external`， `varatt_indirect`， `varatt_expanded`结构体中，然后在对其进行访问，可以使用PG提供的宏`VARATT_EXTERNAL_GET_POINTER`实现这一点。

```c++
/* TOAST pointers are a subset of varattrib_1b with an identifying tag byte */
typedef struct
{
	uint8		va_header;		/* Always 0x80 or 0x01 */
	uint8		va_tag;			/* Type of datum */
	char		va_data[FLEXIBLE_ARRAY_MEMBER]; /* Type-specific data */
} varattrib_1b_e;
```

第二个字节`va_tag`表示类型，有下面四种。每种类型下，它的 `va_data`存储的格式都不是一样的：

// 不同种“Toast 指针”的种类标签

```c++
typedef enum vartag_external
{
 VARTAG_INDIRECT = 1,     // 属于 varatt_indirect 类型的TOAST指针
 VARTAG_EXPANDED_RO = 2,  // 属于 ExpandedObjectHeader 类型的 只读指针
 VARTAG_EXPANDED_RW = 3,  // 属于 ExpandedObjectHeader 类型的 读写指针
 VARTAG_ONDISK = 18  	  // 属于 varatt_external 类型的指针
} vartag_external;
```

下面展示一个创建 `varlena`数据的例子：

```c++
result = (struct varlena *) palloc(length + VARHDRSZ); // 分配堆内存
SET_VARSIZE(result, length + VARHDRSZ);   // 设置头部
memcpy(VARDATA(result), mydata, length); // 写入数据
```



## 2. TOAST 指针

刚才我们提到了varattrib_1b_e是所有TOAST指针的父类，而其下又有三种具体的，不同的类型TOAST指针，分别是`varatt_external`， `varatt_indirect`， `varatt_expanded`。需要注意的是，这里的“TOAST pointer”并不是c语言意义上的指针，而是表示的是指出被Toasted的数据实际存放位置的结构体。

### （1）varatt_external（行外存储TOAST指针）

`struct varatt_external`是一个传统的：TOAST指针“（也就在官方文档中提到的基于物理存储的TOAST指针）。 也就是说，其包含了行外存储在TOAST表中的Datum所需的信息。 仅当`va_extsize <va_rawsize-VARHDRSZ`时，才压缩数据。该结构不得包含任何padding，因为我们有时会使用`memcmp`比较这些结构体。

请再次注意，由于`varatt_external`并未对齐地存储在实际的元组中，因此，在查看这些字段之前，需要将元组中的数据 memcpy 到本地struct变量中，然后才可以查看里面的字段！ （我们之所以使用 memcmp ，是为了避免通过比较两个指针内部的字段值判断两个指针相等，而只检测两个TOAST指针（结构体）的值是否相等就可以了。）

/* TOAST指针的实现*/

```c++
/*
 * struct varatt_external is a traditional "TOAST pointer", that is, the
 * information needed to fetch a Datum stored out-of-line in a TOAST table.
 * The data is compressed if and only if the external size stored in
 * va_extinfo is less than va_rawsize - VARHDRSZ.
 *
 * This struct must not contain any padding, because we sometimes compare
 * these pointers using memcmp.
 *
 * Note that this information is stored unaligned within actual tuples, so
 * you need to memcpy from the tuple into a local struct variable before
 * you can look at these fields!  (The reason we use memcmp is to avoid
 * having to do that just to detect equality of two TOAST pointers...)
 */
typedef struct varatt_external
{
	int32		va_rawsize;		/* Original data size (includes header) */
	uint32		va_extinfo;		/* External saved size (without header) and compression method */
	Oid		va_valueid;		/* Unique ID of value within TOAST table */
	Oid		va_toastrelid;	        /* RelID of TOAST table containing it */
} varatt_external;
```

### （2）varatt_indirect（指向varlena的TOAST指针）

`truct varatt_indirect`只是一个`varlena`指针，可以指向`varatt_external`，`varatt_expanded`，或者是`varattrib_1b`，`varattrib_4b` 类型的原始数据。其指向的必须是存储在**内存中**而不是行外磁盘存储的toast关系中的Datum。 创建者就需要完全负责被引用的空间的生存周期， 只要该引用Datum指针存在。

请注意，就像struct varatt_external一样，此结构未对齐地存储在任何包含元组中。

```c++
typedef struct varatt_indirect

{

struct varlena *pointer; /* Pointer to in-memory varlena */

} varatt_indirect;
```

### （3）varatt_expanded （内存扩展存储的TOAST指针）

struct varatt_expanded是一个“ TOAST指针”，表示存储在**内存中**的行外数据， 采用某种特定于类型的，不一定物理连续的格式，便于计算而不是存储。 `src/include/utils/expandeddatum.h`中提供了 `ExpandedObjectHeader` 类型的操作API。在下面的expandeddatum的章节中会专门介绍这种数据类型。

```c++
typedef struct ExpandedObjectHeader ExpandedObjectHeader;

typedef struct varatt_expanded {

	ExpandedObjectHeader *eohptr;

} varatt_expanded;
```

## 3. Varlena和TOAST指针关系总结

### 3.1 继承结构关系

### 3.2 存储结构关系

## 4. 可变长类型的操作

上一小节介绍了三种可变长类型，在一般情况下，我们不能直接对这些结构体进行操作，因为数据并不是存放在一个栈上的结构体里面，这些结构体只是对存放的字节如何解释做出了定义，如果我们想要访问字段的值需要将Datum强制转换为可变长类型的指针，配合一系列的宏获取字段的值。

下面是一些常见的宏：

### 4.1 通用的宏

通用的宏：

```c++
VARSIZE_ANY(PTR)  	返回任意varlena指针指向的可变对象的长度（包括header）
VARSIZE_ANY_EXHDR(PTR)   返回任意varlena指针指向的可变对象的数据长度（不包括header）
VARDATA_ANY(PTR)  	返回任意varlena指针指向的可变对象的起始数据地址（不支持external or compressed-in-line Datum）
VARATT_IS_EXTENDED(PTR)  指针是否是扩展的类型，除了VARATT_IS_4B_U都是
```

设置varlena类型的长度

```c++
SET_VARSIZE(PTR, len)
SET_VARSIZE_SHORT(PTR, len)
SET_VARSIZE_COMPRESSED(PTR, len)
```

查看header的第一个字节：

```c++
VARATT_IS_COMPRESSED(PTR)  // 数据是否是压缩的
```

### 4.2 类型相关的宏

varattrib_4b相关：

```c++
VARDATA(PTR)  	// 获得varattrib_4b指针类型的数据起始地址
VARSIZE(PTR)	// 获得varattrib_4b指针类型指向的可变对象的长度（包括header）
```

varattrib_1b相关：

```c++
VARATT_IS_SHORT(PTR) // 指针是否是 varattrib_1b 的
VARSIZE_SHORT(PTR)	// 同上
VARDATA_SHORT(PTR)	// 同上
```

varattrib_1b_e相关：

```c++
VARATT_IS_EXTERNAL(PTR)  指针是否是 varattrib_1b_e 的
SET_VARTAG_EXTERNAL(PTR, tag)	设置 varattrib_1b_e 指针类型的va_tag字段
VARTAG_EXTERNAL(PTR)	获得 varattrib_1b_e 指针类型的va_tag字段的起始地址
VARSIZE_EXTERNAL(PTR)	同上
VARDATA_EXTERNAL(PTR)	同上
VARATT_IS_EXTERNAL_ONDISK(PTR)        指针是否是 VARTAG_ONDISK 
VARATT_IS_EXTERNAL_INDIRECT(PTR)     指针是否是 VARTAG_INDIRECT 
VARATT_IS_EXTERNAL_EXPANDED_RO(PTR)  指针是否是VARTAG_EXPANDED_RO
VARATT_IS_EXTERNAL_EXPANDED_RW(PTR)  指针是否是VARTAG_EXPANDED_RW
```

## 5. 内存expanded的数据类型

这部分的内容主要在`expandedaatum.h`

复杂的数据类型，尤其是诸如`array`和`record`之类的容器类型，通常在磁盘上具有紧凑的存储形式，但并利于修改。而且，当我们修改它们时可能会非常低效，因为我们不得不重新复制其余所有值。因此，PG提供了“扩展（expanded）”的概念，这一概念属于内存TOAST技术的一种，这种存储格式仅在**内存**中使用，内存扩展类型针对计算而非存储进行了更多优化。稍后我们会发现，`Array`类型的`expanded`结构是如何加速下标访问的。

我们将出现在磁盘上的格式称为数据类型的“扁平（flattened）”表示形式，flattened的存储格式是连续的字节blob（块）。但是该类型也可以具有`expanded`表示形式用来加速内存中的计算，比如访问或者排序。如果一个数据类型支持`expanded`的表示类型，其必须提供将`expanded`的表示形式转换回`flat`形式的方法。

PG中所有支持`expanded`的数据结构都必须包含`ExpandedObjectHeader`，其定义如下所示：

```c++
struct ExpandedObjectHeader
{
 /* Phony varlena header Phony varlena标头 */
 int32  vl_len_;  /* 对于 ExpandedObjectHeader 对象来说，其vl_len域永远是 -1 */
 const ExpandedObjectMethods *eoh_methods;       // 扩展对象需要实现函数指针结构体，一个是获取flat格式方法，一个是获取flat size的方法
 MemoryContext eoh_context;        // 包含此 header 和 辅助数据 的内存上下文
 char  eoh_rw_ptr[EXPANDED_POINTER_SIZE];	// 读写指针（TOAST指针结构体）
 char  eoh_ro_ptr[EXPANDED_POINTER_SIZE];  // 只读指针（TOAST指针结构体）
};
```

### 5.1 ExpandedObjectMethods

ExpandedObjectMethods中定义了两个需要编码数据结构的程序员实现的方法。所有支持expand的数据类型都需要实现ExpandedObjectMethods中给出的两个方法，`flatten_into`用于在`detoast`的时候将一个`expand`表示转换为`flat`的表示。而`get_flat_size`是方便获得一个`expand`表示展开成`flat`表示后的大小。

```c++
/* Struct of function pointers for an expanded object's methods */
typedef struct ExpandedObjectMethods
{
 EOM_get_flat_size_method get_flat_size;
 EOM_flatten_into_method flatten_into;
} ExpandedObjectMethods;
```

### 5.2 如何判断一个varlena类型是不是ExpandedObjectHeader

PG在设计ExpandedObjectHeader时，考虑到了对于只读函数，如果既能够处理同一种数据类型常规的 ”flat“ 的varlena输入（即`varattrib_4b`类型），也能够处理其扩展的ExpandedObjectHeader 的输入，这是十分方便的。因此为了使得函数确定输入的varlena指针到底指向的是哪一种类型， ExpandedObjectHeader 的第一个int32始终是-1（定义为宏：`EOH_HEADER_MAGIC`）。 `-1`的二进制表示为`1111 11111`，其不会和`varattrib_4b`的header冲突。

这一判断方法被宏：`VARATT_IS_EXPANDED_HEADER(PTR)`所封装，其返回true表示输入的指针指向的是一个ExpandedObjectHeader对象。

举个例子来说，在Array类型的实现中，Array类型的编码人员设计了一个名为`AnyArrayType`的联合体，包含了这两种不同的array varlena类型。就简化了代码的处理逻辑，只需要以宏AARR_XXX开头的宏就可以同时处理这两种数据类型。

```c++
typedef union AnyArrayType
{
 ArrayType	flt;  	// flat格式的array类型
 ExpandedArrayHeader xpn;  // expand格式的array类型
} AnyArrayType;
/** Macros for working with AnyArrayType inputs.  Beware multiple references!
 为了篇幅删去了具体宏定义*/
#define AARR_NDIM(a)
#define AARR_HASNULL(a)
#define AARR_ELEMTYPE(a)
#define AARR_DIMS(a) 
#define AARR_LBOUND(a)
```

### 5.3 ExpandedObjectHeader 和 varatt_expanded 的关系

我们之前在TOAST 指针中介绍过最后一种expand类型的TOAST指针`varatt_expanded` 结构，其内部只有一个ExpandedObjectHeader指针。而`varatt_expanded` 结构又是存放在结构体`varattrib_1b_e` 的va_data字段。也就是说函数传入的参数一般是一个`varattrib_1b_e` 指针（一个Datum）。

所以为了获取实际的ExpandedObjectHeader指针，我们首先把Datum强制转换为`(varattrib_1b_e *)`指针，然后再利用宏`VARDATA_EXTERNAL`确定varattrib_1b_e 结构中va_data域（指针域）的位置，然后使用memcpy将其拷贝到varatt_expanded结构体中，然后再从中提取出`（ExpandedObjectHeader *`）指针。参加如下的函数：

```c++
// expandeddatu.c
// 给定一个作为expanded对象引用的Datum，在将其转化为varatt_indirect后返回其内部的ExpandedObjectHeader指针
ExpandedObjectHeader *
DatumGetEOHP(Datum d)
{
 varattrib_1b_e *datum = (varattrib_1b_e *) DatumGetPointer(d);
 varatt_expanded ptr;
 Assert(VARATT_IS_EXTERNAL_EXPANDED(datum));
 memcpy(&ptr, VARDATA_EXTERNAL(datum), sizeof(ptr));  // dest <=== src
 Assert(VARATT_IS_EXPANDED_HEADER(ptr.eohptr));
 return ptr.eohptr;
}
```

### 5.4 ExpandedObjectHeader 的例子

在Array类型中，array_expanded.c中定义了Array类型的扩展API和对应的header ExpandedArrayHeader，其和基类ExpandedObjectHeader的关系如下所示：