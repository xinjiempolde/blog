---
title: memory model
date: 2022-09-15 10:44:52
tags:
    - memory_model
categories:
    - linux
---

# memory consistency model

![image-20220903234055396](http://img.singhe.art/image-20220903234055396.png)

由于一些优化（可能来自编译器或者操作系统或者硬件之类的），在核1真正执行的时候可能不会按照程序原有的顺序执行，比如core1会将两个操作交换顺序，这就导致多个核之间的执行顺序可能会任意进行。为了让结果符合预期，就产生了memory consistency model这一概念，以及各种不同的memory model。

<!--more-->

# Sequential consistency model

顺序一致性有两点要求：

- 在同一个核里的指令顺序与程序顺序相同（也就是不能交换指令改变指令顺序）
- 在不同核之间可以按照任意的顺序进行。

这就好像洗牌的时候将两堆扑克牌合起来一样，每堆扑克牌各自的顺序保持不变，但是两堆扑克牌合起来的顺序可以任意进行。

![puke](http://img.singhe.art/puke.jpeg)

# references

- [https://www.cis.upenn.edu/~devietti/classes/cis601-spring2016/sc_tso.pdf](https://www.cis.upenn.edu/~devietti/classes/cis601-spring2016/sc_tso.pdf)

- [https://www.youtube.com/watch?v=AUxFuD_IfqA](https://www.youtube.com/watch?v=AUxFuD_IfqA)