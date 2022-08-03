---
title: std::mutex
date: 2022-05-31 18:41:25
tags:
    - C++
categories:
    - C++
---

> 本文转载自[CSDN](https://blog.csdn.net/weixin_42570248/article/details/100849100)

# std::mutex

互斥量是一个可以处于两态之一的变量:解锁和加锁。这样，只需要一个二进制位表示它，不过实际上，常常使用一个整型量，0表示解锁，而其他所有的值则表示加锁。互斥量使用两个过程。当一个线程(或进程)需要访问临界区时，它调用mutex_lock。如果该互斥量当前是解锁的(即临界区可用)，此调用成功，调用线程可以自由进入该临界区。

<!--more-->

```c++
#include <iostream>
#include <vector>
#include <thread>
#include <mutex>
 
class test
{
private:
	std::mutex my_mutex;
public:
	void function()
	{
		my_mutex.lock();
		std::cout << "hello world!" << " i love you!" << std::endl;
		// ...
		my_mutex.unlock();
	}
};
 
int main()
{
	test t;
	std::vector<std::thread> tvec;
	for (int i = 0; i < 5; ++i)
	{
		tvec.push_back(std::thread(&test::function, &t));
	}
	for (auto iter = tvec.begin(); iter != tvec.end(); ++iter)
	{
		if (iter->joinable())
		{
			iter->join();
		}
	}
	return 0;
}
```

互斥量的使用可以在各种方面，比如在对共享数据的读写上，倘若我们多个线程共享同一个数据，那么我们要想保证多线程安全，就必须对共享变量的读写上锁，从而保证线程安全。

***注意事项：***

***我们在使用mutex时，要时刻注意lock()与unlock()的加锁临界区的范围，不能太大也不能太小，太大了会导致程序运行效率低下，大小了则不能满足我们对程序的控制。并且我们在加锁之后要及时解锁，否则会造成死锁，lock()与unlock()应该是成对出现。***



# std::lock_guard

**为了防止我们在使用mutex的过程中意外忘记unlock()，引入了****std::lock_guard****的类模板，有了该类模板，我们就无需自己去控制对互斥量的加锁与解锁。**为了方便起见，我们针对上述代码进行替换，话不多说，放码过来：

```c++
#include <thread>
#include <mutex>
#include <iostream>
 
std::mutex my_mutex;
 
void function()
{
 
	//my_mutex.lock();
	//std::cout << "hello world!" << " i love you!" << std::endl;
	 ... do something
	//my_mutex.unlock();
 
    {
        std::lock_guard<std::mutex> locker(my_mutex);
        std::cout << "hello world!" << " i love you!" << std::endl;
	    // ... do something
    }
}
```

当我们执行到 "std::lock_guard<std::mutex> locker(my_mutex);" 这条语句时，便创建了locker对象与我们的互斥量my_mutex绑定在一起了。



# std::lock()

新的问题出现了，当我们在保护临界区时，有时可能需要获取多个互斥量的锁，这个时候便容易产生死锁现象，为了解决这一问题，我们引入了**std::lock()**，通过std::lock()函数我们可以一次性获取多个互斥量的锁，一旦存在获取不到的锁便全部释放，再次尝试获取：

```c++
#include <thread>
#include <mutex>
#include <iostream>
 
std::mutex my_mutex1;
std::mutex my_mutex2;
 
void function()
{
    std::lock(my_mutex1, my_mutex2);
    // ... do something
    my_mutex1.unlock();
    my_mutex2.unlock();
}
```



# std::adopt_lock

我们观察代码分析发现，问题再次回到了先前的情况：需要手动unlock！那么我们联想一下，能不能结合std::lock()与我们之前的std::lock_guard类模板，答案是肯定的！这里需要引入一个std::lock_guard的参数：**std::adopt_lock**

```c++
#include <thread>
#include <mutex>
#include <iostream>
 
std::mutex my_mutex1;
std::mutex my_mutex2;
 
void function()
{
    std::lock(my_mutex1, my_mutex2);
    std::lock_guard<std::mutex> locker1(my_mutex1, std::adopt_lock);
    std::lock_guard<std::mutex> locker2(my_mutex2, std::adopt_lock);
    // ... do something
}
```

并且我们通过代码调试发现，lock_guard类模板对象中没有任何成员函数！也就是说，仅有构造函数与析构函数。接下来我们从底层实现的角度来分析一下lock_guard，其实lock_guard并没有我们想象的那么神奇，它只是通过在构造函数中，完成对互斥量的加锁lock()，而在析构函数中完成对互斥量的解锁unlock()，是一种典型的RAII的机制。然后不同的重载构造函数有不同的实现，比如我们可以通过向构造函数参数中引入std::adpot_lock，这样在构造函数中就不会执行对互斥量的加锁lock()啦！这样的话，我们就通过简单的构造与析构，让我们对互斥量的加锁与解锁操作成对成对的出现啦！因此在使用lock_guard时有一个小技巧(一般人我不告诉他)，我们可以通过代码块的概念，来控制我们的临界区范围！

# std::unique_lock

***这里给出另一种模板类std::unique_lock，该模板类的特点相较于std::lock_guard而言，更加的灵活，弹性更高，同时也更加的消耗资源。***

```c++
std::unique_lock成员函数:
lock()->加锁
unlock()->解锁
try_lock()->尝试加锁，若拿到锁则返回true，否则返回false(该函数不阻塞)
release()->返回其管理的mutex的指针，并释放所有权
```

其中对于std::unique_lock存在这样的一个参数：**std::defer_lock**，当我们在构造unique_lock对象时若引入了该参数，则会保持其关联的mutex的状态:

```c++
#include <iostream>
#include <thread>
#include <mutex>
 
class test
{
private:
	std::mutex my_mutex;
public:
	void function()
	{
            std::unique_lock<std::mutex> locker(my_mutex, std::defer_lock);    // 此时保持my_mutex的状态(unlock)
            locker.lock();
            // ... do something
            locker.unlock();
            // ... do something
            locker.lock();
            // ... do something
            locker.unlock();
	}
};
 
int main()
{
	test t;
	std::vector<std::thread> tvec;
	for (int i = 0; i < 5; ++i)
	{
		tvec.push_back(std::thread(&test::function, &t));
	}
	for (auto iter = tvec.begin(); iter != tvec.end(); ++iter)
	{
		if (iter->joinable())
		{
			iter->join();
		}
	}
	return 0;
}
```

需要我们注意的是，我们的unique_lock一般与一个mutex所关联，它时刻记录着该mutex的状态，若该mutex为lock状态，则在其析构时便要完成对其的unlock，若该mutex为unlock状态，我们则可以通过该unique_lock对象去弹性的执行lock()。我们可以通过各种方法去移交unique_lock对象对mutex的控制权，下面给出一段代码：

```c++
#include <iostream>
#include <vector>
#include <thread>
#include <mutex>
 
class test
{
private:
	std::mutex my_mutex;
	std::unique_lock<std::mutex> return_unique_lock()
	{
		std::unique_lock<std::mutex> locker(my_mutex, std::defer_lock);
		return locker;
	}
public:
	void function()
	{
		std::unique_lock<std::mutex> locker = return_unique_lock();
		locker.lock();
		// ... do something
		std::cout << "I love China!" << std::endl;
		locker.unlock();
 
		std::unique_lock<std::mutex> new_locker(std::move(locker));
		// ... do something
	}
};
 
int main()
{
	test t;
	std::vector<std::thread> tvec;
	for (int i = 0; i < 5; ++i)
	{
		tvec.push_back(std::thread(&test::function, &t));
	}
	for (auto iter = tvec.begin(); iter != tvec.end(); ++iter)
	{
		if (iter->joinable())
		{
			iter->join();
		}
	}
	return 0;
}
```

观察上述代码，我们发现存在一个return_unique_lock()函数，该函数内部，创建了一个unique_lock对象与my_mutex互斥量绑定，然后返回该对象，显而易见，我们知道该函数返回的是一个临时值(右值)，此时观察function()函数内部，我们通过return_unique_lock()函数获取到的临时对象来初始化unique_lock对象，其中底层调用了unique_lock的移动构造函数，在该函数中，将临时对象对my_mutex的控制权移交给了function函数当中的locker对象。
