---
title: Taas-OpenGauss简单记录
date: 2023-06-30 19:26:36
tags:
---

# 端口约定

Taas使用ZeroMQ进行数据的传输，以下是Taas中所使用到的端口：

- 5551：Client通过该端口将事务发送给Taas节点
- 5552：Taas将事务的执行结果返回给Client
- 5556：Taas将日志（或者叫需持久化的数据）发送给MOT



# OpenGauss初始化

gs_initdb可以对一个已存在的空白目录初始化，使其成为openGauss的数据目录。

```shell
gs_initdb -D /tmp/data --nodename=node1_nodename
```

要使用MOT的话，需要：

```shell
alter system set enable_incremental_checkpoint='off';
```

启动系统：

```shell
gaussdb -D /tmp/data
gs_om -t start
gs_ctl 
```

