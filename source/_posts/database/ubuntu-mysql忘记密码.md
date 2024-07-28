---
title: ubuntu_mysql忘记密码
date: 2023-02-17 11:01:03
tags:
    - linux
categories:
    - linux
---

# 前言

参考：[https://blog.csdn.net/m0_46825740/article/details/127768826](https://blog.csdn.net/m0_46825740/article/details/127768826)

我的系统是`ubuntu 18.04`，Windows用户可以划走了。`ubuntu 18.04`使用`sudo apt install mysql-server`安装的mysql版本是5.7，当忘记了密码的时候可以采取下面的方法。

<!--more-->

# 使用debian-sys-maint账号登录

```shell
$ sudo cat /etc/mysql/debian.cnf
# Automatically generated for Debian scripts. DO NOT TOUCH!
[client]
host     = localhost
user     = debian-sys-maint
password = xFaoTL9Jo2Q4QzVg
socket   = /var/run/mysqld/mysqld.sock
[mysql_upgrade]
host     = localhost
user     = debian-sys-maint
password = xFaoTL9Jo2Q4QzVg
socket   = /var/run/mysqld/mysqld.sock
```

使用该账号和密码进行登录:

```shell
mysql -u debian-sys-maint -p
```



# 修改root账号密码

```shell
use mysql;

update mysql.user set authentication_string=password('123456') where user='root' and Host='localhost';

update user set plugin='mysql_native_password';

flush privileges;
```
