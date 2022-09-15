---
title: condition-variable
date: 2022-08-13 23:24:25
tags:
    - C++
    - thread
categories:
    - C++
---

> 前面已经对mutex以及相关的lock进行了梳理，这里对thread和condition_variable做一个简单的梳理。

# # std::thread

通过该函数能够创建一个线程。

```c++
#include <iostream>
#include <thread>
void worker1_func() {
  for (int i = 0; i < 5; i++) {
    std::cout << "worker1\n";
  }
}
int main(void) {
  std::thread worker1(worker1_func);
  for (int i = 0; i < 5; i++) {
    std::cout << "main func\n";
  }
  worker1.join();
}
```

<!--more-->

通过以下命令编译执行:

```bash
g++ test.cc -o a.out -lpthread
./a.out
```

得到如下结果：

main func
main func
main func
worker1
worker1
worker1
worker1
worker1
main func
main func

那么worker1.join()的用处是什么呢？它的用处是执行这个语句的线程(主线程)必须等待woker1这个线程执行完成后才能继续执行，可以实现一个简单的线程同步。而更复杂的线程同步则需要通过condition_variable来实现。

# std::condition_variable

```c++
wait()
```

wait会阻塞本线程，直到有人用condition_variable将其唤醒或者复活条件成立。在阻塞的过程中会将锁释放，在苏醒后又会重新获取锁。
