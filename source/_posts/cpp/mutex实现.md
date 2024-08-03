---
title: mutex实现
date: 2022-04-04 20:46:11
tags:
    - cpp
categories:
    - cpp
---

> 参考
>
> - [pthread包的mutex实现分析___pthread_mutex_s-CSDN博客](https://blog.csdn.net/tlxamulet/article/details/79047717)
> - [从C++mutex到futex - 别杀那头猪 - 博客园 (cnblogs.com)](https://www.cnblogs.com/HeyLUMouMou/p/17481385.html)

理论上讲，mutex可用初始值=1的信号量表示，只需一个整数表示其状态：0表示未占用，1表示占用。那么，mutex的资源占用就只是一个int型了？

<!--more-->

当然不是，我们可以看一下pthread包中mutex的定义：

```c++
typedef union
{
  struct __pthread_mutex_s
  {
    int __lock;
    unsigned int __count;
    int __owner;
    unsigned int __nusers;
    /* KIND must stay at this position in the structure to maintain
       binary compatibility.  */
    int __kind;
    int __spins;
    __pthread_list_t __list;
  } __data;
  ......
} pthread_mutex_t;
```

这是x86-64处理器下的mutex定义（32位处理器下的定义基本类似），占用32字节的空间。几个比较关键的成员定义如下：
`__lock mutex`：锁状态，0表示未占用，1表示占用
`__count` : 用于可重入锁，记录owner线程持有锁的次数
`__owner owner`: 线程ID
`__kind`: 记录mutex的类型，有以下几个取值：

```c++
　PTHREAD_MUTEX_TIMED_NP，这是缺省值，也就是普通锁。

　PTHREAD_MUTEX_RECURSIVE_NP，可重入锁，允许同一个线程对同一个锁成功获得多次，并通过多次unlock解锁。

　PTHREAD_MUTEX_ERRORCHECK_NP，检错锁，如果同一个线程重复请求同一个锁，则返回EDEADLK，否则与PTHREAD_MUTEX_TIMED_NP类型相同。

　PTHREAD_MUTEX_ADAPTIVE_NP，自适应锁，自旋锁与普通锁的混合。
```

pthread_mutex_init就是初始化上述的pthread_mutex_t内存结构。

pthread_mutex_lock处理了几种类型的mutex，细节请看https://github.com/lattera/glibc/blob/master/nptl/pthread_mutex_lock.c。先看普通锁，就是调用LLL_MUTEX_LOCK宏获得锁。LLL_MUTEX_LOCK宏定义我们稍后再看。：

```c++
if (__builtin_expect (type, PTHREAD_MUTEX_TIMED_NP)
      == PTHREAD_MUTEX_TIMED_NP)
    {
    simple:
      /* Normal mutex.  */
      LLL_MUTEX_LOCK (mutex);
      assert (mutex->__data.__owner == 0);
    }
```

这是可重入锁：

```c++
else if (__builtin_expect (type == PTHREAD_MUTEX_RECURSIVE_NP, 1))
    {
      /* Recursive mutex.  */

      /* Check whether we already hold the mutex.  */
      if (mutex->__data.__owner == id)
    {
      /* Just bump the counter.  */
      if (__builtin_expect (mutex->__data.__count + 1 == 0, 0))
        /* Overflow of the counter.  */
        return EAGAIN;

      ++mutex->__data.__count;

      return 0;
    }

      /* We have to get the mutex.  */
      LLL_MUTEX_LOCK (mutex);

      assert (mutex->__data.__owner == 0);
      mutex->__data.__count = 1;
    }
```

当发现owner就是自身，只是简单的自增__count成员即返回。否则，调用LLL_MUTEX_LOCK宏获得锁，若能成功获得，设置__count = 1，否则挂起。

这是检错锁，会侦测一个线程重复申请锁的情况，如遇到，报EDEADLK，从而避免这种最简单的死锁情形。若无死锁情形，goto simple语句会跳到普通锁的处理流程。：

```c++
else
    {
      assert (type == PTHREAD_MUTEX_ERRORCHECK_NP);
      /* Check whether we already hold the mutex.  */
      if (__builtin_expect (mutex->__data.__owner == id, 0))
    return EDEADLK;
      goto simple;
    }
```

这是自适应锁：

```c++
else if (__builtin_expect (type == PTHREAD_MUTEX_ADAPTIVE_NP, 1))
    {
      if (! __is_smp)
    goto simple;

      if (LLL_MUTEX_TRYLOCK (mutex) != 0)
    {
      int cnt = 0;
      int max_cnt = MIN (MAX_ADAPTIVE_COUNT,
                 mutex->__data.__spins * 2 + 10);
      do
        {
          if (cnt++ >= max_cnt)
          {
            LLL_MUTEX_LOCK (mutex);
            break;
          }

#ifdef BUSY_WAIT_NOP
          BUSY_WAIT_NOP;
#endif
        }
        while (LLL_MUTEX_TRYLOCK (mutex) != 0);

      mutex->__data.__spins += (cnt - mutex->__data.__spins) / 8;
    }
      assert (mutex->__data.__owner == 0);
    }
```

从代码看，这种锁分两个阶段。第一阶段是自旋锁（spin lock），忙等待一段时间后，若还不能获得锁，则转变成普通锁。
所谓“忙等待”，在x86处理器下是重复执行nop指令，nop是x86的小延迟函数：

```c++
/* Delay in spinlock loop.  */
#define BUSY_WAIT_NOP   asm ("rep; nop")
```

获取锁的核心是LLL_MUTEX_LOCK宏，我们来看其定义：

```c++
# define LLL_MUTEX_LOCK(mutex) \
  lll_lock ((mutex)->__data.__lock, PTHREAD_MUTEX_PSHARED (mutex))
```

PTHREAD_MUTEX_PSHARED宏表示该锁是进程锁还是线程锁，0表示线程锁，128表示进程锁，因mutex使用的核心算法既可适用于进程也可适用于线程。
从宏定义可知，获取锁的动作就是尝试修改锁的状态字段：__lock

lll_lock定义如下，我们只看线程锁部分的代码：

```c++
#define lll_lock(futex, private) \
  (void)                                      \
    ({ int ignore1, ignore2;                              \
       if (__builtin_constant_p (private) && (private) == LLL_PRIVATE)        \
     __asm __volatile ("cmpxchgl %1, %2\n\t"                   \
               "jnz _L_lock_%=\n\t"                   \
               ".subsection 1\n\t"                    \
               ".type _L_lock_%=,@function\n"             \
               "_L_lock_%=:\n"                    \
               "1:\tleal %2, %%ecx\n"                 \
               "2:\tcall __lll_lock_wait_private\n"           \
               "3:\tjmp 18f\n"                    \
               "4:\t.size _L_lock_%=, 4b-1b\n\t"              \
               ".previous\n"                      \
               LLL_STUB_UNWIND_INFO_3                 \
               "18:"                          \
               : "=a" (ignore1), "=c" (ignore2), "=m" (futex)     \
               : "0" (0), "1" (1), "m" (futex),           \
                 "i" (MULTIPLE_THREADS_OFFSET)            \
               : "memory");                       \
       else 
```

这是gcc里嵌入汇编的语法，其中：
: “=a” (ignore1), “=c” (ignore2), “=m” (futex)
是输出的寄存器列表，这里的意思表示ignore1使用EAX寄存器，ignore2使用ECX寄存器，futex使用的存储器。
另外，每个操作数会有一个Number与之对应。如果我们一共使用了n个操作数，那么输出操作里的第一个操作数就是0号，之后递增，所以，%0代表ignore1，%1代表ignore2，%2代表futex。

: “0” (0), “1” (1), “m” (futex)
是输入寄存器，”0”表示%0操作数，其值为0，亦即设置ignore1=0，同理ignore2=1

这样cmpxchgl %1, %2等价于：
cmpxchgl ignore2 futex
ignore2就是CAS里的新值N，N=1，futex是当前值V，但E又是什么呢？原来cmpxchgl使用了一个隐藏参数EAX代表E，前面已分析出来，EAX是ignore1，其值为0。则现在一切都清晰了，cmpxchgl检查futex（也就是__lock成员）是否为0（表示锁未占用），如是，赋值1（表示锁被占用），同时ZF标志位设置为1（ZF=1，JZ跳转，JNZ不跳转）；否则（说明锁已被占用），ZF标志位为0，JNZ跳转。

归纳起来就是：先使用CAS判断_lock是否占用，若未占用，直接返回。否则，通过__lll_lock_wait_private调用SYS_futex系统调用迫使线程进入沉睡。
上述过程就是所谓的FUTEX同步机制，CAS是用户态的指令，若无竞争，简单修改锁状态即返回，非常高效，只有发现竞争，才通过系统调用陷入内核态。所以，FUTEX是一种用户态和内核态混合的同步机制，它保证了低竞争情况下的锁获取效率。
