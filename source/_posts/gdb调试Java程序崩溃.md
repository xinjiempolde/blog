---
title: gdb调试Java程序崩溃
date: 2024-06-21 16:25:09
tags:
  - coding
---

> 参考[在Linux生成core dump文件](https://senlinzhan.github.io/2017/12/31/coredump/)

# 0x00 背景

Java程序调用C语言生成的动态链接库(.so)，动态链接库有问题Crash了导致Java程序也Crash了，如何调试动态链接库哪里出现了问题。

<!--more-->

# 0x01 生成coredump文件

## 1.1 设置coredump文件大小

如果进程在运行期间发生奔溃，操作系统会为进程生成一个快照文件，这个文件就叫做 core dump。之后我们可以对 core dump 文件进行分析，弄清楚进程为什么会奔溃。由于 core dump 文件会占据一定的磁盘空间，默认情况下，Linux 不允许生成 core dump 文件。例如，下面的命令显示，Linux 允许的最大 core dump 文件大小为 0：

```shell
$ ulimit -a | grep core
core file size          (blocks, -c) 0
```

可以通过下面设置，允许 Linux 生成 core dump 文件(这个设置只对当前登录回话有效)：

```shell
ulimit -c unlimited
```

## 1.2 设置core dump 文件的路径

那么 core dump 会存放在哪个目录呢？这是由系统参数`kernel.core_pattern`决定的。例如，在 Ubuntu 16.04 中，它的值如下：

```shell
$ cat /proc/sys/kernel/core_pattern
|/usr/share/apport/apport %p %s %c %P
```

在实践中，更好的做法是自己指定 core dump 目录，以及 core dump 文件的命名方式：

```shell
$ sudo vi /etc/sysctl.conf
kernel.core_pattern=/var/crash/%E.%p.%t.%s
$ sudo sysctl -p
```

**%p**:替换为导致核心转储的进程ID（PID）。

**%u**:替换为导致核心转储的实际用户ID（UID）。

**%g**:替换为导致核心转储的实际组ID（GID）。

**%s**:替换为导致核心转储的信号编号（导致程序崩溃的信号）。

**%t**:替换为生成转储的时间戳（自1970年1月1日以来的秒数）。

**%h**:替换为主机名。

**%e**:替换为导致核心转储的程序的可执行文件名（不包含路径）。

**%E**:替换为导致核心转储的程序的可执行文件路径。

# 0x02 gdb调试coredump文件

```shell
gdb /usr/lib/libtaos.so /var/crash/core.org.example.Mai.3701
```

只需要指定动态链接库的路径以及coredump文件就可以调试了
