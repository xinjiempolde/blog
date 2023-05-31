---
title: getopt详解
date: 2023-05-31 11:54:23
tags:
    - linux
categories:
    - linux
---

> 转载自[csdn](https://blog.csdn.net/c1523456/article/details/79173776#commentBox)

# 简介

getopt函数是命令行参数解析函数，在头文件`getopt.h`中，这里对齐用法做一个简单的介绍。



# 使用说明

getopt()函数将传递给mian()函数的argc,argv作为参数，同时接受字符串参数optstring -- optstring是由选项Option字母组成的字符串。关于optstring的格式规范简单总结如下：
(1) 单个字符，表示该选项Option不需要参数。
(2) 单个字符后接一个冒号":"，表示该选项Option需要一个选项参数Option argument。选项参数Option argument可以紧跟在选项Option之后，或者以空格隔开。选项参数Option argument的首地址赋给optarg。
(3) 单个字符后接两个冒号"::"，表示该选项Option的选项参数Option argument是可选的。当提供了Option argument时，必须紧跟Option之后，不能以空格隔开，否则getopt()会认为该选项Option没有选项参数Option argument，optarg赋值为NULL。相反，提供了选项参数Option argument，则optarg指向Option argument。

为了使用getopt()，我们需要在while循环中不断地调用直到其返回-1为止。每一次调用，当getopt()找到一个有效的Option的时候就会返回这个Option字符，并设置几个全局变量。
getopt()设置的全局变量包括：
char *optarg  －－ 当匹配一个选项后，如果该选项带选项参数，则optarg指向选项参数字符串；若该选项不带选项参数，则optarg为NULL；若该选项的选项参数为可选时，optarg为NULL表明无选项参数，optarg不为NULL时则指向选项参数字符串。

int optind  －－ 下一个待处理元素在argv中的索引值。即下一次调用getopt的时候，从optind存储的位置处开始扫描选项。当getopt()返回-1后，optind是argv中第一个Operands的索引值。optind的初始值为1。
int opterr  －－ opterr的值非0时，在getopt()遇到无法识别的选项，或者某个选项丢失选项参数的时候，getopt()会打印错误信息到标准错误输出。opterr值为0时，则不打印错误信息。
int optopt  －－ 在上述两种错误之一发生时，一般情况下getopt()会返回'?'，并且将optopt赋值为发生错误的选项。 



# 一个简单的例子

```c++
int main(int argc, char *argv[])
{
  const char *unix_socket_path = nullptr;
  const char *server_host = "127.0.0.1";
  int server_port = PORT_DEFAULT;
  int opt;
  extern char *optarg;
  while ((opt = getopt(argc, argv, "s:h:p:")) > 0) {
    switch (opt) {
      case 's':
        unix_socket_path = optarg;
        break;
      case 'p':
        server_port = atoi(optarg);
        break;
      case 'h':
        server_host = optarg;
        break;
    }
  }
}
```

