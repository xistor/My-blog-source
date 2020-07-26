---
title: "Linux/Unix系统编程手册-笔记6.内存分配"
date: 2020-07-25T10:53:00+08:00
tags: ["Linux"]
categories: ["Linux系统编程手册阅读"]
---

### 在堆上分配内存

系统提供了两个改变堆大小的系统调用：brk()和sbrk()

```cpp
    int brk(void *end_data_segment); // return 0 on success or -1 on error

    void *sbrk(intptr_t increment);     
        // return previous progrom break on success, or (void *) -1 on error
```

brk()会将program break设置为参数end_data_segment所指定的位置（实际会四舍五入到下一个内存页的边界处）。  
sbrk()会将program break 在原有地址上增加increment 大小。


* program break指进程数据段的末尾(end of process's data segment)  

C中的malloc()和free()函数在堆上分配和释放内存，一般情况下free()并不降低program break的位置。而是将这块内存添加到空闲内存列表中，供后续的malloc()函数循环使用。仅当堆顶空闲内存“足够”大的时候，free()函数的glibc()实现会调用sbrk()来降低program break的地址。这个大小取决于malloc()函数包行为的控制参数（默认为128K） 详细见[mallopt](https://man7.org/linux/man-pages/man3/mallopt.3.html),其中M_TRIM_THRESHOLD参数即此项。  

### malloc()和free()的实现

malloc的实现很简单，它会先扫描之前由free()所释放的空闲内存块列表，以找到一块大小大于或等于要求的空闲内存。基于具体实现扫描策略有所不同，对于大于要求的空闲内存，会被分割。如果空闲列表中找不到足够大的内存块。malloc会调用sbrk()分配一块。而且为了减少sbrk()调用的次数，请求分配的内存并不是严格按照所需的内存，而是以更大幅度（虚拟内存页大小的数倍）来增加program break， 并将超出部分置于空闲内存列表中。
* 每个进程一个空闲内存列表。  

malloc()在分配内存时会额外分配几个字节来记录这块内存大小，该整数位于内存块的起始处。 

![malloc返回的地址](/img/the-linux-programming-interface-s6/mem_block_returned_by_malloc.png)  

当内存块置于空闲内存列表时，free()会使用内存块本身的空间来存放链表指针，将自身添加到列表中
![空闲列表中的内存块](/img/the-linux-programming-interface-s6/a_block_on_the_free_list.png)


随着对内存的不断释放和重新分配，空闲列表中的空闲内存会和已分配的内存混杂在一起，如下图:
![包含有已分配内存和空闲内存列表的堆](/img/the-linux-programming-interface-s6/heap_containing_allocated_blocks_and_a_free_kist.png)


书中的图展示的空闲列表为显式空闲链表，除此之外还有几种实现：
- 隐式空闲链表：如下图，一个块由一个字的头部，有效载荷，以及可能的一些额外的填充，之所以称其为隐式空闲链表，是因为空闲块是通过头部中的大小字段隐含的链接着的。
![隐式空闲列表](/img/the-linux-programming-interface-s6/a_block_of_Implicit_free_list.png)
在隐式空闲链表合并空闲块时，释放当前块后，想要合并前一个空闲块，会遇到问题，因为按照上图的结构，我们无法直接知道前一个块的大小，也就无法在常数时间内完成合并，唯一的选择是遍历整个链表。  
使用下图结构可以解决这个问题，如图，每个块的结尾添加了一个脚部，脚部就是头部的一个副本。每个块的脚部会与下一个块的头部相邻，所以下一个块可以很容易的访问到前一个块的脚部，以知道其前一个块的大小以及是否空闲。
![带边界标记的堆块的格式](/img/the-linux-programming-interface-s6/a_block_with_footer.png)
- 分离的空闲列表：维护多个空闲链表，其中每个链表中的块有大致的相等的大小，不同的分离存储的方法的主要区别就是如何定义大小类似的等价类（大小类），何时进行合并，何时向操作系统请求额外的堆内存，是否允许分割等。主要有：
    * 简单分离存储  
    每个大小类的空闲链表包含大小相等的块，每个块的大小就是这个大小类中最大元素的大小。比如某个大小类定义为{17～32}，那么这个类的空闲链表全由大小为32的块组成。此方法不分割、不合并。
    * 分离适配  
    分配器维护着一个空闲链表的数组。每个空闲链表按照一定关系和一个大小类相关连。为了分配一个块，首先确定大小类，并对适当的空闲链表做首次适配，查找一个合适的块，如果找到了，就分割它，并将剩下部分插入到适当的空闲链表中。如果找不到合适的块，那么就搜索下一个更大的大小类空闲链表。如果空闲链表没有合适的块，那么就向操作系统请求额外的堆内存，从中分割出合适块，将剩下的放置到合适的大小类中。要释放一个块，我们执行合并，并将结果放置到相应的空闲链表中。
    * 伙伴系统
    是分离适配的一个特例。每个大小类都是2的幂。基本思路是假设一个堆的大小为2^m个字，也就是说一开始只有一个2^m个字的空闲块。为了分配一个大小为2^k(0<=k<=m>)的块，我们找到第一个可用的、大小为2^j的块(k<=j<=m),如果j=k,那么我们就完成了。否则，我们递归的二分割这个块，把每个剩下的半块（也叫做伙伴）放置到相应的空闲链表中，直到j=k。要释放一个大小为2^k的块，我们继续合并空闲的伙伴。当遇到一个已分配的伙伴的时候，我们就停止合并。


### 在堆上分配内存的其他方法

```cpp
#include <stdlib.h>

void* calloc(size_t numitems, size_t size);

void* realloc(void *ptr, size_t size);
```

calloc()与malloc()的区别是calloc()会将已分配的内存初始化为0。  
realloc()函数来调整一块内存的大小，此块内存应是之前malloc包中函数分配的。



 ### Exercise
1. 修改free_and_sbrk.c如下，
```cpp
#define _BSD_SOURCE
#include "tlpi_hdr.h"

#define MAX_ALLOCS 1000000

int
main(int argc, char *argv[])
{
    char *ptr[MAX_ALLOCS];
    int blockSize, numAllocs, j;

    printf("\n");

    if (argc < 3 || strcmp(argv[1], "--help") == 0)
        usageErr("%s num-allocs block-size [step [min [max]]]\n", argv[0]);

    numAllocs = getInt(argv[1], GN_GT_0, "num-allocs");
    if (numAllocs > MAX_ALLOCS)
        cmdLineErr("num-allocs > %d\n", MAX_ALLOCS);

    blockSize = getInt(argv[2], GN_GT_0 | GN_ANY_BASE, "block-size");

    

    printf("Initial program break:          %10p\n", sbrk(0));

    printf("Allocating %d*%d bytes\n", numAllocs, blockSize);
    for (j = 0; j < numAllocs; j++) {
        ptr[j] = malloc(blockSize);
        if (ptr[j] == NULL)
            errExit("malloc");
        printf("Program break is now:           %10p\n", sbrk(0));
    }

    for (j = 0; j < numAllocs / 2; j++)
        free(ptr[j]);

    printf("After free(), program break is: %10p\n", sbrk(0));

    for (j = 0; j < numAllocs / 2; j++) {
        ptr[j] = malloc(blockSize);
        if (ptr[j] == NULL)
            errExit("malloc");
    }

    printf("re malloc, program break is: %10p\n", sbrk(0));

    exit(EXIT_SUCCESS);
}

```

运行结果：

```
$./malloc_and_sbrk 50 10240

Initial program break:          0x56289c0f8000
Allocating 50*10240 bytes
Program break is now:           0x56289c0f8000
Program break is now:           0x56289c0f8000
Program break is now:           0x56289c0f8000
Program break is now:           0x56289c0f8000
Program break is now:           0x56289c0f8000
Program break is now:           0x56289c0f8000
Program break is now:           0x56289c0f8000
Program break is now:           0x56289c0f8000
Program break is now:           0x56289c0f8000
Program break is now:           0x56289c0f8000
Program break is now:           0x56289c0f8000
Program break is now:           0x56289c0f8000
Program break is now:           0x56289c0f8000
Program break is now:           0x56289c11b000
Program break is now:           0x56289c11b000
Program break is now:           0x56289c11b000
Program break is now:           0x56289c11b000
Program break is now:           0x56289c11b000
Program break is now:           0x56289c11b000
Program break is now:           0x56289c11b000
Program break is now:           0x56289c11b000
Program break is now:           0x56289c11b000
Program break is now:           0x56289c11b000
Program break is now:           0x56289c11b000
Program break is now:           0x56289c11b000
Program break is now:           0x56289c11b000
Program break is now:           0x56289c13c000
Program break is now:           0x56289c13c000
Program break is now:           0x56289c13c000
Program break is now:           0x56289c13c000
Program break is now:           0x56289c13c000
Program break is now:           0x56289c13c000
Program break is now:           0x56289c13c000
Program break is now:           0x56289c13c000
Program break is now:           0x56289c13c000
Program break is now:           0x56289c13c000
Program break is now:           0x56289c13c000
Program break is now:           0x56289c13c000
Program break is now:           0x56289c13c000
Program break is now:           0x56289c13c000
Program break is now:           0x56289c15f000
Program break is now:           0x56289c15f000
Program break is now:           0x56289c15f000
Program break is now:           0x56289c15f000
Program break is now:           0x56289c15f000
Program break is now:           0x56289c15f000
Program break is now:           0x56289c15f000
Program break is now:           0x56289c15f000
Program break is now:           0x56289c15f000
Program break is now:           0x56289c15f000
After free(), program break is: 0x56289c15f000
re malloc, program break is: 0x56289c15f000

```

符合预期，malloc()不会每次申请内存都去调用sbrk()，而且free()后的内存会再次保存在空闲内存链表中，在malloc()再次申请时返回给调用者。