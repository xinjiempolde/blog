---
title: false sharing
date: 2023-06-09 20:10:45
tags:
categories:
    - linux
---



> 本文转载自[https://blog.csdn.net/qq_28119741/article/details/102815659](https://blog.csdn.net/qq_28119741/article/details/102815659)



# 什么是false sharing

这里需要解决这几个问题

（1）什么是cpu缓存行

（2）什么是内存屏障

（3）什么是伪共享

（4）如何避免伪共享

<!--more-->

# CPU缓存架构

cpu是计算机的心脏，所有运算和程序最终都要由他来执行。

主内存RAM是数据存在的地方，CPU和主内存之间有好几级缓存，因为即使直接访问主内存相对来说也是非常慢的。

如果对一块数据做相同的运算多次，那么在执行运算的时候把它加载到离CPU很近的地方就有意义了，比如一个循环计数，你不想每次循环都到主内存中去取这个数据来增长它吧。



![img](http://img.singhe.art/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy8yOTY5MDYzLWIwYTI2MzVhNDM4ZGZlMDUucG5n)

越靠近CPU的缓存越快也越小

所以L1缓存很小但很快，并且紧靠着在使用它的CPU内核。

L2大一些，但也慢一些，并且仍然只能被一个单独的CPU核使用

L3在现代多核机器中更普遍，仍然更大，更慢，并且被单个插槽上的所有CPU核共享。

最后，主内存保存着程序运行的所有数据，它更大，更慢，由全部插槽上的所有CPU核共享。

当CPU执行运算的时候，它先去L1查找所需的数据，再去L2，然后L3，最后如果这些缓存中都没有，所需的数据就要去主内存拿。

走得越远，运算耗费的时间就越长。所以如果进行一些很频繁的运算，要确保数据在L1缓存中。

# CPU缓存行

缓存是由缓存行组成的，通常是64字节（常用处理器的缓存行是64字节的，比较旧的处理器缓存行是32字节的），并且它有效地引用主内存中的一块地址。

一个java的long类型是8字节，因此在一个缓存行中可以存8个long类型的变量

![img](http://img.singhe.art/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy8yOTY5MDYzLTE3M2RiMTU5ZDdjY2Q5NzYucG5n)

在程序运行的过程中，缓存每次更新都从主内存中加载连续的64个字节。因此，如果访问一个long类型的数组时，当数组中的一个值被加载到缓存中时，另外7个元素也会被加载到缓存中。但是，如果使用的数据结构中的项在内存中不是彼此相邻的，比如链表，那么将得不到免费缓存加载带来的好处。

不过，这种免费加载也有一个坏处。设想如果我们有个long类型的变量a，它不是数组的一部分，而是一个单独的变量，并且还有另外一个long类型的变量b紧挨着它，那么当加载a的时候将免费加载b。

看起来似乎没有什么问题，但是如果一个cpu核心的线程在对a进行修改，另一个cpu核心的线程却在对b进行读取。当前者修改a时，会把a和b同时加载到前者核心的缓存行中，更新完a后其它所有包含a的缓存行都将失效，因为其它缓存中的a不是最新值了。而当后者读取b时，发现这个缓存行已经失效了，需要从主内存中重新加载。

请记着，我们的缓存都是以缓存行作为一个单位来处理的，所以失效a的缓存的同时，也会把b失效，反之亦然。

![img](http://img.singhe.art/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy8yOTY5MDYzLTIzNDExZTFlYTA3ZDhhMTIucG5n)

这样就出现了一个问题，b和a完全不相干，每次却要因为a的更新需要从主内存重新读取，它被缓存未命中给拖慢了。这就是传说中的伪共享。

# 伪共享

当多线程修改互相独立的变量时，如果这些变量共享同一个缓存行，就会无意中影响彼此的性能，这就是伪共享。

```c++
public class FalseSharingTest {
 
    public static void main(String[] args) throws InterruptedException {
        testPointer(new Pointer());
    }
 
    private static void testPointer(Pointer pointer) throws InterruptedException {
        long start = System.currentTimeMillis();
        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 100000000; i++) {
                pointer.x++;
            }
        });
 
        Thread t2 = new Thread(() -> {
            for (int i = 0; i < 100000000; i++) {
                pointer.y++;
            }
        });
 
        t1.start();
        t2.start();
        t1.join();
        t2.join();
 
        System.out.println(System.currentTimeMillis() - start);
        System.out.println(pointer);
    }
}
 
class Pointer {
    volatile long x;
    volatile long y;
}
```



上面这个例子，我们声明了一个Pointer的类，它包含了x和y两个变量（必须声明为volatile，保证可见性，关于内存屏障的东西我们后面再讲），一个线程对x进行自增1亿次，一个线程对y进行自增1亿次。

可以看到，x和y完全没有任何关系，但是更新x的时候会把其它包含x的缓存行失效，同时y也就失效了，运行这段程序输出的时间为3890ms。

# 如何避免

伪共享的原理我们知道了，一个缓存行是64字节，一个long类型是8个字节，所以避免伪共享也很简单，大概有以下三种方式：

（1）在两个long类型的变量之间再加7个long类型

我们把上面的pointer改成下面这个结构

```c++
class Pointer {
    volatile long x;
    long p1, p2, p3, p4, p5, p6, p7;
    volatile long y;
}
```

再次运行程序，会发现输出时间神奇的缩短为695ms

（2）重新创建自己的long类型，而不是java自带的long修改Pointer如下

```c++
class Pointer {
    MyLong x = new MyLong();
    MyLong y = new MyLong();
}
 
class MyLong {
    volatile long value;
    long p1, p2, p3, p4, p5, p6, p7;
}
```

同时把pointer.x++改为pointer.x.value++;等，再次运行程序发现时间是724ms，这样本质上还是填充。

（3）使用@sun.misc.Contended注解（java8）

```c++
@sun.misc.Contended
class MyLong {
    volatile long value;
}
```

默认使用这个注解是无效的，需要在JVM启动参数加上-XX:-RestrictContended才会生效,再次运行程序发现时间是718ms。注意，以上三种方式中的前两种是通过加字段的形式实现的（上面go代码里的实现也是这样的），加的字段又没有地方使用，可能会被jvm优化掉，所以建议使用第三种方式。

内存屏障
1.volatile是一个类型修饰符，volatile的作用是作为指令关键字，确保本条指令不会因编译器的优化而省略。

2.volatile的特性：

（1）保证了不同线程对这个变量进行操作时的可见性，即一个线程修改了某个变量的值，这新值对其它线程来说是立即可见的-》实现可见性

（2）禁止进行指令重排序（实现有序性）

（3）volatile只能保证对单次读写的原子性。i++这种操作不能保证原子性

3.volatile的实现原理中的可见性就是基于内存屏障实现

内存屏障（Memory Barrier）：又称内存栅栏，是一个CPU指令。

在程序运行时，为了提高执行性能，编译器和处理器会对指令进行重排序，JVM为了保证在不同的编译器和CPU上有相同的结果，通过插入特定类型的内存屏障来禁止特定类型的编译器重排序和处理器重排序，插入一条内存屏障会告诉编译器和CPU：不管什么指令都不能和这条内存屏障指令重排序

# 总结
（1）CPU具有多级缓存，越接近CPU的缓存越小也越快

（2）CPU缓存中的数据是以缓存行为单位处理的；

（3）CPU缓存行能带来免费加载数据的好处，所以处理数据性能非常高

（4）CPU缓存行也带来了弊端，多线程处理不相干的变量时会相互影响，也就是伪共享

（5）避免伪共享的主要思路就是让不相干的变量不要出现在同一个缓存行中；

1是每两个变量之间加上7个long类型；2是创建自己的long类型，而不是用原生的；3是使用java8的注解
