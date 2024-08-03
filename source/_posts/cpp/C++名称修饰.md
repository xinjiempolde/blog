---
title: C++名称修饰
date: 2024-07-028 19:49:11
tags:
    - cpp
categories:
    - cpp

---

![在这里插入图片描述](https://img.singhe.art/20200321173317697.png)

<!--more-->

c++filt 可用于解析 C++ 和 Java 中被修饰的符号，比如变量与函数名称。

我们知道， 在 C++ 和 Java 中， 允许函数重载，也就是说我们可以写出多个同名但参数类型不同的函数，其实现依赖于编译器的名字改编（Name Mangling）机制，即编译器会将函数的名称进行修饰，加入参数信息。考察如下程序：

```c++
//
//@file:print.cpp
//

#include <iostream>
#include <string>
using namespace std;

const int dTest=0;

void print(const string& strElfFileName) {
        std::cout<<"readelf "<<strElfFileName<<std::endl;
}

```

使用g++编译生成目标文件print.o

```c++
g++ -c print.cpp -o print.o
```

然后使用命令 strings 查找 print.o 中的可打印字符串。

```text
strings print.o
readelf 
GCC: (GNU) 7.3.0
print.cpp
_ZStL19piecewise_construct
_ZStL8__ioinit
_ZL5dTest
_Z41__static_initialization_and_destruction_0ii
_GLOBAL__sub_I__Z5printRKNSt7__cxx1112basic_stringIcSt11char_traitsIcESaIcEEE
_ZSt4cout
_ZStlsISt11char_traitsIcEERSt13basic_ostreamIcT_ES5_PKc
_ZStlsIcSt11char_traitsIcESaIcEERSt13basic_ostreamIT_T0_ES7_RKNSt7__cxx1112basic_stringIS4_S5_T1_EE
_ZSt4endlIcSt11char_traitsIcEERSt13basic_ostreamIT_T0_ES6_
_ZNSolsEPFRSoS_E
_ZNSt8ios_base4InitC1Ev
__dso_handle
_ZNSt8ios_base4InitD1Ev
__cxa_atexit
.symtab
.strtab
.shstrtab
.rela.text
.data
.bss
.rodata
.rela.init_array
.comment
.note.GNU-stack
.rela.eh_frame
```

使用c++filt还原函数名称：

```text
strings print.o | c++filt 
readelf 
GCC: (GNU) 7.3.0
print.cpp
std::piecewise_construct
std::__ioinit
dTest
__static_initialization_and_destruction_0(int, int)
_GLOBAL__sub_I__Z5printRKNSt7__cxx1112basic_stringIcSt11char_traitsIcESaIcEEE
std::cout
std::basic_ostream<char, std::char_traits<char> >& std::operator<< <std::char_traits<char> >(std::basic_ostream<char, std::char_traits<char> >&, char const*)
std::basic_ostream<char, std::char_traits<char> >& std::operator<< <char, std::char_traits<char>, std::allocator<char> >(std::basic_ostream<char, std::char_traits<char> >&, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const&)
std::basic_ostream<char, std::char_traits<char> >& std::endl<char, std::char_traits<char> >(std::basic_ostream<char, std::char_traits<char> >&)
std::basic_ostream<char, std::char_traits<char> >::operator<<(std::basic_ostream<char, std::char_traits<char> >& (*)(std::basic_ostream<char, std::char_traits<char> >&))
std::ios_base::Init::Init()
__dso_handle
std::ios_base::Init::~Init()
__cxa_atexit
.symtab
.strtab
.shstrtab
.rela.text
.data
.bss
.rodata
.rela.init_array
.comment
.note.GNU-stack
.rela.eh_frame
```