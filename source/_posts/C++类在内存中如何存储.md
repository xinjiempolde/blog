---
title: C++类在内存中如何存储
date: 2022-08-03 14:09:29
tags:
    - C++
categories:
    - C++
---

> 参考[https://baijiahao.baidu.com/s?id=1653384309305389312&wfr=spider&for=pc](https://baijiahao.baidu.com/s?id=1653384309305389312&wfr=spider&for=pc)

# 背景

我们知道在C++中`char`占用一个字节，`int`占用四个字节。那么类在内存中是怎么布局的呢？成员变量和成员函数是放在一起存储的吗？

<!--more-->

# 空类

```c++
#include <iostream>
class A {};
int main(void) {
    A a;
    printf("size of a:%lu", sizeof(a));
    return 0;
}
```

其结果是`size of a: 1`，也就是空类占用一个字节。



# 仅有成员变量

```c++
#include <iostream>
class A {
public:
    int pub_i1;
    int pub_i2;
};
int main(void) {
    A a;
    printf("size of a:%lu\n", sizeof(a));
    printf("address of pub_i1:%p\n", &a.pub_i1);
    printf("address of pub_i2:%p\n", &a.pub_i2);
    return 0;
}
```

结果如下：

```
size of a:8
address of pub_i1:0x7fffe2502740
address of pub_i2:0x7fffe2502744
```

从结果可以知道，公共成员变量占用空间和结构体是一样的，其内存布局如下：

![](http://img.singhe.art/类内存布局.png)



# 非虚成员函数

```c++
#include <iostream>
class A {
public:
    int pub_i1;
    int pub_i2;
    void pub_func1() {}
    void pub_func2() {}
};
int main(void) {
    A a;
    printf("size of a:%lu\n", sizeof(a));
    printf("address of pub_i1:%p\n", &a.pub_i1);
    printf("address of pub_i2:%p\n", &a.pub_i2);
    printf("address of func1:%p\n", (void*)(&A::pub_func1));
    printf("address of func2:%p\n", (void*)(&A::pub_func2));
    return 0;
}
```

结果如下：

```c++
size of a:8
address of pub_i1:0x7ffe58501900
address of pub_i2:0x7ffe58501904
address of func1:0x563f174fa904
address of func2:0x563f174fa91
```

可以知道类的非虚成员函数并不和成员变量存储在一起，类中只存了成员变量，其内存布局如下：

![](http://img.singhe.art/非虚函数布局.png)

# 私有成员变量

```c++
#include <iostream>
class A {
public:
    int pub_i1;
    int pub_i2;

    int *GetI3Addr() { return &pri_i3; }
    int *GetI4Addr() { return &pri_i4; }
private:
    int pri_i3;
    int pri_i4;
};
int main(void) {
    A a;
    printf("size of a:%lu\n", sizeof(a));
    printf("address of pub_i1:%p\n", &a.pub_i1);
    printf("address of pub_i2:%p\n", &a.pub_i2);
    printf("address of pri_i3:%p\n", a.GetI3Addr());
    printf("address of pri_i4:%p\n", a.GetI4Addr());
    return 0;
}
```

结果如下：

```
size of a:16
address of pub_i1:0x7fff65848b00
address of pub_i2:0x7fff65848b04
address of pri_i3:0x7fff65848b08
address of pri_i4:0x7fff65848b0c
```

可以看到私有成员变量和公共成员变量在存储上没有区别，其内存布局如下：

![](http://img.singhe.art/私有成员函数布局.png)

# 虚函数

在上面代码的基础上，增加`virtual void pub_vfunc1(){}`这一个虚函数。

```c++
#include <iostream>
class A {
public:
    int pub_i1;
    int pub_i2;
    void pub_func1() {}
    void pub_func2() {}
    virtual void pub_vfunc1() {}
};
int main(void) {
    A a;
    printf("size of a:%lu\n", sizeof(a));
    printf("address of pub_i1:%p\n", &a.pub_i1);
    printf("address of pub_i2:%p\n", &a.pub_i2);
    printf("address of func1:%p\n", (void*)(&A::pub_func1));
    printf("address of func2:%p\n", (void*)(&A::pub_func2));
    return 0;
}
```

结果为：

```
size of a:16
address of pub_i1:0x7fff7adf7678
address of pub_i2:0x7fff7adf767c
address of func1:0x55c7089739c4
address of func2:0x55c7089739d0
```

从结果可以看出，虚函数占用了8个字节，那么我们再增加一个虚函数是否也会增加类的大小呢？

```c++
#include <iostream>
class A {
public:
    int pub_i1;
    int pub_i2;
    void pub_func1() {}
    void pub_func2() {}
    virtual void pub_vfunc1() {}
    virtual void pub_vfunc2() {}
};
int main(void) {
    A a;
    printf("size of a:%lu\n", sizeof(a));
    printf("address of pub_i1:%p\n", &a.pub_i1);
    printf("address of pub_i2:%p\n", &a.pub_i2);
    printf("address of func1:%p\n", (void*)(&A::pub_func1));
    printf("address of func2:%p\n", (void*)(&A::pub_func2));
    return 0;
}
```

结果为：

```
size of a:16
address of pub_i1:0x7ffe2a89ff18
address of pub_i2:0x7ffe2a89ff1c
address of func1:0x55c205a279e4
address of func2:0x55c205a279f0
```

可以看出，增加虚函数的数量不会对类的大小产生影响，它始终只会增加8个字节的大小，其内存布局如下：

![](http://img.singhe.art/虚表.png)

我们可以通过以下代码来验证：

```c++
#include <iostream>
class A {
public:
    int pub_i1;
    int pub_i2;
    void pub_func1() {}
    void pub_func2() {}
    virtual void pub_vfunc1() {}
    virtual void pub_vfunc2() {}
};
int main(void) {
    A a;
    printf("size of a:%lu\n", sizeof(a));
    printf("address of a:%p\n", &a);
    printf("address of pub_i1:%p\n", &a.pub_i1);
    printf("address of pub_i2:%p\n", &a.pub_i2);
    printf("address of pub_vfunc1:%p\n", (void*)(&A::pub_vfunc1));
    printf("address of pub_vfunc2:%p\n", (void*)(&A::pub_vfunc2));

    // 获取虚表指针
    unsigned long *_vptr = (unsigned long*)(&a);
    // 构建虚表
    unsigned long *table = (unsigned long*)*_vptr;
    printf("content of table[0]:%lx\n", (unsigned long)table[0]);
    printf("content of table[1]:%lx\n", table[1]);
    return 0;
}
```

结果如下：

```
size of a:16
address of a:0x7ffd49855ab0
address of pub_i1:0x7ffd49855ab8
address of pub_i2:0x7ffd49855abc
address of pub_vfunc1:0x55abbe9d3a4a
address of pub_vfunc2:0x55abbe9d3a56
content of table[0]:55abbe9d3a4a
content of table[1]:55abbe9d3a56
```

