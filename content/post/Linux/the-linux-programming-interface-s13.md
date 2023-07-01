---
title: "Linux/Unix系统编程手册-笔记13.文件系统"
date: 2020-08-11T09:49:00+08:00
tags: ["Linux"]
categories: ["Linux系统编程手册阅读"]
---

## 设备文件

一个设备文件对应一个设备，这个设备可以是真实的设备(鼠标、键盘、磁盘等)，也可以是虚拟设备。 设备分两种：
- 字符设备： 以字符为单位处理数据的设备，终端、键盘都是此类型。
- 块设备： 一次处理一个数据块，常见的块设备就是磁盘了。

`ll /dev`看一下`/dev`目录下

```
...
crw——- 1        root root   10, 183     8月 10 10:07 hwrng
crw——- 1        root root   89, 0       8月 10 10:07 i2c-0
crw——- 1        root root   89, 1       8月 10 10:07 i2c-1
crw——- 1        root root   89, 2       8月 10 10:07 i2c-2
crw——- 1        root root   89, 3       8月 10 10:07 i2c-3
crw——- 1        root root   89, 4       8月 10 10:07 i2c-4
crw——- 1        root root   89, 5       8月 10 10:07 i2c-5
crw——- 1        root root   89, 6       8月 10 10:07 i2c-6

...

brw-rw—- 1      root disk   7, 0        8月 10 10:07 loop0
brw-rw—- 1      root disk   7, 1        8月 10 10:07 loop1
brw-rw—- 1      root disk   7, 2        8月 10 10:07 loop2
brw-rw—- 1      root disk   7, 3        8月 10 10:07 loop3
brw-rw—- 1      root disk   7, 4        8月 10 10:07 loop4
brw-rw—- 1      root disk   7, 5        8月 10 10:07 loop5
brw-rw—- 1      root disk   7, 6        8月 10 10:07 loop6
...
```

第一列以`c`开头的为字符设备，以`b`开头的为块设备。设备文件在修改日期之前由逗号隔开的分别是设备主ID和次ID。设备主ID用来标识一类设备，kernel据次来使用合适的驱动打开文件。次设备ID是用来在设备打开时，传给驱动，驱动以此来区分和控制特定设备。

* 关于Linux 设备驱动这本书《Linux Device Drivers》好像不错，待详细了解。

## 文件系统

文件系统是有组织的常规文件和目录的集合。


{{< figure src="/img/the-linux-programming-interface-s13/Layout_of_disk_partitions_and_a_file_system.png"  class="center" title="磁盘分区和文件系统" width="600" >}}

如图为磁盘分区和文件系统的关系。  
文件系统包括以下几个部分:
- Boot block: 文件系统的第一个block, 启动操作系统用的，操作系统只会用到一个文件系统的boot block，其他分区的文件系统大多数不会用到。
- Superblock: 单个block, 紧跟boot block, 包括文件系统的参数信息：
    * i-node table大小
    * 逻辑块大小
    * 此文件系统大（以逻辑块为单位）
- I-node table: 文件系统中的每一个文件和文件夹都在i-node表中有一个唯一的条目，里面记录了文件的各种信息。
- Data blocks: 文件系统中的大多数空间， 也是保存文件和目录的地方。


##  I-nodes

每个文件的i-node信息在i-node表中按顺序存储，并以i-number区分，使用`ls -li`可以查看文件的i-number,第一列就是。  
i-node 中维护的信息包括:

- 文件类型 (比如常规文件、目录、或链接、字符设备等)
- 所有者(owner)
- 组(Group)
- 访问权限
- 三种时间戳：最后访问时间(`ls -lu`)、最后修改时间(`ls -l`)、最后属性修改时间(`ls -lc`，即最后i-node修改时间)，大多数UNIX没有文件创建时间。
- 硬链接到文件的数量
- 文件大小(bytes)
- 文件实际使用的blocks数量 
- 指向文件数据块的指针

**数据块指针**

每个i-node包含15个指针，前12个直接指向文件前12个块在文件系统中的位置，然后第13个是一个指向指针块的指针，指针块中的指针数量取决于块大小，每个指针占用4个字节，因此指针的数量可能在256（块容量为1024字节）～1024（块容量为4096字节）之间。这样就考虑了大型文件的情况。第14个指针是双重间接指针--指向指针块，其中的指针又指向一个指针块。此块中的指针才最终只想文件的数据块，第15个指针为一个三重指针。结构如下图：


{{< figure src="/img/the-linux-programming-interface-s13/structure_of_file_blocks_for_a_file_in_an_ext2_file_system.png"  class="center" title="数据块指针" width="600" >}}


对于大小为4096字节的块而言，只计算三重指针，单个文件大小可达1024×1024×1024×4096字节（即4TB）。  
该设计的另一优点在于文件可以有黑洞。文件系统只需要将i节点和间接指针块中的相应指针打上标记（值0），表明这些指针并未只想实际的磁盘块即可。


## 获得文件系统有关的信息 ： statvfs()

```cpp
#include <sys/statvfs.h>

int statvfs(const char* pathname, struct statvfs * statvfsbuf);
int fstatvfs(int fd, stuct statvfs *statvfsbuf);
```
两者区别仅在于其识别文件系统的方式。返回的 statvfs数据结构如下：

```cpp
struct statvfs
  {
    unsigned long int f_bsize;          // 文件系统 block大小（bytes）
    unsigned long int f_frsize;         // 基本文件系统 block大小（bytes）
    fsblkcnt_t f_blocks;                // 文件系统中block总数
    fsblkcnt_t f_bfree;                 // 空闲block总数
    fsblkcnt_t f_bavail;                // 非特权进程可用的空闲block数
    fsfilcnt_t f_files;                 // i-nodes数量
    fsfilcnt_t f_ffree;                 // 空闲 i-nodes数量
    fsfilcnt_t f_favail;                // 非特权进程可用的i-nodes数量
    unsigned long int f_fsid;           // file-system ID
    unsigned long int f_flag;           // mount flags
    unsigned long int f_namemax;        // 此文件系统上的文件名最大长度
  }
```

- 绝大多数Linux文件系统上f_bsize和f_frsize是相同的，但有些文件系统支持块片段的概念，，在无需使用完整数据块的情况下，可在文件尾部分配较小的存储单元，从而避免因分配完整块而导致的空间浪费。在此类文件系统上f_frsize和f_bsize分别为块片段和整个块的大小。
- 一般文件系统都会给root预留一部分空间，ext3、ext4一般默认预留5%

## Exercises

1. 个人认为区别不大，文件删除只删除i-node信息，i-node信息直接通过i-number可以随机访问，所以随机删除和按顺序删除的耗时应该区别不大。