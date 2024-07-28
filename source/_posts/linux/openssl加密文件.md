---
title: openssl加密文件
date: 2023-06-27 22:43:41
tags:
categories:
    - linux
---

# 引言

为了让服务器上的别人不看我的代码，就决定对文件进行打包，然后对压缩包进行加密。

<!--more-->

# 压缩文件夹

解压

```shell
tar zxvf 文件名.tar.gz
```



压缩

```shell
tar -zcvf 文件名.tar.gz 待压缩的文件名
```



# 对文件进行加密

> 从[https://blog.csdn.net/qq_30624591/article/details/104791412](https://blog.csdn.net/qq_30624591/article/details/104791412)复制



## 使用命令生成私钥
```shell
openssl genrsa -out rsa_private_key.pem 1024
```


参数:

- genrsa 生成密钥 
- -out 输出到文件 
- rsa_private_key.pem 文件名 
- 1024 长度或者2048长度

## 从私钥中提取公钥

```shell
openssl rsa -in rsa_private_key.pem -pubout -out rsa_public_key.pem
```


参数: 

- rsa 提取公钥
-  -in 从文件中读入 
- rsa_private_key.pem 文件名 
- -pubout 输出
-  -out 到文件 
- rsa_public_key.pem 文件名

然后新建一个test.txt 内容是 helloworld



## 使用公钥加密

```shell
openssl rsautl -encrypt -in test.txt -inkey rsa_public_key.pem -pubin -out demo.en
```

参数: 

- rsautl 加解密 
- -encrypt 加密 -
- in 从文件输入 
- test.txt 文件名 
- -inkey 输入的密钥 
- rsa_public_key.pem 上一步生成的公钥 
- -pubin 表名输入是公钥文件 
- -out输出到文件 
- demo.en 输出文件名



## 使用私钥解密

```shell
openssl rsautl -decrypt -in demo.en -inkey rsa_private_key.pem -out demo.de
```


参数: 

- -decrypt 解密 
- -in 从文件输入
- demo.en 上一步生成的加密文件 
- -inkey 输入的密钥 
- rsa_private_key.pem 上一步生成的私钥 
- -out输出到文件 
- demo.de 输出的文件名
