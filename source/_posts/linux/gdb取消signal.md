---
title: gdb取消signal
date: 2024-06-27 20:47:24
tags:
categories:
    - linux
---

```
(gdb) info signal SIGUSR2
Signal        Stop      Print   Pass to program Description
SIGUSR1       Yes       Yes     Yes             User defined signal 1
(gdb) handle SIGUSR2 noprint nostop
Signal        Stop      Print   Pass to program Description
SIGUSR1       No        No      Yes             User defined signal 
```
