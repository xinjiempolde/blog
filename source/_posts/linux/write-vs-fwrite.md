---
title: write_vs_fwrite
date: 2023-12-20 16:09:57
tags:
    - linux
categories:
    - linux
---
> 参考[C++中write和fwrite哪个效率更高？ - 知乎 (zhihu.com)](https://www.zhihu.com/question/546404062/answer/2605567064?utm_psn=1720524853226872832)
>
> [Linux 文件IO学习之open函数深入了解_linux open函数-CSDN博客](https://blog.csdn.net/m0_61167558/article/details/127561314)

# 0x00 Background

`write`和`fwrite`都用于文件写入，知其然更要知其所以然，既然有这么多种设计，就一定各有各的考量，那么它们之间有什么不同呢？

<!--more-->

## write

如果不添加`O_DIRECT`标志，只以普通的方式进行读写(**.O_RDONLY,O_WRONLY,O_RDWR**)，那默认内核是会把文件的这个页面读进来缓存在内核里的，也即所谓的`page cache`。随后再发起新的write syscall写相同的页面时，只要写在page cache里就可以结束。

![write只需要写入到page cache就可以返回](https://img.singhe.art/v2-03b3ccaf25fe5d3564f7a154b92bff45_1440w.webp)

内核的这个`page cache`好处有很多，比如你的程序对IO还没到需要自己做用户态的读写缓存，那内核的这个机制就帮你省去了很多工作，毕竟page cache是在内存里的，而且可以拿来做read hit，相比于每次read/write都要访问磁盘，带来的性能优势还是很不错的，算是惠及大部分普通程序。

如果添加`O_DIRECT`标志，那么后续对这个文件的所有read/write syscall都会bypass掉内核的page cache，也就是read/write直接发起disk io，数据将不会在内核中进行缓存。

![O_DIRECT将绕过page cache](https://img.singhe.art/v2-a1ecc102d248be9e67f989a048299cc9_1440w.webp)





## fwrite

fwrite是用户态的glibc库，相当于把write的系统调用封装了一下，关键一点在于，他在用户态又多加了一个buffer，，只有当你的fwrite写入量够多或者你主动fflush才会真的发起一个write syscall。所以fwrite的好处是对于小量的写，减少syscall的次数，毕竟如果你每写一个字节都要发起一个syscall，然后特权级切换到内核，这就太过耗时了。

![fwrite在用户态也有缓存](https://img.singhe.art/v2-ea2dc2b18ba738a06de7f2a0a7824e38_1440w.webp)



## 对比

buffer，或者缓存虽然有好处，但也有适用条件以及额外的开销：

- 如果你的程序的文件读写几乎没有locality或者什么热点，加缓存不会带来cache hit方面的性能提升
- 反而如果你写入的数据量很大，那用fwrite时，会发生你程序的buffer到glibc  buffer一次拷贝，glibc buffer发起write syscall到page cache一次拷贝；而如果是普通write（或O_DIRECT的write），只会发生你程序的buffer到page cache（disk的buffer cache）一次拷贝。

所以从性能的角度，fwrite并不适合大量写的场景。然而光从这个角度并不能看出O_DIRECT的有无带来什么作用。

O_DIRECT适用于，数据读写性能、一致性、locality、写回时机等等对你的程序已经重要到全都要你自己管理，这时内核自带的page cache那种粗粒度、不太可控的设施已经不能满足你的需求了。

没错，最典型的就是数据库应用。

对数据库而言，依赖于page cache会带来非常多问题，比如：

- writeback.，也即写回磁盘同步的时机不可控。page cache可能在任何时候写回，包括你的事务做到一半，进程遭到调度，内核擅自把部分page cache上的内容写回磁盘，造成预期外的数据不一致。
- evict策略的不可控。即便你有一套自己的热度评价机制，哪个page是热点也是内核说了算，你很难干预内核不要evict掉page cache上的哪个页面，这会带来不可控的性能抖动。
- 小量的写（数据内容更新）也要发起系统调用，带来不必要的特权级切换开销。



# 0x01 write和fwrite系统调用次数对比

通过逐字节的写入32MB的数据到磁盘上，观察`write`和`fwrite`的系统调用的次数，来说明少量多次数据写入对`write`和`fwrite`带来的性能影响。

`write.c`代码：

```c++
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>
int main() {
  int fd = open("write_byte.data", O_WRONLY | O_CREAT | O_TRUNC, 0644);
  // Check if the file is successfully opened.
  if (fd == -1) {
      perror("cannot open the file");
      exit(-1);
  }
  constexpr int data_size = 1 << 25;  // 32MB
  char *data = new char[data_size+1];
  for (long i = 0; i < data_size; ++i) {
    write(fd, data + i, 1);
  }
  delete []data;
}
```

这里的`write`代码并未添加`O_DIRECT`标志，也就意味着`write`在内核中仍然有page cache缓存。

`fwrite.c`代码：

```c
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>

int main() {
  const char *filename = "fwrite_byte.data";
  FILE *file = fopen(filename, "wb");
  // Check if the file is successfully opened.
  if (file == NULL) {
    printf("cannot open file %s", filename);
  }
  constexpr uint64_t data_size = 1LL << 25;  // 32MB
  printf("%lu\n", data_size);
  char *data = new char[data_size+1];
  for (uint64_t i = 0; i < data_size; ++i) {
    fwrite(data + i, 1, 1, file);
  }
  delete []data;
}

```

![fwrite系统调用次数](https://img.singhe.art/image-20231220153731614.png)

![write系统调用次数](https://img.singhe.art/image-20231220155510373.png)

通过`strace`跟踪系统调用，我们可以发现对于32MB数据逐字节写入，`fwrite`共调用了8192次`write`系统调用，而`write`却用了$32 \times 1024 \times 1024 = 33554432$次系统调用。也就是说`write`不管每次写入数据的大小，都会进行一次系统调用。而`fwrite`虽然是逐个字节的写入，但是它会在用户层积累，直到有4KB大小的数据后才会调用一次系统调用，节省了许多时间。通过`google c++ benchmark`跟踪运行时间也可以看出区别，write花了42s，而fwrie却只花了0.38s。

![write和fwrite时间对比](https://img.singhe.art/image-20231220155307044.png)

## 0x02 O_DIRECT对write的影响

上文已说明，如果添加`O_DIRECT`标志，`write`将会绕过内核的page cache，而直接进行写入，那么写入延迟将会更大。从下面的结果也可以看出，不添加`O_DIRECT`逐字节写入32MB数据需要40s，而添加`O_DIRECT`标志则需要201s。

![添加O_DIRECT](https://img.singhe.art/image-20231220160212174.png)



# 0x03 按照4K写入32MB数据

如果`write`和`fwrite`都按照4096字节写入32MB，那么它们系统调用的次数都是相同的。不同的是，`fwrite`会在应用层也会添加一层缓存，那么就会多一层数据拷贝，在这种场景下，`write`将会比`fwrite`更快。

![按照4096字节写入](https://img.singhe.art/image-20231220160806126.png)
