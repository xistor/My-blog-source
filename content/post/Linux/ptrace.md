---
title: "ptrace 笔记"
date: 2021-03-02T22:01:00+08:00
tags: ["Linux"]
categories: ["Linux"]
---


笔记主要参考[Playing with ptrace](https://www.linuxjournal.com/article/6100)， 原文比较久远，而且是在程序是在32位机器上跑的，修改成了64位机器能正确运行的版本。

ptrace() 系统调用提供了一个机制供一个父进程可以追踪和控制其他进程，主要用来实现断点debug以及系统调用追踪等功能。
函数原型如下：

```c
#include <sys/ptrace.h>
long ptrace(enum __ptrace_request request, pid_t pid, void *addr, void *data);
```
https://man7.org/linux/man-pages/man2/ptrace.2.html  


要更容易的看懂例程，需要先了解以下x86的寄存器，32bit和64bit的寄存器存在一些不同,如下图(来自参考3 page27)


{{< figure src="/img/ptrace/AMD_64bit_reg.png"  class="center" title="General Purpose Registers in 64-Bit Mode" width="600">}}

32bit下常用的`eax`, `ebx`在64bit下对应为 `rax`, `rbx`，但是使用`eax`依然可以访问`rax`的低32位。  

在32位情况下，系统调用号被放到`eax`, 传给系统调用的参数被依次放到 `ebx`, `ecx`, `edx`, `esi`, `edi`。在64位系统下，情况有些不同，我们使用[godblolt](https://gcc.godbolt.org)将下面的C程序翻译成汇编代码看一下寄存器使用情况，

```c
#include<unistd.h>
int main() {

    write(1, "hello!", 6);
    return 0;
}
```

翻译出来的汇编代码：
```
.LC0:
        .string "hello!"
main:
        push    rbp
        mov     rbp, rsp
        mov     edx, 6
        mov     esi, OFFSET FLAT:.LC0
        mov     edi, 1
        call    write
        mov     eax, 0
        pop     rbp
        ret
```

可见64位上，系统调用`write()`的三个参数依次被放到了`edi`, `esi`, `edx` 中，这里只使用了寄存器的低32位，所以还是e开头。各架构在系统调用时用到的寄存器如下:

|arch |	syscall number|	return|	arg0 | arg1| arg2| arg3| arg4| arg5|
|-----|---------------|-------|------|-----|-----|-----|-----|-----|
|arm|	r7|	r0|	r0|	r1|	r2|	r3|	r4|	r5|
|arm64|	x8|	x0|	x0|	x1|	x2|	x3|	x4|	x5|
|x86|	eax| eax| ebx| ecx|	edx| esi| edi| ebp|
|x86_64| rax| rax| rdi|	rsi| rdx| r10| r8| r9|



完整的寄存器常用法表见： https://web.stanford.edu/class/cs107/guide/x86-64.html  

知道了这几个寄存器的一般用法，就可以开始看例子了：

```c
#include <sys/ptrace.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>
#include <sys/reg.h>


int main()
{   pid_t child;    
    long orig_rax;
    child = fork();
    if(child == 0) {
        ptrace(PTRACE_TRACEME, 0, NULL, NULL);
        execl("/bin/ls", "ls", NULL);
    }
    else {
        wait(NULL);
        orig_rax = ptrace(PTRACE_PEEKUSER,
                          child, 8 * ORIG_RAX,
                          NULL);
        printf("The child made a "
               "system call %ld\n", orig_rax);
        ptrace(PTRACE_CONT, child, NULL, NULL);
    }
    return 0;
}
```

程序的主进程会追踪子进程的系统调用，将其系统调用号打印出来，然后调用`ptrace(PTRACE_CONT,...)` 让子进程继续运行。当系统调用发生时，内核会保存`rax`的原始内容到内存中，里面的内容就是系统调用号，可以从子进程的USER段读取出来，其偏移地址就是我们传入的`8 * ORIG_RAX`， `ORIG_RAX`定义在`sys/reg.h`文件中，其定义为`#define ORIG_RAX 15` ，因为64bit系统里，USER中的每个数据为8个字节，而orig_rax是第15个数据。USER的数据结构体定义在/usr/include/x86_64-linux-gnu/sys/user.h `struct user_regs_struct` 可以看一下。  


运行输出
```
The child made a system call 59
...
```


查阅系统调用号表 https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl ， 系统调用为`execve`。


看第二个例程：

```cpp
#include <sys/ptrace.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>
#include <sys/reg.h>
#include <sys/syscall.h>
#include <sys/user.h>
#include <stdio.h>

int main()
{   pid_t child;
    long orig_rax, rax;
    long params[3];
    int status;
    int insyscall = 0;
    struct user_regs_struct regs;
    child = fork();
    if(child == 0) {
        ptrace(PTRACE_TRACEME, 0, NULL, NULL);
        execl("/bin/ls", "ls", NULL);
    }
    else {
    
       while(1) {
          wait(&status);
          if(WIFEXITED(status))
              break;
          orig_rax = ptrace(PTRACE_PEEKUSER,
                            child, 8 * ORIG_RAX,
                            NULL);
          if(orig_rax == SYS_write) {
              if(insyscall == 0) {
                 /* Syscall entry */
                 insyscall = 1;
                 ptrace(PTRACE_GETREGS, child,
                        NULL, &regs);
                 printf("Write called with "
                        "%ld, %ld, %ld\n",
                        regs.rdi, regs.rsi,
                        regs.rdx);
             }
             else { /* Syscall exit */
                 rax = ptrace(PTRACE_PEEKUSER,
                              child, 8 * RAX,
                              NULL);
                 printf("Write returned "
                        "with %ld\n", rax);
                 insyscall = 0;
             }
          }
          ptrace(PTRACE_SYSCALL, child,
                 NULL, NULL);
       }
   }
   return 0;
}
```
和第一个例子区别不大，这段程序使用 `ptrace(PTRACE_GETREGS, ...)`函数获取系统调用时所有寄存器的值， 并打印出`rdi`, `rsi`, `rdx`中的值也就是`write()`的三个参数值, 在`write()`返回的时候打印出`rax`中的返回值。  

输出类似于：
```
Write called with 1, 9348640, 99
a.out	      foo.c	      libbar.so		libnice.a     mod1.cpp	nice.cpp	 rtsched	  test_reg.s  wrapjack
Write returned with 99
Write called with 1, 9348640, 103
bar.c	      foo.map	      libdemo.a		libtom.so     mod1.o	nice.o		 rtsched.cpp	  tlpi-dist   wrapjack2
Write returned with 103

```

第三例子，修改系统调用的参数，

```c
#include <sys/ptrace.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>
#include <sys/user.h>
#include <sys/syscall.h>
#include <sys/reg.h>

const int long_size = sizeof(long);
void reverse(char *str)
{   
    int i, j;
    char temp;
    for(i = 0, j = strlen(str) - 2;
        i <= j; ++i, --j) {
        temp = str[i];
        str[i] = str[j];
        str[j] = temp;
    }
}

void getdata(pid_t child, long addr, char *str, int len)
{
    char *laddr;
    int i, j;
    union u {
            long val;
            char chars[long_size];
    }data;
    i = 0;
    j = len / long_size;
    laddr = str;
    while(i < j) {
        data.val = ptrace(PTRACE_PEEKDATA,
                          child, addr + i * 8,
                          NULL);
        memcpy(laddr, data.chars, long_size);
        ++i;
        laddr += long_size;
    }
    j = len % long_size;
    if(j != 0) {
        data.val = ptrace(PTRACE_PEEKDATA,
                          child, addr + i * 8,
                          NULL);
        memcpy(laddr, data.chars, j);
    }
    str[len] = '\0';
}

void putdata(pid_t child, long addr, char *str, int len)
{   
    char *laddr;
    int i, j;
    union u {
            long val;
            char chars[long_size];
    }data;
    i = 0;
    j = len / long_size;
    laddr = str;
    while(i < j) {
        memcpy(data.chars, laddr, long_size);
        ptrace(PTRACE_POKEDATA, child,
               addr + i * 8, data.val);
        ++i;
        laddr += long_size;
    }
    j = len % long_size;
    if(j != 0) {
        memcpy(data.chars, laddr, j);
        ptrace(PTRACE_POKEDATA, child,
               addr + i * 8, data.val);
    }
}

int main()
{
    pid_t child;
    child = fork();
    if(child == 0) {
        ptrace(PTRACE_TRACEME, 0, NULL, NULL);
        execl("/bin/ls", "ls", NULL);
    }
    else {
        long orig_rax;
        long params[3];
        int status;
        char *str, *laddr;
        int toggle = 0;
        while(1) {
            wait(&status);
            if(WIFEXITED(status))
                break;
            orig_rax = ptrace(PTRACE_PEEKUSER,
                            child, 8 * ORIG_RAX,
                            NULL);
            if(orig_rax == SYS_write) {
                if(toggle == 0) {
                toggle = 1;
                params[0] = ptrace(PTRACE_PEEKUSER,
                                    child, 8 * RDI,
                                    NULL);
                params[1] = ptrace(PTRACE_PEEKUSER,
                                    child, 8 * RSI,
                                    NULL);
                params[2] = ptrace(PTRACE_PEEKUSER,
                                    child, 8 * RDX,
                                    NULL);
                str = (char *)calloc((params[2]+1), sizeof(char));
                getdata(child, params[1], str,
                        params[2]);
                reverse(str);
                putdata(child, params[1], str,
                        params[2]);
                }
                else {
                toggle = 0;
                }
            }
        ptrace(PTRACE_SYSCALL, child, NULL, NULL);
        }
    }
    return 0;
}
```

输出类似于下面这样
```sh
$ ls

a.out   fun.c 

$ ./a.out

c.nuf   tuo.a

```

程序使用 `PTRACE_POKEDATA` 修改传给`write()`的参数，`ssize_t write(int fd, const void *buf, size_t count)` 三个参数分别为要写入的文件描述符，buf指针, 写入的字节数。 `getdata()`的作用是调用`ptrace(PTRACE_PEEKDATA,..)`以8个字节为单位取得参数`buf`指向的数据后，写入`str`指向的地址。之后反转字符串再调用`putdata()`使用`ptrace(PTRACE_POKEDATA,..)`写回去，就实现了上面的效果。  

前几个例子，子进程开始时都调用了 `ptrace(PTRACE_TRACEME, 0, NULL, NULL)`来告诉内核对其追踪，但不是每个程序都会去调用这个的，而且我们经常随便拿来一个程序就用`strace`命令跟踪系统调用，那些程序里面也不会都调用了`PTRACE_TRACEME`吧。 要trace一个既有的进程也是可以的，只要使用`ptrace(PTRACE_ATTACH, ..)`就可以了。接下来我们就用`PTRACE_ATTACH`来跟踪一个进程。   

首先写一个简单的程序供我们跟踪, 这个小程序会每两秒计数并输出。

```c
// counter

#include<stdio.h>

int main() {
    for (int i = 0; i < 100; ++i) {
        printf("Counter: %d\n", i);
        sleep(2);
    }
    return 0;
}
```

然后使用`PTRACE_ATTACH`写我们的追踪程序, 结合一下之前的例子，将被追踪程序的输出也截取出来。

```c
// trace_counter

#include <stdio.h>
#include <stdlib.h>
#include <sys/ptrace.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>
#include <sys/reg.h>
#include <sys/user.h>
#include <sys/syscall.h>
#include <string.h>

int long_size = sizeof(long);

void getdata(pid_t pid, long addr, char *str, int len)
{
    char *laddr;
    int i, j;
    union u {
            long val;
            char chars[long_size];
    }data;
    i = 0;
    j = len / long_size;
    laddr = str;
    while(i < j) {
        data.val = ptrace(PTRACE_PEEKDATA,
                          pid, addr + i * 8,
                          NULL);
        memcpy(laddr, data.chars, long_size);
        ++i;
        laddr += long_size;
    }
    j = len % long_size;
    if(j != 0) {
        data.val = ptrace(PTRACE_PEEKDATA,
                          pid, addr + i * 8,
                          NULL);
        memcpy(laddr, data.chars, j);
    }
    str[len] = '\0';
}


int main(int argc, char *argv[])
{   
    long orig_rax, rax;
    int status;
    int insyscall = 0;
    pid_t traced_process;
    struct user_regs_struct regs;
    long ins;
    if(argc != 2) {
        printf("Usage: %s <pid to be traced>\n", argv[0]);
        exit(1);
    }
    traced_process = atoi(argv[1]);
    ptrace(PTRACE_ATTACH, traced_process, NULL, NULL);

    while(1) {
        wait(&status);
        if(WIFEXITED(status))
            break;
        orig_rax = ptrace(PTRACE_PEEKUSER, traced_process, 8 * ORIG_RAX, NULL);
        if(orig_rax == SYS_write) {
            if(insyscall == 0) {
                /* Syscall entry */
                insyscall = 1;
                ptrace(PTRACE_GETREGS, traced_process,
                    NULL, &regs);
                printf("Write called with %lld, %lld, %lld\n", regs.rdi, regs.rsi, regs.rdx);
                char *str = (char *)calloc((regs.rdx+1), sizeof(char));
                getdata(traced_process, regs.rsi, str, regs.rdx);
                printf("write string is : %s", str);
            }
            else { /* Syscall exit */
                rax = ptrace(PTRACE_PEEKUSER, traced_process, 8 * RAX, NULL);
                printf("Write returned with %ld\n", rax);
                insyscall = 0;
            }
        }
        ptrace(PTRACE_SYSCALL, traced_process, NULL, NULL);
    }

    ptrace(PTRACE_DETACH, traced_process, NULL, NULL);
    
    return 0;
}

```

编译两个程序，然后在两个终端运行:  

counter 的输出：
```sh
Counter: 0
Counter: 1
Counter: 2
Counter: 3
Counter: 4
Counter: 5
...
```

trace_counter 的输出：

```sh
Write called with 1, 28184592, 11
write string is : Counter: 7
Write returned with 11
Write called with 1, 28184592, 11
write string is : Counter: 8
Write returned with 11
Write called with 1, 28184592, 11
write string is : Counter: 9
Write returned with 11
Write called with 1, 28184592, 12
write string is : Counter: 10
Write returned with 12

...

```



设置断点：

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <sys/ptrace.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>
#include <sys/reg.h>
#include <sys/user.h>
#include <sys/syscall.h>
#include <string.h>


const int long_size = sizeof(long);

void getdata(pid_t pid, long addr, char *str, int len)
{
    char *laddr;
    int i, j;
    union u {
            long val;
            char chars[long_size];
    }data;
    i = 0;
    j = len / long_size;
    laddr = str;
    while(i < j) {
        data.val = ptrace(PTRACE_PEEKDATA,
                          pid, addr + i * long_size,
                          NULL);
        memcpy(laddr, data.chars, long_size);
        ++i;
        laddr += long_size;
    }
    j = len % long_size;
    if(j != 0) {
        data.val = ptrace(PTRACE_PEEKDATA,
                          pid, addr + i * long_size,
                          NULL);
        memcpy(laddr, data.chars, j);
    }
    str[len] = '\0';
}

void putdata(pid_t child, long addr, char *str, int len)
{   
    char *laddr;
    int i, j;
    union u {
            long val;
            char chars[long_size];
    }data;
    i = 0;
    j = len / long_size;
    laddr = str;
    while(i < j) {
        memcpy(data.chars, laddr, long_size);
        ptrace(PTRACE_POKEDATA, child,
               addr + i * long_size, data.val);
        ++i;
        laddr += long_size;
    }
    j = len % long_size;
    if(j != 0) {
        memcpy(data.chars, laddr, j);
        ptrace(PTRACE_POKEDATA, child,
               addr + i * long_size, data.val);
    }
}

int main(int argc, char *argv[])
{   pid_t traced_process;
    struct user_regs_struct regs, newregs;
    long ins;
    /* int 0x80, int3 */
    char code[] = {0xcd,0x80,0xcc,0};
    char backup[long_size];
    if(argc != 2) {
        printf("Usage: %s <pid to be traced>\n", argv[0]);
        exit(1);
    }
    traced_process = atoi(argv[1]);
    ptrace(PTRACE_ATTACH, traced_process,
           NULL, NULL);
    wait(NULL);
    ptrace(PTRACE_GETREGS, traced_process,
           NULL, &regs);
    /* Copy instructions into a backup variable */
    getdata(traced_process, regs.rip, backup, long_size);
    /* Put the breakpoint */
    putdata(traced_process, regs.rip, code, 3);
    /* Let the process continue and execute
       the int 3 instruction */
    ptrace(PTRACE_CONT, traced_process, NULL, NULL);
    wait(NULL);
    printf("The process stopped, putting back "
           "the original instructions\n");
    printf("Press <enter> to continue\n");
    getchar();
    putdata(traced_process, regs.rip, backup, long_size);
    /* Setting the eip back to the original
       instruction to let the process continue */
    ptrace(PTRACE_SETREGS, traced_process, NULL, &regs);
    ptrace(PTRACE_DETACH, traced_process, NULL, NULL);
    return 0;
}

```

设置断点的主要原理是读取程序的`rip`（instruction pointer）寄存器，将其指向的指令保存下来， 然后用`int3（0xCC）`指令替换掉原指令，当CPU遇到`int3`会发送`SIGTRAP`信号给调试进程，将进程stop。之后再将指令恢复，并将`rip`指回原处，程序就可以继续执行下去了。

参考：  
0. https://www.linuxjournal.com/article/6100
1. 系统调用号表 https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl
2. 寄存器列表 https://web.stanford.edu/class/cs107/guide/x86-64.html
3. https://www.amd.com/system/files/TechDocs/24592.pdf
4. https://theantway.com/2013/01/notes-for-playing-with-ptrace-on-64-bits-ubuntu-12-10/
5. https://abda.nl/posts/understanding-ptrace/