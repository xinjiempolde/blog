---
title: openGauss通过docker运行
date: 2024-07-28 20:47:24
tags:
categories:
    - openGauss

---

> 参考[【openGauss】Ubuntu 三条命令装好 opengauss_ubuntu安装opengauss-CSDN博客](https://blog.csdn.net/dive668/article/details/117268140)

# 0x00 安装docker

```shell
curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun
```

如果官方docker源较慢，可以修改docker源。编辑`/etc/docker/daemon.json`，添加以下内容：

```json
{
    "registry-mirrors": ["https://docker.mirrors.ustc.edu.cn"]
}
```

<!--more-->

# 0x01 运行openGauss镜像

```shell
sudo docker run --name opengauss --privileged=true -d -p 5432:5432 -e GS_PASSWORD=Test@123 enmotech/opengauss:latest
```



# 0x02 进入容器

```shell
sudo docker exec -it opengauss bash
```



# 0x03 使用gsql连接oepnGauss

```shell
gsql
```

