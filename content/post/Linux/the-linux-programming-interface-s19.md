---
title: "Linux/Unix系统编程手册-笔记19.信号"
date: 2020-08-25T18:25:00+08:00
tags: ["Linux"]
categories: ["Linux系统编程手册阅读"]
---

信号通常用来通知某个事件发生，一个进程(有合适的权限的话)可以给另外的进程发送信号，但大多数情况下信号是由kernel发送给进程的。
> 信号和软中断的区别：中断可以看作是CPU和OS kernel之间的通信，信号是OS kernel和进程之间的通信。中断可以由CPU(异常，如:除以零，页面错误)，设备(硬件中断，如:可用输入)，或者由CPU指令(陷阱，如系统调用，断点)发起。它们最终由CPU管理，它“中断”当前任务，并调用os内核提供的ISR/中断处理程序。信号由操作系统内核（如：SIGFPE, SIGSEGV, SIGIO），或者其他进程（kill()）发起。由操作系统内核管理，内核将信号发送给目标线程/进程，调用一个默认动作(ignore, terminate, termiante and dump core) 或者调用进程自定义的handler。  

## 信号类型

信号分两大类：一种是标准信号，编号从1到31，定义在<signal.h>中，被内核用来通知进程事件发生。
另一种是实时信号，信号取值区间SIGRTMIN~SIGRTMAX，没有明确的含义，而是由使用者自己来决定如何使用。

定义的信号如下表，signal number列中字母代表平台架构，S=Sun SPARC, A=Digital Alpha, M=MIPS, P=HP PA_RISC
![信号表](/img/the-linux-programming-interface-s19/signal_number.png)

## 改变信号的处置方式

使用`signal()`函数可以设立一个信号处理handler。参数`sig`是我们想要改变处置方式的信号，`handler`是当信号到达时，我们想要调用的函数，`signal()`的返回值是前一个handler函数。

```cpp
#include <signal.h>

void ( *signal(int sig, void (*handler)(int)) ) (int);
```

当信号处理函数被调用时，可能在任何时间中断主程序流程，内核代表进程取表用处理函数，在处理函数返回后，在程序中断的地方继续执行。信号处理函数的参数的传入参数为信号编号，利用这个可以使用一个信号处理函数来处置多个信号。

## 发送信号

一个进程可以使用kill()给另外的进程发送信号，之所以命名为kill,是因为早期大多数信号的默认处置方式都会终止进程。

```c
#include <signal.h>
int kill(pid_t pid, int sig);   // retturn 0 on success, or -1 on error
```

对于参数pid：
- 如果pid大于0，信号会被发给对应pid的进程。
- pid = 0, 信号会被发给同一个进程组的所有进程，包括调用进程自己。
- pid < -1, 会取绝对值发送。
- pid = -1, 会发给所有有权限发送的进程，除了init和调用进程。


## 信号掩码
对于每个进程，内核维护一个信号掩码,在这个集合中的信号会被阻塞，直到信号从mask中移除后，才会被送达。（信号掩码实际是一个线程级的属性，使用pthread_mask()可配置）。  

一个信号可能以以下几种方式被加入到信号掩码中：
- 当信号处理函数被调用时，信号会自动加入信号掩码，这个是否出现取决于使用sigaction()注册信号处理函数时传入的flag（当包括SA_NODEFER时，在信号处理函数中也会响应信号）。
- 使用sigprocmask()

```c
#include <signal.h>

int sigprocmask(int how, const sigset_t *set, sigset_t *oldset);
```

how参数指定了函数想给掩码带来的变化  
- SIG_BLOCK 将set指向的信号集内的指定信号添加到信号掩码中
- SIG_UNBLOCK 将set指向的信号集内的信号从信号掩码中移除
- SIG_SETMASK 将set指向的信号集赋给信号掩码


sigprocmask()使用例子：

```c
sigset_t blockset, prevMask;

// 这两个函数定义在signal.h中
sigemptyset(&blockset);
sigaddset(&blockset, SIGINT);

sigprocmask(SIG_BLOCK, &blockset, &preMask);

```

## 处于等待中的信号

某进程接受了一个正在阻塞的信号，会把该信号加到进程的等待信号集中，使用`sigpending()`可以看

```c
#include <signal.h>

int sigpending(sigset_t *set);
```

等待信号集仅仅是一个掩码，仅表明一个信号是否发生，而未表明其发生的次数。换言之，如果同一个信号在阻塞状态下发生多次， 那么会将该信号记录在等待信号集中，并在稍后仅传递一次。

## sigaction()

相比signal()， sigaction()系统调用使用更加灵活，可移植性更佳。

```c
#include <signal.h>

int sigaction(int sig, const struct sigaction *act, struct sigation *oldact);

//  return 0 on success, or -1 on error
```
sig参数是想要获取或改变的信号编号  

参数act是指向描述信号新处置的数据结构，如果只想获取现有处置，可以置null。  
oldact和act类型相同，是只想之前信号处置的数据结构。

sigaction结构体如下：

```c
struct sigaction {
    void (*sa_handler)(int);
    sigset_t sa_mask;

    int sa_flags;
    void (*sa_restorer)(void);    
}
```
sa_handler: 信号handler  
sa_mask: handler调用时会阻塞的信号集  
sa_flags: 控制信号处理过程的各种选项  
sa_restorer: 仅仅内部使用，用以恢复进程上下文

## 等待信号

```c
#include <unistd.h>

int pause(void);
```

调用`pause()`将暂停进程的执行，直到一个进程处理函数中断该调用为止。