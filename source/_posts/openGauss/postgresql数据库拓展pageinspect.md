---
title: postgreSQL中使用pageinspect拓展
date: 2024-06-27 20:47:24
tags:
categories:
    - openGauss
---

> 转载自
>
> - [postgresql数据库扩展——pageinspect_pgpageinspect-CSDN博客](https://blog.csdn.net/asmartkiller/article/details/118686612)



如果使用MYSQL 相对页面的层次进行一些了解，估计你就的找大佬们的工具集合，并且为此膜拜大佬们，但PG并不需要这样，PG自身自带的pageinspect 工具，就可以让你对页面级别的层次来进行一个 “透心凉” 的查看和分析，并不在为此苦恼。

<!--more-->

首先确认您是否拥有了 pageinspect 这个 extension ，下图通过查看pg_extension这个表您可以确认，当前您的PG上已经安装了这个extension.

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210712235629691.png)

如果没有请 create extension pageinspect; 执行这条预计在您当前的数据块中，如果还不行，请您确认您的PG 安装与编译是否正常。
select * from heap_page_items(get_raw_page(‘test’,0)) order by lp_off desc;

![在这里插入图片描述](https://img-blog.csdnimg.cn/2021071223570099.png)

通过上面的的图，是可以推理出数据存储是从页尾开始的，数据的插入顺序与步进之间的关系。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210712235731255.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FzbWFydGtpbGxlcg==,size_16,color_FFFFFF,t_70)

SELECT * from page_header(get_raw_page(‘test’, 0));

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210712235753934.png)

lower = 72 , 通过这里可以获知当前PG的表TEST 中曾经有过多少tumple(在这一刻)，PG的每页有28bytes 的页头，同时每个指针是4bytes ，(72 - 28)/4 = 11 ,证明当前的指针有11个。
我们插入一条记录
insert into test select generate_series(1,1), random()*100, random()*1, now();

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210712235824374.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FzbWFydGtpbGxlcg==,size_16,color_FFFFFF,t_70)

从上图可以看出，指针并未有变化，并通过查看数据和页面的情况，看到新插入的记录，使用了之前空出的 ctid (0,1) 位置，所以指针并不需要在重新分配。
我们继续在插入两条记录，可以看出指针分配了4个字节，并且新的记录也插入了未分配的空间，每行的偏移量是64bytes

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210712235851211.png)

我们删除 ID > 5 的记录

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210712235911727.png)

然后 vacuum test表

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210712235936476.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FzbWFydGtpbGxlcg==,size_16,color_FFFFFF,t_70)

通过命令我们也可以看到 vacuum 后的空间回收了，并且页头也重新标记了次页面的容量，但指针是不在回收了。

![在这里插入图片描述](https://img-blog.csdnimg.cn/2021071300000387.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FzbWFydGtpbGxlcg==,size_16,color_FFFFFF,t_70)

通过上面几个简单的命令就可以，理解一些枯燥乏味的PG 某些原理，也是不错的体验。
如果还不理解上面的意思可以看下面这个图（由于信息量太大，所以只能截断成两个图）

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210713000028764.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210713000042613.png)


这两张图拼在一起，呈现的就是一个完整的页面上面28个字节头，+ 每个指针 下面就是你存储的每行数据，所以在此证明了页面存储的方式和逻辑中间的0 都是未占用的空间。

我想到此也就没有什么人不在不理解 PG的页面了，试问还有那个数据库在不通过第三方的插件或软件的情况下，能如此通透的展现一个页面在你面前。

SELECT get_raw_page::text FROM get_raw_page(‘test’, 0);

相关的页面获得的源代码，将页面的内容memcpy到buffer 然后给大家展现出来。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210713000110851.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FzbWFydGtpbGxlcg==,size_16,color_FFFFFF,t_70)

那如果有人问，你的数据到底占用了多少个页面，我看看看怎么来通过某种方式来回答他。
1 一个页面我有多少数据
2 一共有多少行数据
2 /1 约等于 多少页面
我们看看上面的算法是不是可以应用到PG 中

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210713000139797.png)


从结果看，还是比较准确的。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210713000206193.png)