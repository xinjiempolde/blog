---
title: std::future
date: 2023-02-22 11:33:46
tags:
    - C++
categories:
    - coding
---

参考：[https://blog.csdn.net/lizhichao410/article/details/123732787](https://blog.csdn.net/lizhichao410/article/details/123732787)

# 使用背景

在C++中使用std::thread可创建一个线程，使用互斥量std::mutex来确保多个线程对共享数据的读写操作同步问题;使用std::condition_variable来解决线程执行顺序的同步问题。那么多个线程之间怎么传递数据呢？其中一种做法是使用一个全局变量，然后让线程之间互斥的访问就可以传递数据，还有一种做法就是使用std::future和std::async。

<!--more-->



# 同步操作和异步操作

同步操作：在函数调用时，在没有得到结果之前该调用不返回，由调用者主动等待这个调用的结果。

异步操作：在函数调用之后，这个调用就直接返回了，没有返回结果。即当一个异步过程调用发出后，调用者不会立刻得到结果。



# std::future

std::future是一个类模板，用来保存一个异步操作的结果，即这是一个未来值，只能在未来某个时候进行获取。

- get()：等待异步执行结果并返回结果，若得不到结果就会一直等待。
- wait():用于等待异步操作执行结束，但并不返回结果
- wait_for():阻塞当前进程，等待异步任务运行一段时间后返回其状态std::future_status，状态是枚举类型：
  - std::future_status::deferred:异步操作还没有开始
  - std::future_status::ready:异步操作已经完成
  - std::future_status::timeout:异步操作超时



# std::async

std::async是一个函数模板，用来启动一个异步任务，和std::thread类似，但std::async是更高级的抽象，异步返回结果保存在std::future中，使用者可以不必进行线程细节的管理。std::async有两种启动策略：

- std::launch::async : 函数必须以异步方式运行，即创建新的线程。
- std::launch::deferred: 函数只有在std::async所返回的std::future进行get()或wait()调用时才执行，并且调用方会阻塞至运行结束，否则不执行。

若没有指定策略，则会执行默认策略，将会由操作系统决定是否启动新的线程。



# 代码示例

```c++
#include <iostream>
#include <thread>
#include <future>

int getDataDemo() {
  std::cout << "start data query, " << "threadID: " << std::this_thread::get_id() << std::endl;
  std::this_thread::sleep_for(std::chrono::seconds(5));
  return 100;
}

int main() {
  std::cout << "execute data query, " << "threadID: " << std::this_thread::get_id() << std::endl;
  std::future<int> m_future = std::async(std::launch::async, getDataDemo);
  std::future_status m_status;
  do {
    m_status = m_future.wait_for(std::chrono::seconds(1));
    switch (m_status) {
      case std::future_status::ready:
        std::cout << "data query complete, " << "threadID: " << std::this_thread::get_id() << std::endl;
        break;
      case std::future_status::timeout:
        std::cout << "data query is running, " << "threadID: " << std::this_thread::get_id() << std::endl;
        break;
      case std::future_status::deferred:
        std::cout << "data query delay, " << "threadID: " << std::this_thread::get_id() << std::endl;
        m_future.wait();
        break;
      default:
        break;
    }
  } while (m_status != std::future_status::ready);
  int ret = m_future.get();
  std::cout << "data query result, " << ret << "threadID: " << std::this_thread::get_id() << std::endl;
}

```

