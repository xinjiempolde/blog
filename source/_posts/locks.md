---
title: locks
date: 2023-12-13 01:02:25
tags: C++
categories:
      C++
---
# 0x00 背景介绍

多个线程访问同一个互斥资源的时候，需要保证数据的正确性，其中一个方案就是加锁。spinlock(自旋锁)尽可能的减少线程的阻塞，这对于锁的竞争不激烈，且占用锁时间非常短的代码块来说性能能大幅度的提升，因为自旋的消耗会小于线程阻塞挂起再唤醒的操作的消耗，这些操作会导致线程发生两次上下文切换！

但是如果锁的竞争激烈，或者持有锁的线程需要长时间占用锁执行同步块，这时候就不适合使用自旋锁了，因为自旋锁在获取锁前一直都是占用 cpu 做无用功，同时有大量线程在竞争一个锁，会导致获取锁的时间很长，线程自旋的消耗大于线程阻塞挂起操作的消耗，其它需要 cpu 的线程又不能获取到 cpu，造成 cpu 的浪费。所以这种情况下我们要关闭自旋锁。

<!--more-->

# 0x01 Naive Spinlock

```c++
#include <atomic>
#include <thread>

class SpinLock {
private:
    std::atomic<bool> lockFlag;

public:
    SpinLock() : lockFlag(false) {}

    void lock() {
        bool expected = false;
        // 使用atomic的compare_exchange_strong来设置锁的状态
        // 如果lockFlag是false（即未被锁定），则设置为true（锁定状态）
        // 如果lockFlag已经是true，则循环等待
        // compare_exchange_strong成功会将新值赋值给lockFlag，失败会将新值赋值给旧值(expected)
        while (!lockFlag.compare_exchange_strong(expected, true, std::memory_order_acquire)) {
            expected = false;
            // 在这里可以添加一些策略来减少CPU的忙等，例如让出CPU时间片
            std::this_thread::yield();
        }
    }

    void unlock() {
        // 设置锁的状态为false，表示锁已经被释放
        lockFlag.store(false, std::memory_order_release);
    }
};
```

这种简单的自旋锁有一个问题：**无法保证多线程竞争的公平性**。当多个线程想要获取锁时，谁最先将`lockFlag`设为`false`谁就能最先获得锁，这可能会造成某些线程一直都未获取到锁造成`线程饥饿`。就像我们下课后蜂拥的跑向食堂，下班后蜂拥地挤向地铁不排队。



# 0x02 Ticket Lock

就像票据队列管理系统一样。面包店或者服务机构(例如银行)都会使用这种方式来为每个先到达的顾客记录其到达的顺序，而不用每次都进行排队。通常，这种地点都会有一个分配器(叫号器，挂号器等等都行)，先到的人需要在这个机器上取出自己现在排队的号码，这个号码是按照自增的顺序进行的，旁边还会有一个标牌显示的是正在服务的标志，这通常是代表目前正在服务的队列号，当前的号码完成服务后，标志牌会显示下一个号码可以去服务了。ticket lock是基于先进先出(FIFO) 队列的机制。它增加了锁的公平性

```c++
#include <atomic>
#include <thread>

class TicketLock {
private:
    // 取票机器，递增序号
    std::atomic<unsigned> ticket;
    // 叫号机器
    std::atomic<unsigned> nowServing;

public:
    TicketLock() : ticket(0), nowServing(0) {}

    void lock() {
        unsigned myTicket = ticket.fetch_add(1, std::memory_order_relaxed);
        while (myTicket != nowServing.load(std::memory_order_relaxed)) {
            // 自旋等待，直到myTicket等于nowServing
            std::this_thread::yield();
        }
    }

    void unlock() {
        unsigned newServing = nowServing.load(std::memory_order_relaxed) + 1;
        nowServing.store(newServing, std::memory_order_relaxed);
    }
};

```

TicketLock 虽然解决了公平性的问题，但是多处理器系统上，每个进程/线程占用的处理器都在读写同一个变量`ticket`取号 ，每次读写操作都必须在多个处理器缓存之间进行缓存同步，这会导致繁重的系统总线和内存的流量，大大降低系统整体的性能。



# 0x03 CLH Lock

上面说到Ticket Lock 是基于队列的，那么 CLH Lock 就是基于链表设计的，CLH的发明人是：Craig，Landin and Hagersten，用它们各自的字母开头命名。CLH 是一种基于链表的可扩展，高性能，公平的自旋锁，申请线程只能在本地变量上自旋，它会不断轮询前驱的状态，如果发现前驱释放了锁就结束自旋。

```c++
#include <atomic>
#include <thread>

class CLHLock {
private:
    struct Node {
        std::atomic<bool> locked;
    };

    std::atomic<Node*> tail;
    // thread_local表示每个线程都有一个这样的副本
    thread_local static Node* myNode;
    thread_local static Node* myPred;

public:
    CLHLock() {
        tail.store(nullptr);
    }

    void lock() {
        myNode = new Node{true};
        myNode->locked.store(true, std::memory_order_relaxed);
        myPred = tail.exchange(myNode);
        if (myPred != null) {
            // 前驱节点不为null表示当锁被其他线程占用，通过不断轮询判断前驱节点的锁标志位等待前驱节点释放锁
            while (myPred->locked.load(std::memory_order_acquire)) {
                // 自旋等待
                std::this_thread::yield();
            }
        }
        // 如果不存在前驱节点，表示该锁没有被其他线程占用，则当前线程获得锁

    }

    void unlock() {
        auto tmp = myNode;
        // 如果当前myNode不是尾节点，将当前的locked状态设置为0
        if (!tail.compare_exchange_strong(tmp, nullptr, std::memory_order_acquire)) {
            myNode->locked.store(false, std::memory_order_release);
        }
        delete myNode;
        myNode = nullptr;
    }
};

thread_local CLHLock::Node* CLHLock::myNode = nullptr;
thread_local CLHLock::Node* CLHLock::myPred = nullptr;

```



# 0x04 MCS Lock

MCS Spinlock 是一种基于链表的可扩展、高性能、公平的自旋锁，申请线程只在本地变量上自旋，直接前驱负责通知其结束自旋，从而极大地减少了不必要的处理器缓存同步的次数，降低了总线和内存的开销。MCS 来自于其发明人名字的首字母：John Mellor-Crummey 和 Michael Scott。

```c++
#include <atomic>
#include <thread>
#include <iostream>

typedef struct mcs_node {
    mcs_node *next;
    bool locked;
} mcs_node;

class mcs_lock {
public:
  mcs_lock():tail(nullptr) {}
  void lock() {
      std::cout << "lock\n";
      /* Atomically place ourselves at the end of the queue: */
      qnode.locked = true;
      qnode.next = nullptr;
      const auto predecessor = tail.exchange(&qnode);

      /**
       * If tail was nullptr, predecesor is nullptr, thus nobody has been waiting,
       * and we've acquired the lock.
       * Otherwise, we need to place ourselves in the queue, and spin:
       */
      if (predecessor != nullptr) {
          /**
           * If the lock is taken, there's two cases:
           * 1. Either there is nobody waiting on the lock, and *tail == this.qnode (more
           *    on this later)
           * 2. One or more CPUs are waiting on the lock, and *tail is the tail of the queue
           * Either way, we mark the lock is taken:
           */

          /* Link ourselves to the tail of the queue: */
          predecessor->next = &qnode;

          /* Now we can spin on a local variable: */
          while (qnode.locked) {
          }
      }
  }

   void unlock() {
      /* We are holding the lock, therefore qnode.next is our successor: */
      auto *successor = qnode.next;

      if (successor == nullptr) {
          auto *expcted = &qnode;
          if (tail.compare_exchange_strong(expcted, nullptr, std::memory_order_acquire)) {
              /* No CPUs were waiting for the lock, set it to nullptr: */
              return;
          }
      }

      /**
       * We could not set our successor to nullptr, therefore qnode.next is out of sync with tail,
       * therefore another CPU is in the middle of `enter`, prior to linking themselves in the queue.
       * We wait for that to happen:
       */
      while (successor == nullptr) {
        // 这里非常关键。可能出现的情况是在赋值的那一刻后继节点为空，但是立刻有新的节点进入了，这样就会导致死循环
        successor = qnode.next;
      }

      /* The other CPU has linked themselves, all we need to do is wake it up as the next-in-line: */
      successor->locked = false;
  }

private:
    std::atomic<struct mcs_node *> tail;
    thread_local static mcs_node qnode;
};
 thread_local mcs_node mcs_lock::qnode;

int main() {
  mcs_lock lock;
  int counter = 0;
  std::cout << "hello" << "\n";
  #pragma omp parallel for default(none) shared(lock, counter)
  for (int i = 0; i < 100000; i++) {
    lock.lock();
    ++counter;
    lock.unlock();
  }

  std::cout << "counter=" << counter << "\n";

  return 0;
}
```

- 都是基于链表，不同的是CLH Lock是基于隐式链表，没有真正的后续节点属性，MCS Lock是显示链表，有一个指向后续节点的属性。
- 将获取锁的线程状态借助节点(node)保存,每个线程都有一份独立的节点，这样就解决了Ticket Lock多处理器缓存同步的问题。
