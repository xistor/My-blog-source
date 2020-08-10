---
title: "Linux/Unix系统编程手册-笔记12.文件I/O缓冲"
date: 2020-08-10T10:53:00+08:00
tags: ["Linux"]
categories: ["Linux系统编程手册阅读"]
---


首先看书中的一张图，就对文件I/O缓冲有个大概的了解了：

![文件I/O](/img/the-linux-programming-interface-s12/summary_of_IO_buffering.png)

I/O缓冲分两种
- stdio的输入输出函数内部所使用的缓冲，也就是图中的user memory部分，在执行标准库函数之后的某一刻，会调用fflush()后刷新缓冲去，此时才会调用系统调用wirte()等写入内核缓冲区。这部分缓冲的作用可以减少系统调用的次数。
- 内核缓冲区，系统调用的read(2)/write(2)和真实的磁盘读写之间也存在一层缓冲,以减少读写磁盘的耗时操作。

### 控制stdio I/O缓冲

```cpp
#include <stdio.h>
int setvbuf(FILE *stream, char *buf, int mode, size_t size);
                // Returns 0 on success, or nonzero on error
```

`buf`和`size`指定了file stream 使用的buffer和大小。
- `buf`非NULL, 则将`buf`指向的内存块给stream作为缓冲使用，`buf`所指向的内存必须从堆上分配。
- `buf`为NULL, stdio库将会自动分配一块内存作为缓冲(除非使用非缓冲I/O), `size`参数会被忽略(glibc)。

`mode` 参数可选值：

- IONBF 不使用缓冲，每次调用stdio输入输出函数将会马上调用write()或read()系统调用。
- IOLBF 行缓冲，输出时：遇到换行符就写入内核，读取时：每次读取一行到缓冲区。
- IOFBF 全缓冲，读、写数据(调用read, write)的大小与缓冲区大小一致，默认使用此模式。

**另外的控制函数**  

```cpp
#include <stdio.h>
void setbuf(FILE *stream, char *buf);

// 此函数相当于：
setvbuf(fp, buf, (buf != NULL) ? _IOFBF: _IONBF, BUFSIZ);
                        // BUFSIZ 定义在<stdio.h>中
```

```cpp
#define _BSD_SOURCE
#include <stdio.h>
void setbuffer(FILE *stream, char *buf, size_t size);

// 此函数相当于：
setvbuf(fp, buf, (buf != NULL) ? _IOFBF : _IONBF, size);
```

**Flushing a stdio buffer**

`fflush`将强制将stdio缓冲区的数据写入到内核缓冲区。

```cpp
#include <stdio.h>
int fflush(FILE *stream);
```

如果参数`stream`为NULL, `fflush`将刷新所有stdio缓冲区。`fflush`用户read stream 时会把读缓冲区中的数据全部丢弃。

**stdio缓冲区刷新时机** 
除了 无缓冲、行缓冲、全缓冲这三个模式下规定的刷新时机，还有以下两种情况。
- 文件流关闭时，会自动调用`fflush`
- 某些c库会当stdin有数据输入时，会`fflush(stdout)`

### 控制内核I/O缓冲

**刷新内核缓冲**  
```cpp
#include <unistd.h>
int fsync(int fd);
int fdatasync(int fd);
void sync(void);
```
`fdatasync`和`fsync`区别在`fdatasync`在文件元信息(大小、权限)未改变的情况下只同步文件数据，若文件大小、权限改变了才会同步文件元信息。

内核缓冲保存在文件系统的Page cache中， Page cache中被修改的页称为脏页(Dirty Page)。  
**内核缓冲写入磁盘的的时机和条件:**  

1. 当空闲内存低于一个特定的阈值时，内核必须将脏页写回磁盘，以便释放内存。可以通过`sysctl vm.dirty_background_ratio`命令查看参数，当脏页占比达到参数值所代表的百分比时，就会触发flush把脏数据写回磁盘。
2. 当脏页在内存中驻留时间超过一个特定的阈值时，内核必须将超时的脏页写回磁盘。`sysctl vm.dirty_expire_centisecs`可以看，单位时1/100秒。
3. 用户进程调用sync(2)、fsync(2)、fdatasync(2)系统调用时，内核会执行相应的写回操作。

**其它Dirty page可配置参数**
```sh
sysctl -a | grep dirty
 vm.dirty_background_ratio = 10
 vm.dirty_background_bytes = 0
 vm.dirty_ratio = 20
 vm.dirty_bytes = 0
 vm.dirty_writeback_centisecs = 500
 vm.dirty_expire_centisecs = 3000

```
> vm.dirty_background_ratio is the percentage of system memory that can be filled with “dirty” pages — memory pages that still need to be written to disk — before the pdflush/flush/kdmflush background processes kick in to write it to disk. My example is 10%, so if my virtual server has 32 GB of memory that’s 3.2 GB of data that can be sitting in RAM before something is done.

> vm.dirty_ratio is the absolute maximum amount of system memory that can be filled with dirty pages before everything must get committed to disk. When the system gets to this point all new I/O blocks until dirty pages have been written to disk. This is often the source of long I/O pauses, but is a safeguard against too much data being cached unsafely in memory.

> vm.dirty_background_bytes and vm.dirty_bytes are another way to specify these parameters. If you set the _bytes version the _ratio version will become 0, and vice-versa.

> vm.dirty_expire_centisecs is how long something can be in cache before it needs to be written. In this case it’s 30 seconds. When the pdflush/flush/kdmflush processes kick in they will check to see how old a dirty page is, and if it’s older than this value it’ll be written asynchronously to disk. Since holding a dirty page in memory is unsafe this is also a safeguard against data loss.

> vm.dirty_writeback_centisecs is how often the pdflush/flush/kdmflush processes wake up and check to see if work needs to be done.

**O_SYNC**

```cpp
fd = open(pathname, O_WRONLY | O_SYNC);
```
open后后续写入回直接写入磁盘。

