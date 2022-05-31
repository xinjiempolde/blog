---
title: explicit关键字
date: 2022-05-31 11:33:07
tags:
    - C++
categories:
    - C++
---

> 本文参考自[知乎](https://zhuanlan.zhihu.com/p/137947734)

C++中explicit常用于构造函数前，防止因隐式转换而导致错误的发生。本文将通过构造函数带有和不带有explicit两个例子，从而说明explicit的作用

<!--more-->

# 没有explicit

以下代码为不含有`explicit`的示例。

```c++
#include <iostream>
class Foo {
 public:
  Foo(int a) {
      value_ = a;
  }

  int GetValue() {
      return value_;
  }

 private:
  int value_;
};
int main(void) {
    Foo instance(20);
    std::cout << "origin value:" << instance.GetValue() << std::endl;
    instance = 12;
    std::cout << "modifyed value:" << instance.GetValue() << std::endl;
    return 0;
}
```

运行程序，得到以下结果:

```
origin value:20
modifyed value:12
```

这里其实进行了隐式转换，`instance = 12`被隐式转换成为`instance = A tmp(2)`。



# 含有explicit的情况

以下代码为含有explicit的示例。

```c++
#include <iostream>
class Foo {
 public:
  explicit Foo(int a) {
      value_ = a;
  }

  int GetValue() {
      return value_;
  }

 private:
  int value_;
};
int main(void) {
    Foo instance(20);
    std::cout << "origin value:" << instance.GetValue() << std::endl;
    instance = 12;
    std::cout << "modifyed value:" << instance.GetValue() << std::endl;
    return 0;
}
```

当编译的时候，出现下面的错误：

```shell
test.cc: In function ‘int main()’:
test.cc:18:16: error: no match for ‘operator=’ (operand types are ‘Foo’ and ‘int’)
     instance = 12;
                ^~
test.cc:2:7: note: candidate: constexpr Foo& Foo::operator=(const Foo&)
 class Foo {
       ^~~
test.cc:2:7: note:   no known conversion for argument 1 from ‘int’ to ‘const Foo&’
test.cc:2:7: note: candidate: constexpr Foo& Foo::operator=(Foo&&)
test.cc:2:7: note:   no known conversion for argument 1 from ‘int’ to ‘Foo&&’
```

因此，再加上explicit后，将不会进行隐式转换。

# 总结

explicit关键字的作用就是防止类构造函数的隐式自动转换。

explicit在下面两种情况下有效：

- 类的构造函数只有一个参数时；
- 类的构造函数中除了第一个参数以外，其他参数都有默认值的时候。(第一个参数可以有默认值，也可以没有)

google C++规范中也约定所有单参数的构造函数都必须是显示的，即使用explicit关键字。
