---
title: foundationdb的安装和测试
date: 2023-02-22 15:39:24
tags:
    - foundationdb
categories:
    - database
---

# 安装

```shell
# 用这个版本
wget https://github.com/apple/foundationdb/releases/download/7.2.3/foundationdb-server_7.2.3-1_amd64.deb
wget https://github.com/apple/foundationdb/releases/download/7.2.3/foundationdb-clients_7.2.3-1_amd64.deb
# 安装
sudo dpkg -i foundationdb-clients_7.2.3-1_amd64.deb
sudo dpkg -i foundationdb-server_7.2.3-1_amd64.deb
# 卸载
sudo dpkg --purge foundationdb-server
sudo dpkg --purge foundationdb-clients


wget https://github.com/apple/foundationdb/releases/download/6.2.11/foundationdb-server_6.2.11-1_amd64.deb
wget https://github.com/apple/foundationdb/releases/download/6.2.11/foundationdb-clients_6.2.11-1_amd64.deb
sudo dpkg -i foundationdb-clients_6.2.11-1_amd64.deb
sudo dpkg -i foundationdb-server_6.2.11-1_amd64.deb
sudo dpkg --purge foundationdb-server
sudo dpkg --purge foundationdb-clients

```



查看服务状态

```shell
service foundationdb status
service foundationdb start
service foundationdb stop
```



安装完成之后会自动启动foundationdb，输入`fdbcli`之后就可以看到相关信息了。

```shell
singheart@FX504GE:~/cpl$ fdbcli
Using cluster file `/etc/foundationdb/fdb.cluster'.

The database is available.

Welcome to the fdbcli. For help, type `help'.
fdb> status

Using cluster file `/etc/foundationdb/fdb.cluster'.

Configuration:
  Redundancy mode        - single
  Storage engine         - memory-2
  Encryption at-rest     - disabled
  Coordinators           - 1
  Usable Regions         - 1

Cluster:
  FoundationDB processes - 1
  Zones                  - 1
  Machines               - 1
  Memory availability    - 7.7 GB per process on machine with least available
  Retransmissions rate   - 1 Hz
  Fault Tolerance        - 0 machines
  Server time            - 02/22/23 15:57:08

Data:
  Replication health     - Healthy
  Moving data            - 0.000 GB
  Sum of key-value sizes - 0 MB
  Disk space used        - 105 MB

Operating space:
  Storage server         - 1.0 GB free on most full server
  Log server             - 213.6 GB free on most full server

Workload:
  Read rate              - 22 Hz
  Write rate             - 0 Hz
  Transactions started   - 9 Hz
  Transactions committed - 0 Hz
  Conflict rate          - 0 Hz

Backup and DR:
  Running backups        - 0
  Running DRs            - 0

Client time: 02/22/23 15:57:08

fdb> 
```

这里也提供下卸载的命令：

```shell
sudo dpkg --purge foundationdb-server
sudo dpkg --purge foundationdb-clients
```



# 测试

## C-API简单测试

test_fdb.c

```c
#define FDB_API_VERSION 710

#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include <fdb_c.h>
#include <unistd.h>

FDBTransaction *tr = NULL;
FDBDatabase *db = NULL;
pthread_t netThread;

static void checkError(fdb_error_t errorNum) {
  if (errorNum) {
    fprintf(stderr, "Error (%d): %s\n", errorNum, fdb_get_error(errorNum));
    exit(errorNum);
  }
}

static void waitAndCheckError(FDBFuture *future) {
  checkError(fdb_future_block_until_ready(future));
  if (fdb_future_get_error(future) != 0) {
    checkError(fdb_future_get_error(future));
  }
}

static void runNetwork() { checkError(fdb_run_network()); }

void createDataInDatabase() {
  int committed = 0;
  /*  Create transaction. */
  checkError(fdb_database_create_transaction(db, &tr));

  while (!committed) {
    /* Create data */
    char *key1 = "Test Key1";
    char *val1 = "Test Value1";
    fdb_transaction_set(tr, key1, (int)strlen(key1), val1, (int)strlen(val1));

    /* Commit to database.*/
    FDBFuture *commitFuture = fdb_transaction_commit(tr);
    checkError(fdb_future_block_until_ready(commitFuture));
    if (fdb_future_get_error(commitFuture) != 0) {
      waitAndCheckError(
          fdb_transaction_on_error(tr, fdb_future_get_error(commitFuture)));
    } else {
      committed = 1;
    }
    fdb_future_destroy(commitFuture);
  }
  /* Destroy transaction. */
  fdb_transaction_destroy(tr);
}

void readDataFromDatabase() {
  FDBTransaction *tr = NULL;
  const uint8_t *value = NULL;
  fdb_bool_t valuePresent;
  int valueLength;
  char *key = "Test Key1";

  checkError(fdb_database_create_transaction(db, &tr));
  FDBFuture *getFuture = fdb_transaction_get(tr, key, (int)strlen(key), 0);
  waitAndCheckError(getFuture);

  checkError(
      fdb_future_get_value(getFuture, &valuePresent, &value, &valueLength));

  printf("Got Value from db. %s: '%.*s'\n", key, valueLength, value);
  fdb_transaction_destroy(tr);
  fdb_future_destroy(getFuture);
}

int main() {
  /* Default fdb cluster file. */
  char *cluster_file = "/etc/foundationdb/fdb.cluster";

  /* Setup network. */
  checkError(fdb_select_api_version(FDB_API_VERSION));
  checkError(fdb_setup_network());
  puts("Created network.");

  pthread_create(&netThread, NULL, (void *)runNetwork, NULL);

  checkError(fdb_create_database(cluster_file, &db));
  puts("Created database.");

  createDataInDatabase();
  readDataFromDatabase();


  puts("Program done. Now exiting...");
  fdb_database_destroy(db);
  db = NULL;
  checkError(fdb_stop_network());
  int rc = pthread_join(netThread, NULL);
  if (rc)
    fprintf(stderr, "ERROR: network thread failed to join\n");
  exit(0);
}
```

编译命令如下：

```shell
gcc test_fdb.c -lfdb_c -lpthread -I /usr/include/foundationdb/ -ggdb3 -O0 -g3 -o test_fdb
```



## YCSB测试

### 安装go环境

```shell
sudo snap install go
```



### 下载go-ycsb

```shell
git clone https://github.com/pingcap/go-ycsb.git
make
```



```shell
# 加载数据到foundationdb
./bin/go-ycsb load foundationdb -P workloads/workloadb -p threadcount=64 -p recordcount=1000000 -p operationcount=10000
# 运行测试
./bin/go-ycsb run foundationdb -P workloads/workloadb -p threadcount=256 -p recordcount=10000 -p operationcount=1000
```



## 多节点foundationdb

