---
title: MineCraft在linux上开服
date: 2022-05-31 18:41:25
tags:
    - MineCraft
categories:
    - linux

---



# 环境配置

发行版为centos 7.6

现在更新一下环境

```bash
yum update -y
```

然后查看一下是否有`screen`和`java`

```bash
screen -version
java -version
```

安装`screen`和`java`

```bash
yum install screen
wget https://download.oracle.com/java/17/latest/jdk-17_linux-x64_bin.rpm
rpm -i jdk-17_linux-x64_bin.rpm
```

<!--more-->

# 安装和配置MineCraft服务器

```bash
mkdir -p /usr/MinecraftServer
cd /usr/MinecraftServer
wget https://mohistmc.com/builds/1.19.2/mohist-1.19.2-50-server.jar
```

在同一个目录下，创建启动文件

```bash
vim start.sh
```

以下为start.sh的内容,`Xmx16G`设置最大的堆内存为16G,`Xms4G`设置初始堆内存为4G，当不够程序运行的时候逐渐增加，最大到16G：

```
java -Xmx16G -Xms4G -jar mohist-1.19.2-50-server.jar
stty echo
```

# 使用Screen

创建新的Screen会话

```bash
screen -S MinecraftServer
```

查看screen会话，得到会话ID号

```bash
screen -ls
```

进入screen会话

```bash
screen -r 25519
```

启动server

```bash
chmod  +x start.sh
./start.sh
```

初次启之后需要同意协议，等待要让同意协议的时候，键入`true`并按下回车继续

更改`server.properties`，让非正版用户能够登陆。将`online-mode=true`修改为`online-mode=false`

```bash
online-mode=false
```