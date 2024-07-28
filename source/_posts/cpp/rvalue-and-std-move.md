---
title: rvalue_and_std::move
date: 2022-08-03 21:05:20
tags:
    - cpp
categories:
    - cpp
---

> 参考文章[https://www.cprogramming.com/c++11/rvalue-references-and-move-semantics-in-c++11.html](https://www.cprogramming.com/c++11/rvalue-references-and-move-semantics-in-c++11.html)

# 问题引出

<!--more-->

## 一个例子

`problem.cc`

```c++
#include <iostream>
#include <string.h>
class String {
public:
  String(const char *s) {
    std::cout << "initlize\n";
    size_ = strlen(s) + 1;
    data_ = new char[size_];
    strcpy(data_, s);
  }

  ~String() {
    std::cout << "call deconstructor\n";
    delete []data_;
  }

  // copy
  String(const String &s) {
    std::cout << "call copy constructor\n";
    size_ = s.size_;
    data_ = new char[size_];
    memcpy(data_, s.data_, size_);
  }

  friend std::ostream& operator<<(std::ostream& out, const String &s) {
    if (s.size_ == 0 || s.data_ == NULL) {
      return out;
    }
    out << s.data_;
    return out;
  }
public:
  int size_;
  char *data_;
};

String test() {
    String str("hello");
    return str;
}
int main(void) {
    String s = test();
    std::cout << "in main function\n";
    return 0;
}
```

让我们编译运行一下：

> 需要注意使用编译选项 -fno-elide-constructors关闭RVO(Return Value Optimization)，否则现在的编译器会给你自动优化

```shell
 g++ problem.cc -o a.out -fno-elide-constructors
```

运行的结果如下：

```
第一行 initlize
第二行 call copy constructor
第三行 call deconstructor
第四行 call copy constructor
第五行 call deconstructor
第六行 in main function
第七行 call deconstructo
```

我们可以看到该程序调用了两次拷贝构造函数和三次析构函数。那么这个程序的流程是怎么样的呢？下面作者将对每一行输出做出解释。



## 输出解释

### 第一行

在调用`test`函数之后，`String str("hello")`会调用构造函数构造对象`str` ，对应输出的结果`initlize`。

### 第二行

当`test`函数返回str的时候，它返回的是一个临时对象，这个临时对象是通过对`str`拷贝而构造的，而不是直接返回的`str`本身。因为`str`只是`test`函数中的局部变量，它的作用域只在`test`函数间，不可能跳到函数外再传给主函数中的`s`，因此`test`函数返回的只是一个临时变量。

### 第三行

上面已经阐明，`test`函数返回的只是一个临时变量。所以当`test`函数退出的时候，局部变量`str`将会被析构。对应第三行的输出。

### 第四行

在`main`函数中，`test`返回的临时变量会通过拷贝构造函数赋值给`s`，因此对应第四行中的拷贝构造函数。

### 第五行

当临时变量复制给`s`之后，这个临时变量也就被析构了。因此对第五行的输出。

### 第六行

本行的输出在临时变量析构之后发生，在`s`析构之前输出。

### 第七行

`main`函数中的局部变量`s`析构。

# 问题说明

我们可以看到，`test`函数本来应该返回的`str`，但其实返回的是一个临时变量，这就导致了不必要的拷贝构造。

那么我们有什么办法来解决这个问题呢？将`test`函数的返回值变成引用类型`String &`行不行呢？

这显然是不行的，因为`str`只是一个临时变量，当`test`函数结束的时候就会被释放，如果引用一个将被释放的值可能会出错。

因此对于临时变量这一问题，`C++`引入了右值(rvalue)和`std::move`这两个概念。



# 右值(rvalue)和std::move

什么是右值呢，它与左值相对于。左值也就是我们的变量之类的，它可以放在等号的左边。而右值可以理解为临时变量，`C++`使用&&作为右值引用的符号。而std::move可以将左值变为右值。

算了，不想写了，就这样吧。把代码贴上。

```c++
#include <iostream>
#include <string.h>
#include <string>
class String {
public:
  String(const char *s) {
    std::cout << "initlize\n";
    size_ = strlen(s) + 1;
    data_ = new char[size_];
    strcpy(data_, s);
  }

  ~String() {
    std::cout << "deallocate string:" << "\n";
    delete []data_;
  }

  // copy
  String(const String &s) {
    size_ = s.size_;
    data_ = new char[size_];
    memcpy(data_, s.data_, size_);
    std::cout << "allocate and copy string\n";
  }

  // move
  String(String &&s) {
    size_ = s.size_;
    data_ = s.data_;
    s.size_ = 0;
    s.data_ = nullptr;
  }

  friend std::ostream& operator<<(std::ostream& out, const String &s) {
    if (s.size_ == 0 || s.data_ == NULL) {
      return out;
    }
    out << s.data_;
    return out;
  }
public:
  int size_;
  char *data_;
};

String test() {
  String str("test");
  return str;
}

int main(void) {
  String s("hello");
  // 调用move constructor
  String b = std::move(s);
  // s的内容被移走了
  std::cout << b << ", " << s << "\n";

  // std::string也同样实现了move constructor
  std::string ss = "hle";
  std::string bb(std::move(ss));
  std::cout << bb << std::endl;
  String a("hello");
  return 0;
}
```



