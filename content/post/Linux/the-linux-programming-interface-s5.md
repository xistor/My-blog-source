---
title: "Linux/Unix系统编程手册-笔记5.进程"
date: 2020-07-16T14:01:00+08:00
tags: ["Linux"]
categories: ["Linux系统编程手册阅读"]
---

### 进程和程序
进程是一个运行中的程序实例，一个程序中包含了如何构建一个进程的信息，这些信息包括：
- 二进制格式识别信息：每个程序都包含了元信息来描述可执行文件的格式，内核依靠这个来解释文件中的剩余信息。
历史上广泛使用的有a.out(assembler output)和之后的COFF(Common Object File Format),现在大多使用Executable and Linking Format (ELF)。
- 机器指令
- 程序入口地址：标识程序开始执行的指令的位置
- data: 程序中用于初始化变量的值或者程序用到的常量
- Symbol and relocation tables:描述函数或变量名称以及在程序中的位置，用于debug或dynamic linking
- 共享库和动态链接信息：程序文件中包含字段列出了程序运行时需要的所有共享库，以及用来载入lib的动态链接器的路径名
- 其他信息

从系统角度来看，一个进程包括一段用户空间内存，里面有程序的代码和用到的变量等等，以及一段内核空间内存，里面维护了程序状态等信息（PID, 虚拟内存表，open file descriptors, 信号传递和处理的相关信息， 进程资源使用及限制，当前工作目录，以及其他信息）。


### 进程ID和父进程ID

```c
#include <unistd.h>
pid_t getpid(void); /*获取进程ID*/

pid_t getppid(void); /*获取父进程ID*/    
```

Linux 内核限制PID小于等于32767,当分配的PID到达了32767，会从300开始再次分配，因为较小的PID多数为系统进程或守护进程。Linux 2.6之后可以通过/proc/sys/kernel/pid_max来调整，在64位系统上可以到2^22。

如果一个进程的父进程终止，它会变为孤儿进程，会被init“领养”，getppid()会返回init的进程号，也就是1。

### 进程的内存布局

- 代码段(text):包含了程序的机器码指令，代码段只读，不同进程可以共享代码段。
-  initialized data segment： 保存显式初始化过的全局变量和静态变量。这些变量的值是在程序载入内存时从可执行文件内读取对的。
- uninitialized data segment： 保存未显式初始化过的全局变量和静态变量。在程序启动之前，这部分内存会被初始化为0。也称BSS段。此段被和已经初始化的变量分开的原因是，当程序保存在硬盘上的时候是不需要保存这些没初始化的变量的，但进程跑起来的时候需要，而且部分空间是在程序载入内存的时候才会分配。
- 栈区(stack): 存储局部变量，参数，返回值。
- 堆(heap): 运行时动态分配的内存。

size命令可以查看可执行文件的各段的大小：

```
~/cpp/test/$ size
text data bss dec hex filename
2194 616 280 3090 c12 a.out

```

### 虚拟内存管理

引入虚拟内存是为了更高效的利用CPU和RAM,因为程序存在局部性。
- 空间局部性：程序会趋向于访问最近访问过的内存地址附近的数据，因为程序指令连续，还有数据一般也是连续存储。
- 时间局部性：程序会趋向于访问最近访问过的同一块内存（因为程序中有循环）。


{{< figure src="/img/the-linux-programming-interface-s5/vm_layout.png"  class="center" title="虚拟内存布局" width="500" >}}

图中的argv和environ是命令行输入的参数和进程环境变量。 etext, edata, 和end可以获得程序段的地址。
使用方法,在程序中声明：

```c
extern char etext, edata, end;
```

虚拟内存会将程序使用的内存分成等大小的单元，称为“页”(pages)。在任意时刻只有程序需要的页才会在内存中，成为常驻内存集，其他未用的页放在硬盘上的swap区域。当程序访问到了不在物理内存上的页时会触发page fault,内核会挂起程序，将所需的页从硬盘读入到内存中。

{{< figure src="/img/the-linux-programming-interface-s5/overview_of_vm.png"  class="center" title="虚拟内存" width="600" >}}

如图，内核会为每一个进程维护一个page table,并不是进程的所有虚拟内存地址都有对应的page table入口，通常有一大部分虚拟地址空间是没用到的。当进程试图访问没有一个没有page table入口的地址时，会收到SIGSEGV信号。  

虚拟内存的好处：
- 进程之间相互隔离，通过将每个进程的page-table指向相互分离的物理内存就可以实现。
- 在适当的时候，进程间可以共享内存，内核可以通过将不同进程的page-table指向同一个物理内存上的page来实现。
- 内存保护机制很好实现，因为page-table入口可以标记对应的页是可读、可写或者可执行。当进程间共享内存时就可以针对不同进程设置不同的内存保护等级。
- 程序员和编译器、链接器不必考虑程序在RAM中的物理布局。
- 程序只需要一部分常驻内存，程序加载运行将更快。

### 栈
栈是随着函数调用和返回自动增长缩小的，存在一个特殊的寄存器stack pointer跟踪当前的栈顶。
在大多数实现中，栈帧被释放后并不会还给系统，而是留着复用。
user 栈里主要有两种信息：
- 函数参数和局部变量。
- 调用信息：函数会使用特定的CPU 寄存器，比如程序计数器(program counter)，当函数调用其他函数时，会把寄存器copy一份保存到栈中，以便在调用返回时恢复寄存器。

### 环境变量
每个进程会收到一份其父进程的环境变量拷贝。
在c程序中访问环境变量：
- 使用全局变量 `char **environ` 访问环境列表。
- 在main()函数参数中添加声明
```c
int main(int argc, char *argv[], char *envp[])
```
- getenv()函数从进程环境中检索单个值
```c
#include <stdlib.h>

char *getenv(const char *name); // 返回指向字符串的指针，若没找到为NULL
```


### 非局部跳转

c 中的goto语句可以在函数内部跳转，setjmp()和longjmp()提供跨越函数跳转的功能。

```c
#include <setjmp.h>

int setjmp(jmp_buf env);  /*第一次调用返回0， 通过longjmp()返回的为longjmp()的参数val指定的非零值*/

void longjmp(jmp_buf env, int val); /*若val为0会被替换成1*/
```

setjmp()调用为后续由longjmp()调用执行的跳转确立了跳转目标，即调用longjmp()会跳到setjmp()调用的位置。setjmp()两次被调用的区别在于返回值不同。

setjmp函数的使用限制：
- 构成选择或迭代语句中(if、switch、while等)的整个控制表达式
- 作为一元操作符!(not)的操作对象，其最终表达式构成了选择或迭代语句的整个控制表达式。
- 作为比较（==、!=、<）等的一部分，另一操作对象必须是一个整数常量表达式，且其最终表达式构成了选择或迭代语句的整个控制表达式。
- 作为独立函数调用  

`s = setjmp(env);`语句是不符合标准的。  

之所以有这些限制，是因为作为常规函数的setjmp()实现无法保证拥有足够的信息来保存所有寄存器值和封闭表达式中用到的临时栈位置，因此，仅允许在足够简单且无需临时存储的表达式中调用setjmp()。

滥用longjmp():  
longjmp()的调用不能跳转到一个已经返回的函数，因为函数返回后，env中保存的栈信息已经失效了。


编译器优化  
优化器对代码的优化会受到longjmp() 干扰，因此最好将局部变量声明为volatile，但最好尽可能避免使用setjmp()和longjmp()。

 ### Exercises
 1. 编译后的mem_segments使用ll查看结果：

 ```
-rwxr-xr-x  1 x x 11624 Jul 20 23:35 mem_segments*
 ```

 使用命令`size mem_segments`查看：
 ```
text	   data	    bss	        dec	        hex	    filename
1918	   636	    10305568	10308122	9d4a1a	mem_segments
 ```
 可见bss区的大小已经10305568字节(大约9.8M)了,原因是bss段保存未初始化的全局变量和静态变量，当程序保存在硬盘上的时候是不需要保存这些没初始化的变量的，其空间是在程序载入内存的时候才会分配。

 