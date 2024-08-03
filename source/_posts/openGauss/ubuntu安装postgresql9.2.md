# 一、安装并配置，并设置远程登陆的用户名和密码

1、安装postgreSQL

```shell
sudo add-apt-repository "deb http://apt.postgresql.org/pub/repos/apt/ xenial-pgdg main"
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -

sudo apt-get update

sudo apt-get install postgresql-9.6
```

- 在Ubuntu下安装Postgresql后，会自动注册为服务，并随操作系统自动启动。
- 在Ubuntu下安装Postgresql后，会自动添加一个名为postgres的操作系统用户，密码是随机的。并且会自动生成一个名字为postgres的数据库，用户名也为postgres，密码也是随机的。

2、修改postgres数据库用户的密码为manage

打开客户端工具（psql）

sudo -u postgres psql

- 其中，sudo -u postgres 是使用postgres 用户登录的意思
- PostgreSQL数据默认会创建一个postgres的数据库用户作为数据库的管理员，密码是随机的

***\*postgres=#\** ALTER USER postgres WITH PASSWORD 'manage';** 

- postgres=#为PostgreSQL下的命令提示符，--注意最后的分号；

3、退出PostgreSQL psql客户端

***\*postgres=#\** \q**

4、修改[ubuntu](https://so.csdn.net/so/search?q=ubuntu&spm=1001.2101.3001.7020)操作系统的postgres用户的密码（密码要与数据库用户postgres的密码相同）

切换到root用户

su root

删除PostgreSQL用户密码

**sudo passwd -d postgres**

- passwd -d 是清空指定用户密码的意思

设置PostgreSQL系统用户的密码

**sudo -u postgres passwd**

按照提示，输入两次新密码

- 输入新的 UNIX 密码
- 重新输入新的 UNIX 密码
- passwd：已成功更新密码

5、修改PostgresSQL数据库配置实现远程访问

**vi /etc/postgresql/9.6/main/postgresql.conf**

1.监听任何地址访问，修改连接权限

**#listen_addresses = 'localhost' 改为 listen_addresses = '\*'**

2.启用密码验证

**#password_encryption = on 改为 password_encryption = on**

**vi /etc/postgresql/9.6/main/pg_hba.conf**

在文档末尾加上以下内容

**host all all 0.0.0.0 0.0.0.0 md5**

6、重启服务

/etc/init.d/postgresql restart

7、5432端口的防火墙设置

5432为postgreSQL默认的端口

**iptables -A INPUT -p tcp -m state --state NEW -m tcp --dport 5432 -j ACCEPT**

# 二、内部登录，管理数据库、新建数据库、用户和密码

1、登录postgre SQL数据库

**psql -U postgres -h 127.0.0.1**

2、创建新用户zhangps，但不给建数据库的权限

***\*postgres=#\** create user "zhangps" with password '123456' nocreatedb;**

- 用户名处是双引号

3、建立数据库，并指定所有者

***\**\*postgres=#\*\**\*create database "testdb" with owner = "zhangps";**

# 三、外部登录，管理数据库、新建数据库、用户和密码

1、在外部命令行的管理命令，创建用户pencil

**sudo -u postgres createuser -D -P pencil**

- 输入新的密码:
- 再次输入新的密码:

2、建立数据库(tempdb)，并指定所有者为（pencil）

**sudo -u postgres createdb -O pencil tempdb**

- -O设定所有者为pencil

postgres的 日志目录，

/var/lib/postgresql/9.6/main

如果不修改日志目录，则应该在

/var/log/postgresql中

在目录**/etc/postgresql/9.6/main/postgresql.conf**

**可以修改日志，重新定向目录为\**/var/lib/postgresql/9.6/main\****

***\**\*log_destination = 'stderr'\*\*
\*\*logging_collector = on\*\*
\*\*log_directory = 'pg_log'\*\*
\*\*log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'\*\*
\*\*log_rotation_age = 1d\*\*
\*\*log_rotation_size = 100MB\*\*
\*\*log_min_messages = info\*\**\***