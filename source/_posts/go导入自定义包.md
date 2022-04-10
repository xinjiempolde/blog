---
title: go导入自定义包
date: 2022-04-08 18:19:52
tags:
    - go语言
    - 编程
categories:
    - GO
---

> 本文参考[CSDN博客](https://blog.csdn.net/deroy/article/details/123021040)

# 环境

- ubuntu18.04
- go 1.18

<!--more-->

# 项目结构

![](http://img.singhe.art/architecture.png)



# 创建go项目

创建一个项目目录`learn`，在`learn`目录下编辑`main.go`文件

```go
package main

import (
	"fmt"
	"learn/math"
)
func main()  {
	fmt.Println(math.Add(1, 3))
}
```



# 自定义包

在`learn`目录下创建`math`目录，在`math`目录下创建`math.go`文件

```go
package math
func Add(a int, b int) int {
	return a + b
}

func Sub(a int, b int) int {
	return a - b	
}

func Mul(a int, b int) int {
	return a * b
}

func Div(a int, b int) int {
	return a / b
}
```

这里需要注意的是：

1. 包目录下所有文件 `package` 必须和目录名保持一致，例如`math`目录下所有包文件 `*.go` 必须引用 `package math`.
2. 外部接口必须以大写字母开头，例如`Add()`,否则外部无法调用



# 导入自定义包

在main包里面导入自定义包，需通过mod命令初始化，将go项目添加到环境中去。

打开终端，进入hello目录

```go
go mod init hello
```

此刻生成go.mod文件便可以自由导入自定义包而不会找不到包了



main.go文件导入包格式如下

```go
package main

import (
	"fmt"
	"learn/math"
)
func main()  {
	fmt.Println(math.Add(1, 3))
}
```

go mod init learn的时候将learn目录导入到go环境查找目录中去了，所以导入的库目录需以"learn/“为查找路径，math目录在learn目录下，查找路径便是"learn/math”;
