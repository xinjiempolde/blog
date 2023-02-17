---
title: GFS
date: 2022-05-31 18:41:25
tags:
    - MIT6.824
categories:
    - paper
---

# 系统架构

![image-20220902151055829](http://img.singhe.art/image-20220902151055829.png)

在GFS中，一个文件由多个chunk组成，每一个chunk大小都是64MB，而一个文件的chunk可能不在同一个chunkserver上，也就是分布在不同的chunkserver上。

GFS采用的是单主架构，client通过filename和offset向master索要chunk信息，master返回对应的chunk在哪个chunkserver上，然后client再和chunkserver交互。

每个chunkserver直接使用linux的文件系统。

<!--more-->

# master数据结构

```c++
filename -> array of chunk handles (Disk)
handle -> list of chunkservers
handle -> version number (Disk)
handle -> primary
handle -> lease expiration
Log, Checkpoint (Disk)
```

- filename -> array of chunk handles保存的是文件名到chunk块号(64 bits)的映射，即记录每个文件对应的所有的chunk。
- handle -> list of chunk servers。每一个chunk有多个副本，master需要记录每个chunk对应的副本在哪个chunkserver上。
- handle -> version number。version number记录当前chunk最新的版本号，由此来判断哪个chunkserver上的数据是最新的。
- handle -> lease expiration。GFS通过lease(租期)来分配primary和secondary。master指定primary并分配lease，lease过期的时间是60s，在此期间primary可以将client的指令组织好顺序发送给secondary。



>  以下部分转载自[https://blog.csdn.net/kdb_viewer/article/details/116111241](https://blog.csdn.net/kdb_viewer/article/details/116111241)

# 一致性模型

gfs的数据一致性是针对多个chunk server保存的相同文件副本来说的，文件按照每64MB一个chunk的形式组织，一个文件可能占用多个chunk，每个chunk都复制多份保存在不同chunk server上，默认副本数量是3。对数据库有一定了解的同学对事务的一致性级别一定不会陌生，gfs对于文件的修改操作和数据库事务有相通之处，对于数据修改后文件的一段数据(region)，定义如下两个一致性级别：

- 一致的(consistency)：所有client无论从哪个副本读一个region，读到的都是同样的内容
- 已定义的(defined)：region一致，且client能看到写入操作的全部内容

下面总结了所有操作的一致性级别：

|          | 任意位置写 | 记录追加           |
| -------- | ---------- | ------------------ |
| 串行成功 | 已定义     | 已定义，部分不一致 |
| 并行成功 | 一致未定义 | 已定义，部分不一致 |
| 失败     | 不一致     | 不一致             |



这里对写操作和记录追加两种修改方式分别做解释

对写操作，串行成功和失败的情况都很好理解，串行成功是最严格的一致性要求，成功后数据一定是一致的；gfs对失败操作没有类似数据库事务的回滚操作，可能在一个副本写入了数据其他副本没写入，或者每个副本写入的数据长度不一样，从而多副本之间的数据是不一致的。这里主要解释下并行成功的情况。并行写的情况发生在多个client写同一个文件区域，例如两个client同时写一个文件alibaba.txt，client A从文件偏移1000的位置开始写入100个字节，client B从文件偏移1050的位置开始写入100个字节，这样两个client的写入有50字节冲突，如图：

![img](http://img.singhe.art/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2tkYl92aWV3ZXI=,size_16,color_FFFFFF,t_70.png)

并行成功不知道1050到1100区间写入的是谁的数据，可能是A也可能是B，甚至可能这段区间混杂着两者的写入，比如对1050位置的一个字节，A先写入成功，然后B写入成功把A刚写入的结果覆盖了，而对1051位置的字节相反，A覆盖了B的写入。物理上这个region每个字节在所有副本上都相同，但是无法读取一个client写入的全部数据，因此这里造成的结果是undefined，也就是一致但未定义。换个角度来说，这个region的数据不是一个可以合理解释的记录，虽然在物理上具有一致性但是这个region的数据是无法使用的，这也就是「未定义」这个一致性级别的名字来源。

对记录追加操作，失败时不一致不做过多解释了，着重解释下成功操作导致的「已定义，但是部分不一致」这个结果。第一眼看可能有些费解，因为「已定义」是比「一致」更高级别的一致性要求，怎么会出现「已定义但是不一致性」这种结果呢，要理解这个要从gfs提供的记录追加方式说起。gfs保证成功的记录追加在多个副本上一定是原子的、最终一致的、自定义偏移的。这里举一个例子帮助理解，一个client在alibaba.txt文件上做记录追加，假设alibaba.txt当前有100个字节，因此本次偏移从100位置开始，追加内容为"hello"，此文件chunk有3副本，如图：

![img](http://img.singhe.art/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2tkYl92aWV3ZXI=,size_16,color_FFFFFF,t_70-20220902153733235.png)

此时该client在第一个副本成功，在第二个副本和第三个副本都写入了一部分内容，此时若另一个client也发起了对alibaba.txt的记录追加，内容为“world”，那么会造成如下结果：

![img](http://img.singhe.art/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2tkYl92aWV3ZXI=,size_16,color_FFFFFF,t_70-20220902153801154.png)

gfs的原子追加保证第一个client发现了在第二个副本自己的追加失败了，因此“hello”的追加会重新发起，第二个client也同样，直到最终两个client的写入都会成功，如下：

![img](http://img.singhe.art/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2tkYl92aWV3ZXI=,size_16,color_FFFFFF,t_70-20220902153817667.png)

gfs对成功的记录追加会返回一个偏移，这个偏移是gfs自己选择的，经过可能多次重试造成的结果。对于第一个client，追加成功后可以得知自己在111偏移位置成功写入了“hello”内容，对第二个client，同样在106偏移位置成功写入了“world”内容。从上图可以明显发现，从100到106位置的字节是不一致的，gfs并不尝试在这种被抛弃的region上强制多副本同步，这也就是「已定义但是不一致」这个看起来矛盾的一致性级别的来源。



# 系统交互

![image-20220902154256623](http://img.singhe.art/image-20220902154256623.png)

master会选择chunk的一个副本为主chunk并建立租约，租期内，主chunk对chunk的所有操作进行序列化，chunk的所有副本都按照这个顺序执行。流程如下：

1-2：client向master询问所有chunk副本位置和主chunk信息

3-6：client向chunk server推送信息，控制流和数据流分离，控制流发给主chunk，数据流顺序推送，目的是使用到100%的网卡带宽，在这种线性管道推送方式下， 传送B字节到R个副本的时间为B/T + LR，其中T是网络吞吐，L是两台机器间的网络延迟，一般小于1ms可以忽略不计。例如，在100Mbps的网络上传输1MB数据，花费80ms

7：主chunk返回client，任何一个副本错误都会认为失败，此时各副本被修改的region处于不一致状态，client负责重试3-6的步骤



# 快照

gfs提供的快照机制是标准的COW（copy-on-write）。具体实现是，client发送快照命令到master，master首先取消当前chunk的租约，保证后续所有对该chunk的访问都通过master进行，然后在文件名空间中创建快照文件，和原始文件关联到同一个chunk，记录该chunk的引用计数为2。当有修改该chunk的操作到来，master发现chunk计数大于2，命令chunk server对该chunk创建副本，此后该chunk和原始chunk就可以分开独立访问了。



# master节点管理

## 文件锁

首先介绍下gfs的文件名空间。和unix文件系统不同，gfs的文件名空间就是一个全路径到元数据的映射表，通过前缀压缩的形式全部在内存中维护，因此没有可以列出一个目录下全部文件的功能，也没有软硬链接的概念。前缀压缩的文件名空间构成了一个树形结构，每个绝对路径对应树形结构中的一个节点，每个节点都有一个关联的读写锁。看个例子：

![img](http://img.singhe.art/20210424234651586.png)

这里leaf节点对应的文件名为/d1/d2/leaf，对leaf进行操作，需要获取d1、d2的读锁，和leaf的读写锁。获取d1和d2的读锁是为了防止操作过程中父目录丢失。所有获取锁的操作都从根节点进行以防止死锁。

## 文件删除

gfs使用惰性文件删除策略。删除指令发送到master后，master将文件重命名为一个包含删除时间戳的隐藏文件名。master节点有例行扫描文件名空间任务，发现超过3天的隐藏文件才发起物理删除，在此期间都可以撤销删除操作以防止误删除。这样设计有如下几个原因：

- 若实时删除，那么master发给chunk server的删除消息可能丢失，master需要维护重试机制，相比之前惰性删除更一致、更可靠
- 惰性策略下实际物理删除发生在master后台定时任务，操作被批量执行，开销分散，master的cpu使用更平缓
- 为人为操作导致的误删除提供兜底

## 过期失效的副本检测

chunk多副本存储在多个chunk server上，可能出现的一种情况是，一个chunk server短暂失联，导致丢失了client对一个chunk的最新修改，针对这种场景，master对每个chunk维护一个版本号，每次和一个chunk建立租约确立主chunk时，都将版本号+1，并通知所有副本，同时client也会在查询chunk位置信息时获取这个信息。失联的chunk server上，chunk版本号不会改变，当chunk server重连向master报告时master会发现过期失效的副本，同时若client向此chunk server读数据也可以通过版本号判断过期。

# 高可用

这里解释gfs实现高可用的技术手段。

- 服务快速拉起，master和chunk server都设计成秒级启动
- chunk复制策略，保证数据不丢

master节点建设主备，client操作在主master和从master全部落盘后才返回，外部监控进程监控master状态并在master故障后选择新的master升主

# 数据完整性

由于gfs的原子追加操作会导致「已定义但是不一致」的状态，数据在byte-wise级别本身就不是一致的，因此chunk服务数据完整性无法通过跨chunk server方案实现，只能在chunk server内部检查。chunk server将chunk切成64KB大小的块，并为每个块维护一个32位的checksum。对读操作，数据返回client之前会检查checksum。对写操作，需要对写范围有覆盖的第一个64KB块和最后一个先进行校验，防止原来存在损坏的数据被本次写隐藏了，然后进行实际写入并重新计算checksum。chunk server空闲时会对所有chunk做整体扫描，尤其针对一个不活动的chunk，防止master认为chunk已经有足够的副本数量了但是实际上副本内容已经损坏。