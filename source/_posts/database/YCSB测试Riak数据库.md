---
title: YCSB测试Riak数据库
date: 2022-06-31 18:41:25
tags:
    - database
    - Riak
    - YCSB
categories:
    - database
---

# 前言

笔者踩了许多坑，最大的两个问题是Riak必须配置3个节点，第二个问题是得手动修改jar包才能进行配置。至于用Python和Java连接Riak，笔者有时间会单独出一篇教程。

# 环境

CentOS 7.6 64位

若未有特殊说明，所有操作均在`root`下进行操作。

<!--more-->
# 安装Riak

> 参考[Riak - 安装运维篇](https://cloud.tencent.com/developer/article/1811945)

安装必要的软件和依赖

```bash
yum install pam-devel gcc gcc-c++ glibc-devel make ncurses-devel openssl-devel autoconf git
```

之后，需要下载编译ErLang。因为Riak是Erlang编写的，我们从源代码编译Riak安装。

```shell
wget http://s3.amazonaws.com/downloads.basho.com/erlang/otp_src_R16B02-basho8.tar.gz
tar zxvf otp_src_R16B02-basho8.tar.gz
cd OTP_R16B02_basho8/
./otp_build autoconf
CFLAGS="-DOPENSSL_NO_EC=1" ./configure && make && sudo make install
```

安装好后，输入`erl`， 可以看到

```shell
#erl
Erlang R16B02_basho8 (erts-5.10.3) [source] [64-bit] [smp:32:32] [async-threads:10] [hipe] [kernel-poll:false]

Eshell V5.10.3  (abort with ^G)
1>
```

安装Riak，编译安装3个Riak实例：

```shell
wget http://s3.amazonaws.com/downloads.basho.com/riak/2.1/2.1.4/rhel/6/riak-2.1.4-1.el6.src.rpm
rpm -ivh riak-2.1.4-1.el6.src.rpm
```

需要注意的是，根据[Riak文档](https://riak.docs.hw.ag/riak/kv/latest/configuring/strong-consistency/#minimum-cluster-size)所说，应该至少有3个节点才能保证Riak是可用的，否则用YCSB进行测试的时候会出现`RiakResponseException: unavailable`的错误。

![image-20220619101547044](http://img.singhe.art/image-20220619101547044.png)

这之后会把源代码文件夹安装到对应的rpm安装位置，find下即可，我这里是`/root/rpmbuild/SOURCES/riak-2.1.4.tar.gz`

```shell
cp /root/rpmbuild/SOURCES/riak-2.1.4.tar.gz ./
tar zxvf riak-2.1.4.tar.gz
cd riak-2.1.4
make devrel DEVNODES=3 # 节点数量至少为3
```

make的时候会下载一些东西（比如solr等），需要耐心些。安装完成之后5个实例都都在当前目录的dev文件夹下。tree一下

```shell
#tree -L 2 dev/
dev/
├── dev1
│   ├── bin
│   ├── data
│   ├── erts-5.10.3
│   ├── etc
│   ├── lib
│   ├── log
│   └── releases
├── dev2
│   ├── bin
│   ├── data
│   ├── erts-5.10.3
│   ├── etc
│   ├── lib
│   ├── log
│   └── releases
├── dev3
│   ├── bin
│   ├── data
│   ├── erts-5.10.3
│   ├── etc
│   ├── lib
│   ├── log
│   └── releases
```

我们先启动一个实例，dev1，首先修改下配置文件：

```shell
cd dev/dev1
vim ./etc/riak.conf
```

主要修改： 

- nodename = dev1@127.0.0.1
- listener.http.internal = 127.0.0.1:10018
- listener.protobuf.internal = 127.0.0.1:10017
- storage_backend = leveldb
- strong_consistency = on

 这三项为自己实际要绑定的IP即可，若是在同一台服务器上运行YCSB和Riak，保持默认的127.0.0.1即可；若是一台服务器运行Riak，另一台服务器运行YCSB，则需要将IP设置为Riak服务器的IP。  对于`dev2`和`dev3`进行同样的修改，之后启动：

```shell
./bin/riak start
```

启动成功后，查看状态和集群状态：

```shell
#./bin/riak-admin cluster status
---- Cluster Status ----
Ring ready: true

+------------------------+------+-------+-----+-------+
|          node          |status| avail |ring |pending|
+------------------------+------+-------+-----+-------+
| (C) dev1@127.0.0.1     |valid |  up   |100.0|  --   |
+------------------------+------+-------+-----+-------+

Key: (C) = Claimant; availability marked with '!' is unexpected
```

再配置两个node，分别是dev2和dev3文件夹下的，同样只是修改那三个属性的IP即可，端口已经自动写好了，不用改，之后启动。

```shell
#./dev2/bin/riak start
#./dev2/bin/riak-admin cluster status
---- Cluster Status ----
Ring ready: true

+------------------------+------+-------+-----+-------+
|          node          |status| avail |ring |pending|
+------------------------+------+-------+-----+-------+
| (C) dev2@127.0.0.1     |valid |  up   |100.0|  --   |
+------------------------+------+-------+-----+-------+

Key: (C) = Claimant; availability marked with '!' is unexpected
#./dev3/bin/riak start
#./dev3/bin/riak-admin cluster status
---- Cluster Status ----
Ring ready: true

+------------------------+------+-------+-----+-------+
|          node          |status| avail |ring |pending|
+------------------------+------+-------+-----+-------+
| (C) dev3@127.0.0.1     |valid |  up   |100.0|  --   |
+------------------------+------+-------+-----+-------+

Key: (C) = Claimant; availability marked with '!' is unexpected
```

之后，开始配置一个三个Riak node的集群。首先，将dev1@127.0.0.1和dev2@127.0.0.1组成一个集群。

```shell
#./dev2/bin/riak-admin cluster join dev1@127.0.0.1
Success: staged join request for 'dev2@127.0.0.1' to 'dev1@127.0.0.1'
#./dev2/bin/riak-admin cluster plan
=============================== Staged Changes ================================
Action         Details(s)
-------------------------------------------------------------------------------
join           'dev2@127.0.0.1'
-------------------------------------------------------------------------------


NOTE: Applying these changes will result in 1 cluster transition

###############################################################################
                         After cluster transition 1/1
###############################################################################

================================= Membership ==================================
Status     Ring    Pending    Node
-------------------------------------------------------------------------------
valid     100.0%     50.0%    'dev1@127.0.0.1'
valid       0.0%     50.0%    'dev2@127.0.0.1'
-------------------------------------------------------------------------------
Valid:2 / Leaving:0 / Exiting:0 / Joining:0 / Down:0

WARNING: Not all replicas will be on distinct nodes

Transfers resulting from cluster changes: 32
  32 transfers from 'dev1@127.0.0.1' to 'dev2@127.0.0.1'

#./dev2/bin/riak-admin cluster commit
Cluster changes committed
#./dev2/bin/riak-admin cluster status
---- Cluster Status ----
Ring ready: true

+------------------------+------+-------+-----+-------+
|          node          |status| avail |ring |pending|
+------------------------+------+-------+-----+-------+
| (C) dev1@127.0.0.1     |valid |  up   | 87.5|  50.0 |
|     dev2@127.0.0.1     |valid |  up   | 12.5|  50.0 |
+------------------------+------+-------+-----+-------+

Key: (C) = Claimant; availability marked with '!' is unexpected
#./dev2/bin/riak-admin cluster status
---- Cluster Status ----
Ring ready: true

+------------------------+------+-------+-----+-------+
|          node          |status| avail |ring |pending|
+------------------------+------+-------+-----+-------+
| (C) dev1@127.0.0.1     |valid |  up   | 75.0|  50.0 |
|     dev2@127.0.0.1     |valid |  up   | 25.0|  50.0 |
+------------------------+------+-------+-----+-------+

Key: (C) = Claimant; availability marked with '!' is unexpected
#./dev2/bin/riak-admin cluster status
---- Cluster Status ----
Ring ready: true

+------------------------+------+-------+-----+-------+
|          node          |status| avail |ring |pending|
+------------------------+------+-------+-----+-------+
| (C) dev1@127.0.0.1     |valid |  up   | 62.5|  50.0 |
|     dev2@127.0.0.1     |valid |  up   | 37.5|  50.0 |
+------------------------+------+-------+-----+-------+

Key: (C) = Claimant; availability marked with '!' is unexpected
#./dev2/bin/riak-admin cluster status
---- Cluster Status ----
Ring ready: true

+------------------------+------+-------+-----+-------+
|          node          |status| avail |ring |pending|
+------------------------+------+-------+-----+-------+
| (C) dev1@127.0.0.1     |valid |  up   | 50.0|  --   |
|     dev2@127.0.0.1     |valid |  up   | 50.0|  --   |
+------------------------+------+-------+-----+-------+

Key: (C) = Claimant; availability marked with '!' is unexpected
```

分为三步，先join某一个集群（只用填集群中一个节点的名字即可），之后plan，看下分布和需要的移动操作（这些在commit之后riak会自己做），最后确认无误，则commit。  对于dev3@127.0.0.1也是一样：

```shell
#./dev3/bin/riak-admin cluster join dev1@127.0.0.1
Success: staged join request for 'dev3@127.0.0.1' to 'dev1@127.0.0.1'
#./dev3/bin/riak-admin cluster status
---- Cluster Status ----
Ring ready: true

+------------------------+-------+-------+-----+-------+
|          node          |status | avail |ring |pending|
+------------------------+-------+-------+-----+-------+
|     dev3@127.0.0.1     |joining|  up   |  0.0|  --   |
| (C) dev1@127.0.0.1     | valid |  up   | 50.0|  --   |
|     dev2@127.0.0.1     | valid |  up   | 50.0|  --   |
+------------------------+-------+-------+-----+-------+

Key: (C) = Claimant; availability marked with '!' is unexpected
#./dev3/bin/riak-admin cluster plan
=============================== Staged Changes ================================
Action         Details(s)
-------------------------------------------------------------------------------
join           'dev3@127.0.0.1'
-------------------------------------------------------------------------------


NOTE: Applying these changes will result in 1 cluster transition

###############################################################################
                         After cluster transition 1/1
###############################################################################

================================= Membership ==================================
Status     Ring    Pending    Node
-------------------------------------------------------------------------------
valid      50.0%     34.4%    'dev1@127.0.0.1'
valid      50.0%     32.8%    'dev2@127.0.0.1'
valid       0.0%     32.8%    'dev3@127.0.0.1'
-------------------------------------------------------------------------------
Valid:3 / Leaving:0 / Exiting:0 / Joining:0 / Down:0

WARNING: Not all replicas will be on distinct nodes

Transfers resulting from cluster changes: 21
  10 transfers from 'dev1@127.0.0.1' to 'dev3@127.0.0.1'
  11 transfers from 'dev2@127.0.0.1' to 'dev3@127.0.0.1'

#./dev3/bin/riak-admin cluster commit
Cluster changes committed
#./dev3/bin/riak-admin cluster status
---- Cluster Status ----
Ring ready: true

+------------------------+------+-------+-----+-------+
|          node          |status| avail |ring |pending|
+------------------------+------+-------+-----+-------+
| (C) dev1@127.0.0.1     |valid |  up   | 43.8|  34.4 |
|     dev2@127.0.0.1     |valid |  up   | 50.0|  32.8 |
|     dev3@127.0.0.1     |valid |  up   |  6.3|  32.8 |
+------------------------+------+-------+-----+-------+

Key: (C) = Claimant; availability marked with '!' is unexpected
#./dev3/bin/riak-admin cluster status
---- Cluster Status ----
Ring ready: true

+------------------------+------+-------+-----+-------+
|          node          |status| avail |ring |pending|
+------------------------+------+-------+-----+-------+
| (C) dev1@127.0.0.1     |valid |  up   | 34.4|  --   |
|     dev2@127.0.0.1     |valid |  up   | 32.8|  --   |
|     dev3@127.0.0.1     |valid |  up   | 32.8|  --   |
+------------------------+------+-------+-----+-------+

Key: (C) = Claimant; availability marked with '!' is unexpected
```

这里riak-admin cluster status可以查看集群状态。status是目前每个节点的状态。avail代表是否可以访问，ring就是每个节点持有多少百分比的数据。和Dynamo的思想一致，riak以一致性哈希环保存数据。这个ring就是虚节点，里面的百分比就是每个节点持有虚节点个数。

运行成功后，我们需要创建bucket-type，供YCSB测试使用。

```shell
dev/dev1/riak-admin bucket-type create ycsb '{"props":{"consistent":true}}'
dev/dev1/riak-admin bucket-type activate ycsb
dev/dev1/riak-admin bucket-type create fakeBucketType '{"props":{"allow_mult":"false","n_val":1,"dvv_enabled":false,"last_write_wins":true}}'
dev/dev1/riak-admin bucket-type activate fakeBucketType
```



# 安装YCSB

> 参考[YCSB/riak at master · brianfrankcooper/YCSB · GitHub](https://github.com/brianfrankcooper/YCSB/tree/master/riak)

下载YCSB最新版本

```shell
wget https://github.com/brianfrankcooper/YCSB/releases/download/0.17.0/ycsb-0.17.0.tar.gz
tar -zxvf ycsb-0.17.0.tar.gz
cd ycsb-0.17.0
```

笔者参考了所有的相关资料，发现配置文件对`Riak-YCSB`并不起作用，阅读源代码知道，配置文件写死在了jar包里面不能自己设置，因此只能够修改`Riak-binding`的jar包。在`ycsb-0.17.0`目录下运行一下命令

```shell
cd riak-binding/lib
mv riak-binding-0.17.0.jar riak-binding-0.17.0.jar.bak
unzip riak-binding-0.17.0.jar.bak
```

解压jar包后会得到`META-INF`，`riak.properties`和`site`三个文件（或文件夹）

![image-20220619104949932](http://img.singhe.art/image-20220619104949932.png)

修改`riak.properties`为如下配置。

```shell

# implied. See the License for the specific language governing
# permissions and limitations under the License. See accompanying
# LICENSE file.
#

# RiakKVClient - Default Properties
# Note: Change the properties below to set the values to use for your test. You can set them either here or from the
# command line. Note that the latter choice overrides these settings.

# riak.hosts - string list, comma separated list of IPs or FQDNs.
# EX: 127.0.0.1,127.0.0.2,127.0.0.3 or riak1.mydomain.com,riak2.mydomain.com,riak3.mydomain.com
riak.hosts=127.0.0.1

# riak.port - int, the port on which every node is listening. It must match the one specified in the riak.conf file
# at the line "listener.protobuf.internal".
riak.port=10017

# riak.bucket_type - string, must match value of bucket type created during setup. See readme.md for more information
riak.bucket_type=ycsb

# riak.r_val - int, the R value represents the number of Riak nodes that must return results for a read before the read
# is considered successful.
riak.r_val=3

# riak.w_val - int, the W value represents the number of Riak nodes that must report success before an update is
# considered complete.
riak.w_val=3

# riak.read_retry_count - int, number of times the client will try to read a key from Riak.
riak.read_retry_count=5

# riak.wait_time_before_retry - int, time (in milliseconds) the client waits before attempting to perform another
# read if the previous one failed.
riak.wait_time_before_retry=200

# riak.transaction_time_limit - int, time (in seconds) the client waits before aborting the current transaction.
riak.transaction_time_limit=10

# riak.strong_consistency - boolean, indicates whether to use strong consistency (true) or eventual consistency (false).
riak.strong_consistency=true
```

之后，再将jar包重新打包，如果出错则去掉m使用`jar -cvf`：

```shell
jar -cvfm riak-binding-0.17.0.jar riak.properties META-INF/MANIFEST.MF site
```

之后，在`ycsb-0.17.0`目录下运行YCSB进行数据的插入：

```shell
bin/ycsb load riak -P workloads/workloada
```

![image-20220619105637278](http://img.singhe.art/image-20220619105637278.png)

最后，测试读取、更新、删除操作：

```shell
bin/ycsb run riak -P workloads/workloada
```

![image-20220619105756247](http://img.singhe.art/image-20220619105756247.png)
