---
title: centos7.6安装postreSQL9.2
date: 2024-06-27 20:47:24
tags:
categories:
    - openGauss
---

# 安装步骤

## 1. 添加yum 配置

```shell
yum install -y http://download.postgresql.org/pub/repos/yum/9.2/redhat/rhel-7-x86_64/pgdg-centos92-9.2-3.noarch.rpm
```

<!--more-->

## 2. 安装服务

```shell
yum install -y postgresql92-server postgresql92-contrib
```

## 3. 启动

```shell
#执行初始化
/usr/pgsql-9.2/bin/postgresql92-setup initdb

#重新启动
systemctl restart postgresql-9.2.service
systemctl status postgresql-9.2.service
```

## 4. 调整配置文件

```shell
# 59 行左右, 修改监听IP, 端口
vim /var/lib/pgsql/9.2/data/postgresql.conf　　

# 82 行左右修改成 trust
vim /var/lib/pgsql/9.2/data/pg_hba.conf

host  all       all       127.0.0.1/32      trust

# 重启
systemctl restart postgresql-9.2
```

## 5. 创建用户

```shell
create user singheart with password 'Test@123';
GRANT ALL PRIVILEGES TO singheart LOGIN;
ALTER ROLE test_user LOGIN;
GRANT ALL PRIVILEGES ON DATABASE testdb TO singheart;
```

## 6. 连接

```shell
psql -p 11111 -h 127.0.0.1 -d postgres
```

注意需要把-h参数加上，否则会出现`psql: could not connect to server: No such file or directory, Is the server running locally and accepting connections on Unix domain socket "/run/postgresql/.s.PGSQL.11111"?`这个问题