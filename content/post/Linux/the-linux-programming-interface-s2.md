---
title: "Linux/Unix系统编程手册-笔记2.系统调用"
date: 2020-06-21T16:01:00+08:00
tags: ["Linux"]
categories: ["Linux系统编程手册阅读"]
---

### 系统调用

- 系统调用将处理器从用户态切换到核心态，以便CPU访问收到保护的内核内存。
- 系统调用的组成是固定的。每个系统调用都有一个唯一的数字来标识。
- 每个系统调用的参数，可以对用户态和内核态之间相互传递的参数加以规范。

系统调用步骤：
1. 应用程序调用C语言库中的外壳(wrapper)函数，发起系统调用。
2. 外壳函数确保参数可用，并将参数复制到寄存器，供内核使用。
3. 外壳函数将系统调用编号复制到一个特殊的CPU寄存器（%eax）中。
4. 外壳函数执行一条中断机器指令（int 0x80）,引发处理器从用户态切换到核心态，并执行系统中断0x80的中断矢量所指向的代码。
5. 为享用中断0x80,内核会调用system_call()响应。
    a. 在内核中保存寄存器值。
    b. 审核系统调用编号的有效性。
    c. 以系统调用编号查找对应的服务程序，检查参数有效性，执行服务程序，最后服务程序会将结果返回给system_call()。
    d. 从内核中恢复各寄存器的值，并将系统调用返回值置于栈中。
    e. 返回至外壳函数， 同时将处理器切换回用户态。
6. 若系统调用服务程序的返回值表明调用有误，外壳函数会使用该返回值来设置全局变量errno

strace 命令可以查看系统调用过程，写一个小程序一探究竟，程序功能为往文件中写一句“hello”：

```
#include<iostream>
#include <sys/types.h>
#include <unistd.h>
#include <fstream>

int main()
{
    std::ofstream in;
    in.open("hello.txt",std::ios::trunc);
    in << "hello";
    in.close();
    return 0;
}
```

编译后执行`strace  -o ./straceout.txt ./a.out`，输出到调用过程到straceout.txt文件中，内容如下，“=”左边是系统调用，右边是返回值

```
...

mprotect(0x7fc7e9fb5000, 4096, PROT_READ) = 0
munmap(0x7fc7e9f98000, 118140)          = 0
brk(NULL)                               = 0x560971b81000
brk(0x560971ba2000)                     = 0x560971ba2000
openat(AT_FDCWD, "hello.txt", O_WRONLY|O_CREAT|O_TRUNC, 0666) = 3
write(3, "hello", 5)                    = 5
close(3)                                = 0
exit_group(0)                           = ?
+++ exited with 0 +++
```
省略了前面一坨，在文件最后，和程序功能相关的几个系统调用就在其中，openat打开一个文件并返回一个文件描述符，write向里面写了“hello”,
然后close文件，程序退出。


### 处理系统调用错误

系统调用失败时，会将全局整形变量errno设置一个正值，以标识具体的错误（需包含<errno.h>头文件）。如果调用系统和库函数成功，errno绝不会被重置为0，所以errno可能为上次调用失败留下的值。因此进行错误检查时，必须首先检查函数的返回值是否表明调用出错，然后检查errno确定错误原因。

函数strerror可以将错误号转成容易理解的错误字符串

```
#include<string.h>
char* strerror(int errnum);
```

需要注意的是strerror返回的字符串指针指向的是一块静态分配的内存，所以后续调用strerror可能会覆盖该字符串。


### 编译书中带的源码

下载链接 https://man7.org/tlpi/code/ 一般下载Distribution version即可。
下载解压后，进入tlpi-dist目录执行`make`,若报错

```
cc: error: unrecognized command line option ‘-Wimplicit-fallthrough’
```

考虑将gcc版本升到7以上，参考 https://www.jianshu.com/p/7a8878397213 


### 可移植性问题

书中给了几个可移植性问题的例子：
0. 数据类型，系统相关的数据类型如pid_t等在不同系统上实现不同，所以程序应使用定义好的系统数据类型，而不是原生的int, long等。
1. 结构体在不同系统实现下，内部元素存储顺序不定，所以最好不使用列表初始化的方式: `struct example s = {1, 2, 3}`,而使用下面这种方式。
```
struct example s; 
s.a=1;
s.b=2;
s.c=3;
```

2. 有些宏可能并没有在所有类UNIX系统上实现，使用前需要先判断是否已定义。如 WCOREDUMP在SUSv3就没有定义。
3. 有些时候有些头文件在某些系统上不需要包含。


### Exercise
reboot()系统调用

```
int reboot(int magic, int magic2, int cmd, void *arg);
```

参数magic需要等于 LINUX_REBOOT_MAGIC1(0xfee1dead) 且magic2 为以下之一才不会调用失败(应该是为了避免误调)

```
LINUX_REBOOT_MAGIC2 ( 672274793)
LINUX_REBOOT_MAGIC2A ( 85072278)   since 2.1.17
LINUX_REBOOT_MAGIC2B ( 369367448)  since 2.1.97
LINUX_REBOOT_MAGIC2C ( 537993216)  since 2.5.71 
```

这几个参数的意义是 Linus Torvalds和他三个女儿的生日。