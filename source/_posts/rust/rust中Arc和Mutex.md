---
title: rust中ARC和Mutex
date: 2024-07-28 20:04:21
tags:
    - rust
categories:
    - rust
---

简单地来说，Arc就是为了让变量能够再线程间共享，可以通过clone(并不会真正地深拷贝)的方式将数据的所有权给其他线程。但是如果数据要在多个线程之间修改的话，为了保证一致性，需要上锁，可供选择的方式有Mutex和RwLock等等。

RwLock.write()返回的是一个RwLockWriteGuard，我们可以解引用访问其内容，也可以通过解引用修改其内容。
