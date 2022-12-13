---
title: tar使用总结
date: 2022-04-10 13:21:23
tags:
    - coding
    - linux
categories:
    - linux
---

> tar是归档文件，.bz2或者.gzip是压缩格式。
# 1. 解压&提取
## 1.1 对于tar.gz结尾的压缩包
```shell
tar -zxvf *.tar.gz
```
参数解读
- `-z`:使用gzip来处理压缩包。
- `-x`:extract，提取文件。
- `-v`:verbose，显示提取细节
- `f`:file，指定文件

<!--more-->

## 1.2 对于tar.bz2结尾的压缩包
```shell
tar -jxvf *.tar.bz2
```
参数解读
- `-j`:使用`bunzip2`来处理压缩包。
- `-x`:extract，提取文件。
- `-v`:verbose，显示提取细节
- `-f`:file，指定文件


# 2. 归档&压缩
## 2.1 使用gzip来压缩
```shell
tar -zcvf *.tar.gz /path/to/dir /path2/to/dir
```
使用`gzip`处理的文件结尾为`.gz`
参数解读
- `-z`:使用`gzip`压缩文件
- `-c`:create，创建归档文件
- `-v`:verbose，显示详细信息
- `-f`:file，指定文件

## 2.2 使用bunzip2来压缩
```shell
tar -jcvf *.tar.bz2 /path/to/dir /path2/to/dir
```
使用`bunzip2`处理的文件结尾为`.bz2`
- `-j`:使用`bunzip2`压缩文件
- `-c`:create，创建归档文件
- `-v`:verbose，显示详细信息
- `-f`:file，指定文件
