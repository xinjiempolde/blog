---
title: objdum-readelf-ar简单使用
date: 2023-06-22 11:07:08
tags:
categories:
    - linux
---

# file

file能够查看文件的类型

```shell
singheart@FX504GE:~/Project/assembly$ file libadd.so 
libadd.so: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, BuildID[sha1]=4f628c334595143b6f98886acebd9f8df5a964fa, not stripped
singheart@FX504GE:~/Project/assembly$ file a.o 
a.o: ELF 64-bit LSB relocatable, x86-64, version 1 (SYSV), not stripped
```

<!--more-->

# size

能够查看ELF文件的代码段、数据段和BSS段的长度(dec表示3个段长度的和的十进制，hex表示长度和的十六进制)

```shell
singheart@FX504GE:~/Project/assembly$ size SimpleSection.o 
   text	   data	    bss	    dec	    hex	filename
    182	      8	      8	    198	     c6	SimpleSection.o
```



# objcopy

使用objcopy工具，可以将图片、MP3音乐等作为目标文件中的一个段

```shell
objcopy -I binary -O elf32-i386 image.jpg image.o
objdump -ht image.o
```



# objdump

## objdump -h

`objdump -h`能够输出ELF文件中各个段的信息

```shell
singheart@FX504GE:~/Project/assembly$ objdump -h SimpleSection.o 

SimpleSection.o：     文件格式 elf64-x86-64

节：
Idx Name          Size      VMA               LMA               File off  Algn
  0 .text         0000005a  0000000000000000  0000000000000000  00000040  2**0
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, CODE
  1 .data         00000008  0000000000000000  0000000000000000  0000009c  2**2
                  CONTENTS, ALLOC, LOAD, DATA
  2 .bss          00000008  0000000000000000  0000000000000000  000000a4  2**2
                  ALLOC
  3 .rodata       00000004  0000000000000000  0000000000000000  000000a4  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  4 .comment      0000002e  0000000000000000  0000000000000000  000000a8  2**0
                  CONTENTS, READONLY
  5 .note.GNU-stack 00000000  0000000000000000  0000000000000000  000000d6  2**0
                  CONTENTS, READONLY
  6 .eh_frame     00000058  0000000000000000  0000000000000000  000000d8  2**3
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, DATA
```



## objdump -s

`objdump -s`能够将所有段的内容以十六进制的方式打印出来

```shell
singheart@FX504GE:~/Project/assembly$ objdump -s SimpleSection.o 

SimpleSection.o：     文件格式 elf64-x86-64

Contents of section .text:
 0000 554889e5 4883ec10 897dfc8b 45fc89c6  UH..H....}..E...
 0010 488d0500 00000048 89c7b800 000000e8  H......H........
 0020 00000000 90c9c355 4889e548 83ec10c7  .......UH..H....
 0030 45f80100 00008b15 00000000 8b050000  E...............
 0040 000001c2 8b45f801 c28b45fc 01d089c7  .....E....E.....
 0050 e8000000 008b45f8 c9c3               ......E...      
Contents of section .data:
 0000 54000000 55000000                    T...U...        
Contents of section .rodata:
 0000 25640a00                             %d..            
Contents of section .comment:
 0000 00474343 3a202855 62756e74 75203131  .GCC: (Ubuntu 11
 0010 2e312e30 2d317562 756e7475 317e3138  .1.0-1ubuntu1~18
 0020 2e30342e 31292031 312e312e 3000      .04.1) 11.1.0.  
Contents of section .eh_frame:
 0000 14000000 00000000 017a5200 01781001  .........zR..x..
 0010 1b0c0708 90010000 1c000000 1c000000  ................
 0020 00000000 27000000 00410e10 8602430d  ....'....A....C.
 0030 06620c07 08000000 1c000000 3c000000  .b..........<...
 0040 00000000 33000000 00410e10 8602430d  ....3....A....C.
 0050 066e0c07 08000000                    .n......  
```



## objdump -d

`objdump -d`可以将所有包含指令的段反汇编

```shell
singheart@FX504GE:~/Project/assembly$ objdump -d SimpleSection.o 

SimpleSection.o：     文件格式 elf64-x86-64


Disassembly of section .text:

0000000000000000 <func1>:
   0:	55                   	push   %rbp
   1:	48 89 e5             	mov    %rsp,%rbp
   4:	48 83 ec 10          	sub    $0x10,%rsp
   8:	89 7d fc             	mov    %edi,-0x4(%rbp)
   b:	8b 45 fc             	mov    -0x4(%rbp),%eax
   e:	89 c6                	mov    %eax,%esi
  10:	48 8d 05 00 00 00 00 	lea    0x0(%rip),%rax        # 17 <func1+0x17>
  17:	48 89 c7             	mov    %rax,%rdi
  1a:	b8 00 00 00 00       	mov    $0x0,%eax
  1f:	e8 00 00 00 00       	callq  24 <func1+0x24>
  24:	90                   	nop
  25:	c9                   	leaveq 
  26:	c3                   	retq   

0000000000000027 <main>:
  27:	55                   	push   %rbp
  28:	48 89 e5             	mov    %rsp,%rbp
  2b:	48 83 ec 10          	sub    $0x10,%rsp
  2f:	c7 45 f8 01 00 00 00 	movl   $0x1,-0x8(%rbp)
  36:	8b 15 00 00 00 00    	mov    0x0(%rip),%edx        # 3c <main+0x15>
  3c:	8b 05 00 00 00 00    	mov    0x0(%rip),%eax        # 42 <main+0x1b>
  42:	01 c2                	add    %eax,%edx
  44:	8b 45 f8             	mov    -0x8(%rbp),%eax
  47:	01 c2                	add    %eax,%edx
  49:	8b 45 fc             	mov    -0x4(%rbp),%eax
  4c:	01 d0                	add    %edx,%eax
  4e:	89 c7                	mov    %eax,%edi
  50:	e8 00 00 00 00       	callq  55 <main+0x2e>
  55:	8b 45 f8             	mov    -0x8(%rbp),%eax
  58:	c9                   	leaveq 
  59:	c3                   	retq   
```



## objdump -r

查看重定位段的信息

```shell
singheart@FX504GE:~/Project/assembly$ objdump -r a.o 

a.o：     文件格式 elf64-x86-64

RELOCATION RECORDS FOR [.text]:
OFFSET           TYPE              VALUE 
0000000000000016 R_X86_64_PC32     shared-0x0000000000000004
0000000000000026 R_X86_64_PLT32    swap-0x0000000000000004


RELOCATION RECORDS FOR [.eh_frame]:
OFFSET           TYPE              VALUE 
0000000000000020 R_X86_64_PC32     .text
```



## objdump -t

查看静态库或者目标文件中的符号相关信息

```shell
singheart@FX504GE:~/Project/assembly$ objdump -t SimpleSection.o 

SimpleSection.o：     文件格式 elf64-x86-64

SYMBOL TABLE:
0000000000000000 l    df *ABS*	0000000000000000 SimpleSection.c
0000000000000000 l    d  .text	0000000000000000 .text
0000000000000000 l    d  .data	0000000000000000 .data
0000000000000000 l    d  .bss	0000000000000000 .bss
0000000000000000 l    d  .rodata	0000000000000000 .rodata
0000000000000004 l     O .data	0000000000000004 static_var.1
0000000000000004 l     O .bss	0000000000000004 static_var2.0
0000000000000000 l    d  .note.GNU-stack	0000000000000000 .note.GNU-stack
0000000000000000 l    d  .eh_frame	0000000000000000 .eh_frame
0000000000000000 l    d  .comment	0000000000000000 .comment
0000000000000000 g     O .data	0000000000000004 global_init_var
0000000000000000 g     O .bss	0000000000000004 global_uninit_var
0000000000000000 g     F .text	0000000000000027 func1
0000000000000000         *UND*	0000000000000000 _GLOBAL_OFFSET_TABLE_
0000000000000000         *UND*	0000000000000000 printf
0000000000000027 g     F .text	0000000000000033 main

singheart@FX504GE:~/Project/assembly$ objdump -t /usr/lib/x86_64-linux-gnu/libc.a

dl-reloc-static-pie.o：     文件格式 elf64-x86-64

SYMBOL TABLE:
0000000000000000 l    d  .text	0000000000000000 .text
0000000000000000 l    d  .rodata	0000000000000000 .rodata
0000000000000000 l     O .rodata.str1.16	000000000000001a __PRETTY_FUNCTION__.9805
0000000000000020 l     O .rodata.str1.16	0000000000000015 __PRETTY_FUNCTION__.9847
0000000000000000 l       .rodata.str1.8	0000000000000000 .LC0
0000000000000000 l       .rodata.str1.1	0000000000000000 .LC2
0000000000000040 l       .rodata.str1.8	0000000000000000 .LC1
00000000000000e0 l       .rodata.str1.8	0000000000000000 .LC8
0000000000000100 l       .rodata.str1.8	0000000000000000 .LC9
0000000000000017 l       .rodata.str1.1	0000000000000000 .LC3
0000000000000043 l       .rodata.str1.1	0000000000000000 .LC7
000000000000002a l       .rodata.str1.1	0000000000000000 .LC6
00000000000000a8 l       .rodata.str1.8	0000000000000000 .LC5
0000000000000080 l       .rodata.str1.8	0000000000000000 .LC4
0000000000000000 l    d  .data	0000000000000000 .data
0000000000000000 l    d  .bss	0000000000000000 .bss
0000000000000000 l    d  .rodata.str1.8	0000000000000000 .rodata.str1.8
0000000000000000 l    d  .rodata.str1.1	0000000000000000 .rodata.str1.1
0000000000000000 l    d  .rodata.str1.16	0000000000000000 .rodata.str1.16
0000000000000000 l    d  .note.GNU-stack	0000000000000000 .note.GNU-stack
0000000000000000 l    d  .eh_frame	0000000000000000 .eh_frame
0000000000000000 g     F .text	0000000000000c22 .hidden _dl_relocate_static_pie
0000000000000000         *UND*	0000000000000000 .hidden _dl_get_dl_main_map
0000000000000000         *UND*	0000000000000000 .hidden _DYNAMIC
0000000000000000         *UND*	0000000000000000 .hidden _GLOBAL_OFFSET_TABLE_
0000000000000000  w      *UND*	0000000000000000 _dl_rtld_map
0000000000000000         *UND*	0000000000000000 _dl_argv
0000000000000000         *UND*	0000000000000000 .hidden _dl_dprintf
0000000000000000         *UND*	0000000000000000 .hidden _dl_tlsdesc_return
0000000000000000         *UND*	0000000000000000 memcpy
0000000000000000         *UND*	0000000000000000 .hidden _dl_tlsdesc_undefweak
0000000000000000         *UND*	0000000000000000 .hidden _dl_reloc_bad_type
0000000000000000         *UND*	0000000000000000 .hidden _dl_allocate_static_tls
0000000000000000         *UND*	0000000000000000 .hidden __assert_fail



dl-vdso.o：     文件格式 elf64-x86-64

SYMBOL TABLE:
0000000000000000 l    d  .text	0000000000000000 .text
0000000000000000 l    d  .data	0000000000000000 .data
0000000000000000 l    d  .bss	0000000000000000 .bss
0000000000000000 l    d  .note.GNU-stack	0000000000000000 .note.GNU-stack
0000000000000000 l    d  .eh_frame	0000000000000000 .eh_frame
0000000000000000 g     F .text	0000000000000099 _dl_vdso_vsym
0000000000000000         *UND*	0000000000000000 _dl_sysinfo_map
0000000000000000         *UND*	0000000000000000 .hidden _dl_lookup_symbol_x
0000000000000000         *UND*	0000000000000000 _GLOBAL_OFFSET_TABLE_
0000000000000000         *UND*	0000000000000000 __stack_chk_fail

```



# readelf

## readelf -h

`readelf -h`能够查看ELF头的信息

```
singheart@FX504GE:~/Project/assembly$ readelf -h SimpleSection.o 
ELF 头：
  Magic：   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  类别:                              ELF64
  数据:                              2 补码，小端序 (little endian)
  版本:                              1 (current)
  OS/ABI:                            UNIX - System V
  ABI 版本:                          0
  类型:                              REL (可重定位文件)
  系统架构:                          Advanced Micro Devices X86-64
  版本:                              0x1
  入口点地址：               0x0
  程序头起点：          0 (bytes into file)
  Start of section headers:          1104 (bytes into file)
  标志：             0x0
  本头的大小：       64 (字节)
  程序头大小：       0 (字节)
  Number of program headers:         0
  节头大小：         64 (字节)
  节头数量：         13
  字符串表索引节头： 12
```



## readelf -S

能够查看ELF文件的段，和`objdump -h`作用相同

```shell
singheart@FX504GE:~/Project/assembly$ readelf -S SimpleSection.o 
There are 13 section headers, starting at offset 0x450:

节头：
  [号] 名称              类型             地址              偏移量
       大小              全体大小          旗标   链接   信息   对齐
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .text             PROGBITS         0000000000000000  00000040
       000000000000005a  0000000000000000  AX       0     0     1
  [ 2] .rela.text        RELA             0000000000000000  00000340
       0000000000000078  0000000000000018   I      10     1     8
  [ 3] .data             PROGBITS         0000000000000000  0000009c
       0000000000000008  0000000000000000  WA       0     0     4
  [ 4] .bss              NOBITS           0000000000000000  000000a4
       0000000000000008  0000000000000000  WA       0     0     4
  [ 5] .rodata           PROGBITS         0000000000000000  000000a4
       0000000000000004  0000000000000000   A       0     0     1
  [ 6] .comment          PROGBITS         0000000000000000  000000a8
       000000000000002e  0000000000000001  MS       0     0     1
  [ 7] .note.GNU-stack   PROGBITS         0000000000000000  000000d6
       0000000000000000  0000000000000000           0     0     1
  [ 8] .eh_frame         PROGBITS         0000000000000000  000000d8
       0000000000000058  0000000000000000   A       0     0     8
  [ 9] .rela.eh_frame    RELA             0000000000000000  000003b8
       0000000000000030  0000000000000018   I      10     8     8
  [10] .symtab           SYMTAB           0000000000000000  00000130
       0000000000000198  0000000000000018          11    11     8
  [11] .strtab           STRTAB           0000000000000000  000002c8
       0000000000000076  0000000000000000           0     0     1
  [12] .shstrtab         STRTAB           0000000000000000  000003e8
       0000000000000061  0000000000000000           0     0     1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  l (large), p (processor specific)
```



## readelf -s

能查看ELF文件中的符号段的信息

```shell
Symbol table '.symtab' contains 17 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS SimpleSection.c
     2: 0000000000000000     0 SECTION LOCAL  DEFAULT    1 
     3: 0000000000000000     0 SECTION LOCAL  DEFAULT    3 
     4: 0000000000000000     0 SECTION LOCAL  DEFAULT    4 
     5: 0000000000000000     0 SECTION LOCAL  DEFAULT    5 
     6: 0000000000000004     4 OBJECT  LOCAL  DEFAULT    3 static_var.1
     7: 0000000000000004     4 OBJECT  LOCAL  DEFAULT    4 static_var2.0
     8: 0000000000000000     0 SECTION LOCAL  DEFAULT    7 
     9: 0000000000000000     0 SECTION LOCAL  DEFAULT    8 
    10: 0000000000000000     0 SECTION LOCAL  DEFAULT    6 
    11: 0000000000000000     4 OBJECT  GLOBAL DEFAULT    3 global_init_var
    12: 0000000000000000     4 OBJECT  GLOBAL DEFAULT    4 global_uninit_var
    13: 0000000000000000    39 FUNC    GLOBAL DEFAULT    1 func1
    14: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND _GLOBAL_OFFSET_TABLE_
    15: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND printf
    16: 0000000000000027    51 FUNC    GLOBAL DEFAULT    1 main
```

`nm`也能达到类似的效果

```shell
singheart@FX504GE:~/Project/assembly$ nm SimpleSection.o 
0000000000000000 T func1
0000000000000000 D global_init_var
                 U _GLOBAL_OFFSET_TABLE_
0000000000000000 B global_uninit_var
0000000000000027 T main
                 U printf
0000000000000004 d static_var.1
0000000000000004 b static_var2.0
```



## readelf -l

查看可执行文件的Segment header

```shell
singheart@FX504GE:~/Project/assembly$ readelf -l hello

Elf 文件类型为 EXEC (可执行文件)
Entry point 0x4000b0
There are 2 program headers, starting at offset 64

程序头：
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  LOAD           0x0000000000000000 0x0000000000400000 0x0000000000400000
                 0x00000000000000d2 0x00000000000000d2  R E    0x200000
  LOAD           0x00000000000000d2 0x00000000006000d2 0x00000000006000d2
                 0x000000000000000e 0x000000000000000e  RW     0x200000

 Section to Segment mapping:
  段节...
   00     .text 
   01     .data
```



## readelf -d

查看".dynamic"段的内容：

```shell
singheart@FX504GE:~/Project/assembly$ readelf -d Lib.so 

Dynamic section at offset 0xe20 contains 24 entries:
  标记        类型                         名称/值
 0x0000000000000001 (NEEDED)             共享库：[libc.so.6]
 0x000000000000000c (INIT)               0x520
 0x000000000000000d (FINI)               0x66c
 0x0000000000000019 (INIT_ARRAY)         0x200e10
 0x000000000000001b (INIT_ARRAYSZ)       8 (bytes)
 0x000000000000001a (FINI_ARRAY)         0x200e18
 0x000000000000001c (FINI_ARRAYSZ)       8 (bytes)
 0x000000006ffffef5 (GNU_HASH)           0x1f0
 0x0000000000000005 (STRTAB)             0x368
 0x0000000000000006 (SYMTAB)             0x230
 0x000000000000000a (STRSZ)              163 (bytes)
 0x000000000000000b (SYMENT)             24 (bytes)
 0x0000000000000003 (PLTGOT)             0x201000
 0x0000000000000002 (PLTRELSZ)           48 (bytes)
 0x0000000000000014 (PLTREL)             RELA
 0x0000000000000017 (JMPREL)             0x4f0
 0x0000000000000007 (RELA)               0x448
 0x0000000000000008 (RELASZ)             168 (bytes)
 0x0000000000000009 (RELAENT)            24 (bytes)
 0x000000006ffffffe (VERNEED)            0x428
 0x000000006fffffff (VERNEEDNUM)         1
 0x000000006ffffff0 (VERSYM)             0x40c
 0x000000006ffffff9 (RELACOUNT)          3
 0x0000000000000000 (NULL)               0x0
```



## readelf -sD

查看动态符号表的内容和哈希表的内容

```shell
singheart@FX504GE:~/Project/assembly$ readelf -sD Lib.so 

Symbol table of `.gnu.hash' for image:
  Num Buc:    Value          Size   Type   Bind Vis      Ndx Name
    7   0: 0000000000201030     0 NOTYPE  GLOBAL DEFAULT  22 _edata
    8   0: 0000000000201038     0 NOTYPE  GLOBAL DEFAULT  23 _end
    9   1: 0000000000201030     0 NOTYPE  GLOBAL DEFAULT  23 __bss_start
   10   1: 0000000000000520     0 FUNC    GLOBAL DEFAULT   9 _init
   11   2: 000000000000066c     0 FUNC    GLOBAL DEFAULT  13 _fini
   12   2: 0000000000000635    54 FUNC    GLOBAL DEFAULT  12 foobar
```



# c++filt

能够解析C++修饰过的符号名称

```shell
singheart@FX504GE:~/Project/assembly$ c++filt _ZN1N1C4funcEi
N::C::func(int)
singheart@FX504GE:~/Project/assembly$ c++filt _ZZ4mainE3foo
main::foo
```



# ar

## ar -t

查看静态库中包含哪些目标文件

```shell
singheart@FX504GE:~/Project/assembly$ ar -t /usr/lib/x86_64-linux-gnu/libc.a
init-first.o
libc-start.o
sysdep.o
version.o
check_fds.o
libc-tls.o
elf-init.o
dso_handle.o
errno.o
errno-loc.o
iconv_open.o
iconv.o
iconv_close.o
gconv_open.o
gconv.o
gconv_close.o
gconv_db.o
gconv_conf.o
gconv_builtin.o
gconv_simple.o
gconv_trans.o
gconv_cache.o
gconv_dl.o
gconv_charset.o
setlocale.o
findlocale.o
loadlocale.o
loadarchive.o
localeconv.o
...
```



# ldd

ldd可以查看程序主模块或者一个共享库依赖于哪些共享库

```shell
singheart@FX504GE:~/Project/assembly$ ldd Program1
	linux-vdso.so.1 (0x00007ffdc75d1000)
	./Lib.so (0x00007f60a19ab000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f60a15ba000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f60a1daf000)
```



# 使用C语言解析ELF

```c
#include <fcntl.h>
#include <stdio.h>
#include <unistd.h>
#include <string.h>
#include <iostream>
#include <elf.h>

int fd;
void ParseELFHeader(const Elf64_Ehdr &elf_header);
void ParseSectionHeader(const Elf64_Ehdr &elf_header, Elf64_Shdr *section_header);

void ParseELF(const char * filepath) {
  fd = open(filepath, O_RDONLY);
  if (fd < 0) {
    std::cerr << "Error opening file " << filepath << "\n";
    exit(1);
  }
  Elf64_Ehdr elf_header;
  int read_elf_header_size = read(fd, &elf_header, sizeof(Elf64_Ehdr));
  if (read_elf_header_size < sizeof(Elf64_Ehdr)) {
    std::cerr << "ELF file is not complete\n";
    exit(1);
  }
  ParseELFHeader(elf_header);
  
  // 解析Section header table
  Elf64_Shdr *section_header = new Elf64_Shdr[elf_header.e_shnum];
  lseek(fd, elf_header.e_shoff, SEEK_SET);
  read(fd, section_header, sizeof(Elf64_Shdr) * elf_header.e_shnum);
  ParseSectionHeader(elf_header, section_header);
  delete[] section_header;
}

void ParseELFHeader(const Elf64_Ehdr &elf_header) {
  printf("ELF文件头格式如下：\n");
  printf("Magic(e_ident):");
  for (int i = 0; i < EI_NIDENT; i++) {
    printf("%.2x ", elf_header.e_ident[i]);
  }
  printf("\n");
  // TODO: 按照Elf64_Ehdr的格式分析完
  // ...
  printf("Section header table的文件内偏移地址：0x%lx\n", elf_header.e_shoff);
  printf("一共有多少个段：%d\n", elf_header.e_shnum);
  printf("Section header string table的下标: %d\n", elf_header.e_shstrndx);
}

void ParseSectionHeader(const Elf64_Ehdr &elf_header, Elf64_Shdr *section_header) {
  printf("%x", section_header[1].sh_name);
  // 先解析Section header string table
  Elf64_Shdr *shstrtab_header = &section_header[elf_header.e_shstrndx];
  printf("shstrtab的文件内偏移地址：0x%lx\n", shstrtab_header->sh_offset);
  printf("shstrtab段的总字节大小：%ld\n", shstrtab_header->sh_size);
  char *shstrtab = new char[shstrtab_header->sh_size];
  lseek(fd, shstrtab_header->sh_offset, SEEK_SET);
  read(fd, shstrtab, shstrtab_header->sh_size);
  
  // 解析其他段的内容，把段名打印出来
  printf("所有的段段名如下：\n");
  int symtable_idx = 0;
  int strtab_idx = 0;
  int text_idx = 0;
  for (int i = 0; i < elf_header.e_shnum; i++) {
    printf("%s\n", shstrtab + section_header[i].sh_name);
    if (strcmp(".symtab", shstrtab + section_header[i].sh_name) == 0) {
      symtable_idx = i;
    } else if (strcmp(".strtab", shstrtab + section_header[i].sh_name) == 0) {
      strtab_idx = i;
    } else if (strcmp(".text", shstrtab + section_header[i].sh_name) == 0) {
      text_idx = i;
    }
  }
  printf("符号表的数组下标：%d\n", symtable_idx);
  printf("字符串表的数组下标：%d\n", strtab_idx);

  // text属性
  printf("VMA: %lx\n", section_header[text_idx].sh_addr);

  // 解析字符串表的内容
  char *strtab = new char[section_header[strtab_idx].sh_size];
  lseek(fd, section_header[strtab_idx].sh_offset, SEEK_SET);
  read(fd, strtab, section_header[strtab_idx].sh_size);

  // 解析符号表的内容
  printf(".symtab的文件内偏移地址：0x%lx\n", section_header[symtable_idx].sh_offset);
  printf(".symtab的段大小：%ld\n", section_header[symtable_idx].sh_size);
  Elf64_Sym *symtab = reinterpret_cast<Elf64_Sym *>(new char[section_header[symtable_idx].sh_size]);
  lseek(fd, section_header[symtable_idx].sh_offset, SEEK_SET);
  read(fd, symtab, section_header[symtable_idx].sh_size);
  int sym_num = section_header[symtable_idx].sh_size / sizeof(Elf64_Sym);
  // 打印所有的符号
  for (int i = 0; i < sym_num; i++) {
    printf("%s\n", strtab + symtab[i].st_name);
  }

  delete []strtab;
  delete []shstrtab;
  delete []symtab;
  
}
int main() {
  ParseELF("../ab");
  return 0;
}
```

