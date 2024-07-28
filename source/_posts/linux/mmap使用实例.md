---
title: mmap使用实例
date: 2023-06-27 20:47:24
tags:
categories:
    - linux
---

# 引言

通过mmap，能将file的一段内容映射到一段虚拟地址上，我们就可以在这个虚拟地址上对文件进行更改，操作系统会自动将脏数据刷回到磁盘上。



<!--more-->

# 例子

```c
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/mman.h>
int main() {
	char buf[128];
	//打开文件
	int fd = open("testdata",O_RDWR);
	//创建mmap
	char *start = (char *)mmap(NULL,128,PROT_READ|PROT_WRITE,MAP_SHARED,fd,0);
	//读取文件	
	strcpy(buf,start);
	printf("%s\n",buf);
	//写入文件
	strcpy(start,"Write to file!\n");
 
	munmap(start,128);
	close(fd);
	return 0;
}

```



# xclip的使用

顺带讲以下linux下怎么在终端和剪贴板间互动

> 从[https://segmentfault.com/a/1190000024579429](https://segmentfault.com/a/1190000024579429)复制

```shell
// 查看剪切板中的内容
$ xclip -o
$ xclip -selection c -o

// 将输出复制至剪贴板
$ echo "hello xclip" | xclip-selection c

// 将文件中的内容全部复制至剪贴板
$ xclip -selection c remade.md

// 将剪切板中的内容粘贴至文件
$ xclip -selection c -o > remade.md
```

