---
title: forward和move
date: 2024-07-28 11:33:46
tags:
    - cpp
categories:
    - cpp
---

既然左值引用和右值引用都是地址，那么它们和指针有什么区别呢？引用可以看做是被限制的指针，和普通指针的区别在于，引用只能在声明的时候初始化，且不可更改。可以发现，引用和指针常量在功能上是等同的。

<!--more-->

# std::move

```cpp
//函数1
void function(int&& t) {
    std::cout<<"rvalue\n";
    /*...*/
}
//函数2
void function(int& t){
    std::cout<<"lvalue\n";
    /*...*/
}

int main()
{
    int&& a =2;
    function(a);//lvalue
    return 0;
}
```

假设没有`std::move`，如果我们想用函数1来处理a对应的资源数据，是不可能实现的。根据前面的讲解我们可以知道，如果某块资源能够通过变量名获取，那么这块资源是可取地址的，换句话说，它是一个左值引用。因此当我们把一个引用名传递给`function()`，调用的始终是函数2。`std::move`的作用就是告诉编译器，用函数1来处理引用a指向的数据。



# std::forward

首先解释一下什么是完美转发，它指的是函数模板可以将自己的参数“完美”地转发给内部调用的其它函数。所谓完美，即不仅能准确地转发参数的值，还能保证被转发参数的左、右值属性不变。举个例子（注意！本例中使用右值的方式并没有提高运行效率，因为它不满足我们前面提到的三个要求，这样写只是为了说明完美转发的作用）

```c++
//函数1
void function(int&& t) {
    std::cout<<"rvalue\n";
    /*...*/
}
//函数2
void function(int& t){
    std::cout<<"lvalue\n";
    /*...*/
}

int main()
{
    int n = 10;
    int & num = n;
    function(num); // 输出lvalue
    int && num2 = 11;
    function(num2); // 输出lvalue
    return 0;
}
```

我们期望`function(num2);`输出`rvalue`，但实际上输出的是`lvalue`，这是因为右值引用`num2`被`11`初始化后，`num2`就是一个有名称、或者说可以取地址的值了，换句话说，经过编译后，`num2`由右值引用变成了左值引用。我们修改一下代码：

```c++
int main()
{
    int n = 10;
    int & num = n;
    function(std::forward<int>(num)); // 输出lvalue
    int && num2 = 11;
    function(std::forward<int>(num2)); // 输出rvalue
    return 0;
}
```

结果与预期相符，也就是说完美转发能够保证左值引用在传递过程中始终是左值引用，右值引用在传递过程中始终是右值引用。其实我们把代码修改如下，也能得到同样的输出结果：

```c++
int main()
{
    int n = 10;
    int & num = n;
    function(num); // 输出lvalue
    int && num2 = 11;
    function(std::move(num2)); // 输出rvalue
    return 0;
}
```

如果我们想调用函数1，直接把变量名传递给`function`，如果我们想调用函数2，对变量名`move`操作后再传递给`function`就可以了，代码也很简洁，既然如此，为什么还要引入`std::forward`？其实在非模板编程中，由于我们很容易知道每个引用到底是左值引用还是右值引用，完全可以不用`std::forward`；但是在模板编程中，经过层层引用折叠之后，我们很难知道某个类型是左值引用类型还是右值引用类型，`std::forward`就是为了简化模板编程而出现的
