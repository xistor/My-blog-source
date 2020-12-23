---
title: "Linux/Unix系统编程手册-笔记32. System V 共享内存"
date: 2020-12-23T18:30:00+08:00
tags: ["Linux"]
categories: ["Linux系统编程手册阅读"]
---

共享内存的读写比较简单，而且速度快，但需要访问同步手段，一般和信号量一起使用。  

使用System V 共享内存步骤,一般分以下几个步骤：

- 调用`shmget()`创建一块新的共享内存， 或者获取当前存在的共享内存标识符。
- 使用`shmat()`来attach一块共享内存，这步将使共享内存片段作为调用进程的虚拟内存的一部分。
- 然后就可以和一块普通内存一样使用共享内存了，共享内存的起始地址是`shmat()`的返回值。
- 调用`shmdt()` detach共享内存，这步可选，进程退出时会自动执行。
- 调用`shmctl()` 删除共享内存。


## 共享内存在虚拟内存中的位置

使用 `/proc/PID/maps` 文件可以查看程序的内存布局。

![内存布局](/img/the-linux-programming-interface-s32/share_mem_location.png)

共享内存被attach到虚拟内存中的未分配区域，也就是堆的生长方向和栈的生长方向之间。

`shmat()`的函数原型如下，参数`shmaddr`可以指定attach的地址，但是最好设为null,由内核attach到合适的地址。

```c
#include <sys/types.h> /* For portability */
#include <sys/shm.h>

void *shmat(int shmid, const void *shmaddr, int shmflg);

// Returns address at which shared memory is attached on success, or (void *) –1 on error
```

## 在共享内存中保存指针

共享内存在不同进程中attach的内存地址不同，所以当在共享内存中保存一个指向共享内存中另一地址的指针时，需要使用相对地址，而不是绝对地址。

![共享内存中保存指针](/img/the-linux-programming-interface-s32/share_mem_location.png)

如图baseaddr为共享内存起始地址，target是共享内存中的一个绝对地址，比如0x7fe950304000。如果我们想保存它那像`*p = target;`这样是不对的，应该为`*p = (target - baseaddr);` ,解引用指针时 `targrt = baseaddr + *p`。

这里的问题在于，不同的进程attach同一块共享内存后，其虚拟内存地址是不同的，所以如果想把一个进程中指向共享内存的指针给attach同一块共享内存的另一个进程用，需要像前面那样使用相对地址。  
