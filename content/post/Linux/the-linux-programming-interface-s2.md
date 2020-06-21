---
title: "Linux/Unix系统编程手册-笔记2.系统调用"
date: 2020-06-21T16:01:00+08:00
tags: ["Linux"]
categories: ["Linux系统编程手册阅读"]
---

#### 系统调用

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

编译后执行`strace  -o ./straceout.txt ./a.out`，输出到straceout.txt文件中，内容如下，“=”左边是系统调用，右边是返回值

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


