---
title: "Linux/Unix系统编程手册-笔记33. 内存映射"
date: 2020-12-26T18:47:00+08:00
tags: ["Linux"]
categories: ["Linux系统编程手册阅读"]
---


## 概念

mmap()内存映射有两种：
- 文件映射： 将一个文件的一部分直接映射到调用进程的虚拟内存中，映射的分页会按需要从文件中自动加载。
- 匿名映射： 没有对应的映射文件，映射的分页被初始化为0。


内存映射的分页可能多个进程共享，主要在以下以下两个情形下会出现：
- 当多个进程映射一个文件的同一部分，他们共享物理内存中的同一分页
- fork()后的子进程继承父进程的映射。

当多个进程共享同一分页的时，不同进程对之间是否能看到其他进程对共享内存分页的修改取决于是私有映射还是共享映射：
- 私有映射(MAP_PRIVATE) : 对映射内存的修改对其他进程不可见。对于文件映射，对内存的修改也不会反应到底层文件。一般以copoy-on-write实现。
- 共享映射(MAP_SHARED) : 对映射内存的修改对其他进程可见，对于文件映射，内存修改也会反映到底层文件上。


内存映射类型和是否共享这两个属性组合会产生四种内存映射类型：
- 私有文件映射： 映射内存的内容以文件内容初始化。进程的代码段和初始化过的数据段就是以这种方式映射到虚拟内存的。
- 私有匿名映射： 每次调用mmap()创建的私有匿名映射会和其他私有匿名映射区分开，即使子进程继承的私有内存映射，可以访问，但copy-on-write会保证它看不到其他进程的修改。 私有匿名映射的主要用处就是给一个进程分配0填充的新内存。
- 共享文件映射： 所有映射文件同一部分的进程共享同一内存分页。对内存内容的修改会反应到文件上。此类有两种用处，一是提供了一种可选的读写文件的方式，二是进程间可以以此实现IPC。
- 共享匿名内存映射：fork()之后，对共享内存映射的内存分页对其他进程可见，和共享内存类似，但只能用在相关进程之间。

## API

```c
#include <sys/mman.h>
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
// Returns starting address of mapping on success, or MAP_FAILED on error
```
- `addr`: 映射到虚拟内存的位置，若为null,内核会选取合适的地址。
- `length`: 指定映射内存的大小(bytes)。
-  `port`: 映射处的保护mask，不同进程对同一内存映射区域可能有不同的保护。
    |值|描述|
    |--|----|
    |PORT_NONE|不可访问|
    |PORT_READ|内容可读|
    |PORT_WRITE|内容可写|
    |PORT_EXEC|内容可执行|
    port值需要和打开要映射的文件描述符时指定的权限相兼容。若用`O_WRONLY`打开一个文件，内存映射时只指定PORT_WRITE是不行的，因为硬件架构上就不允许内存页只写。若是私有映射，保护flag可以随意设置，因为不会牵扯到文件的读写。

- `flags`: 控制内存映射的选项，有`MAP_PRIVATE`和`MAP_SHARED`等。
- `fd`和`offset`:用于文件映射，用于指定映射的文件以及开始的偏移（需要页对齐）。在匿名映射时会忽略。

一个文件映射的栗子：

```c
// 打开一个文件
fd = open("a.txt" O_RDONLY);
if (fd == -1)
    errExit("open");
/* Obtain the size of the file and use it to specify the size of
 the mapping and the size of the buffer to be written */
if (fstat(fd, &sb) == -1)
    errExit("fstat");
// 将文件整个映射到虚拟内存中，这之后文件描述符就可以关闭了
addr = mmap(NULL, sb.st_size, PROT_READ, MAP_PRIVATE, fd, 0);

if (addr == MAP_FAILED)
    errExit("mmap");
// 将内存中的内容写到输出
if (write(STDOUT_FILENO, addr, sb.st_size) != sb.st_size)
    fatal("partial/failed write")
```

```c
#include <sys/mman.h>
int munmap(void *addr, size_t length);
// Returns 0 on success, or –1 on error
```

- `addr`: 内存映射起始地址，必须页对齐。
- `length`: 映射内存的大小。

可以unmap映射内存的一部分，也可以多块映射内存一起unmap。  


内存映射并不能改变文件大小，如需要增大文件或截断文件，在mmap()之前使用ftruncate()或write()。

但Linux提供了重新映射的系统调用`mremap()`（不可移植），可以改变映射大小。

```c
#define _GNU_SOURCE
#include <sys/mman.h>
void *mremap(void *old_address, size_t old_size, size_t new_size, int flags, ...);
// Returns starting address of remapped region on success, or MAP_FAILED on error
```

## MAP_NORESERVE 和交换空间过量使用

swap是虚拟内存在硬盘上的一部分，物理内存中暂时用不到的页将会被交换到swap中。一些应用创建了超大的内存映射（一般是私有匿名映射），但只使用了很小一块，如果内存为每一一个映射分配足够的交换空间(swap space), 会造成交换空间的浪费，所以就有了lazy swap reservation 机制, 此机制允许交换空间超额使用，只有应用访问的页才会载入。若RAM和swap耗尽会引来OOM killer杀进程释放内存。  

在`mmap()`时指定MAP_NORESERVE flag可以告诉内核不必检查剩余交换空间是否足够，不过这个flag只有在`/proc/sys/vm/overcommit_memory`为0时管用。其他情况：
```
/proc/sys/vm/overcommit_memory : 1 内核认为内存总是足够的，不检查。
/proc/sys/vm/overcommit_memory : 2 分配时总是检查是否足够，从不超额分配。可分配的虚拟内存地址空间的计算方式为：
CommitLimit = (total_RAM - total_huge_TLB) * overcommit_ratio / 100 + total_swap

total_RAM: 系统RAM的总量
total_huge_TLB: 为大页面预留的
overcommit_ratio:  /proc/sys/vm/overcommit_ratio 中的值，默认是50
total_swap: swap空间的大小
```

超额检测只有在以下两种情况下起作用：
- 私有可写映射（无论是文件还是匿名），swap空间使用的大小和每个进程的内存映射大小相同。
- 共享匿名映射，swap空间使用和内存映射大小相同。

给一个只读的私有映射保留swap是没有必要的，因为映射的内容是不能被更改的。共享文件映射也没必要使用swap,因为修改会被反应到映射的文件上，这个文件就和swap的作用相同。

##  MAP_FIXED

 若`mmap()`不指定MAP_FIXED，参数addr仅作为参考值，内核会在其周围选一个页对齐的地址作为映射地址，当指定了MAP_FIXED后，addr就是映射地址了，此时addr必须是页对齐的。