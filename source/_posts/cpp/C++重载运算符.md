---
title: C++重载运算符>>和<<
date: 2022-04-04 20:46:11
tags:
    - cpp
categories:
    - cpp
---

# 概述

要重载一个运算符，有两种方式：

- 作为成员方法，只需要一个参数。
- 作为全局函数，需要两个参数。

而`cin`和`cout`这两个对象我们无法对其内部成员进行修改，因此对`cin`和`cout`的重载只能使用第二种办法。

<!--more-->

# 代码示例

```C++
#include <iostream>

using namespace std;

class Rectangle {
private:
    int _height, _weight;
public:
    Rectangle(int height, int weight);
    ~Rectangle();
    friend ostream& operator<<(ostream& out, const Rectangle& rect);
    friend istream& operator>>(istream& in, Rectangle& rect);
};

Rectangle::Rectangle(int height, int weight): _height(height), _weight(weight){}
Rectangle::~Rectangle(){}

ostream& operator<< (ostream& out, const Rectangle& rect) {
    out << "height is:" << rect._height << " weight is:" << rect._weight << endl;
    return out;
}

istream& operator>>(istream& in, Rectangle& rect) {
    in >> rect._height >> rect._weight;
    return in;
}
int main() {
    Rectangle rect(3, 5);
    cout << rect;
    return 0;
}
```
