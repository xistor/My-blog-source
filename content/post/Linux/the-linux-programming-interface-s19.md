---
title: "Linux/Unix系统编程手册-笔记19.信号"
date: 2020-08-28T18:20:00+08:00
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

发送信号需要适当的权限：
- 特权级（CAP_KILL）进程可以向任何进程发送信号
- init例外，只能接受写了信号处理函数的信号，这是为了防止init意外被杀掉。
- 发送者的实际或有效用户ID匹配于接受者的实际用户ID或者保存设置用户ID（saved set-user-id），也就是说，用户可以向由他们启动的set-user-id程序发送信号。
- SIGCONT信号，非特权用户可以向同一会话中的任何其他进程发送这一信号。  

如果无权发送信号，kill()将调用失败，且将errno置为EPERM。

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

## 信号处理函数

信号处理函数尽量越简单越好，以避免数据竞争。  
一般的信号处理函数设计：
- 在信号处理函数内去设一个全局变量，在主程序中定时或者监控此变量的变化。
- 在信号处理函数内做一些清理工作，之后结束掉进程或通过非局部跳转到主程序中。


### 可重入和异步信号安全

可重入：一个函数在执行完毕前由于某种原因被中断，在中断处理程序中该函数被再次调用的情况下，函数在不被上述中断影响下的执行结果与出现上述中断情况下的最终执行结果一致。  

一个函数内如果写了全局变量或静态变量，那么它可能是不可重入的。若一个函数内只有局部变量，那么它一定是可重入的。

异步信号安全:一个异步信号安全函数是可重入的，或者不会被信号处理函数打断的。  

所以我们在写信号处理函数时一定要保证不去调用不安全的函数。

### 线程安全和异步信号安全

线程安全(thread-safety)和异步信号安全(async-signal-safety)概念有点相似，查了些资料，解释如下：
一般来说异步信号安全意味着线程安全（[反例](https://en.wikipedia.org/wiki/Reentrancy_(computing))）。线程安全意味着在多个线程内同时调用函数是没问题的。而异步信号安全要求更高一些，因为两次相关的调用可以发生在同一个线程。线程之间可以使用互斥锁来防止资源竞争。但信号处理函数和主程序之间不能使用锁这种东西，比如，在主程序使用`printf()`输出时，来了一个信号进入信号处理函数，在信号处理函数中也使用`printf()`输出，那么两个输出就会交织在一起。如果使用`mutex`来保护`printf()`，当前一个线程正在`printf()`，并持有和lock了`mutex`，这时来了一个信号，进入信号处理函数并也要去`printf()`,就会再次尝试获取`mutex`,然后就造成了死锁。

### 全局变量和sig_atomic_t

尽管存在可重入问题， 但信号处理函数和主程序之间共享全局变量还是非常有用的。为了保证个这个过程正确，一般可以这样声明共享的全局变量：

```c
volatile sig_atomic_t flag;
```
`sig_atomic_t`并不是一个C++中那样的原子类型，实际上一般是一个int，在signal.h可以看到定义是`typedef int sig_atomic_t`。因为int类型一般读或写只需要一条机器指令。


## 在备选栈中处理信号

当进程的栈增长达到了了限制，内核将为该进程产生SIGSEGC信号，不过，因为栈空间已经耗尽，信号处理函数也就无法被调用，进程就中止了。这就需要使用`signalstack()`来指定备选栈。

```c
#include <signal.h>

int signalstack(const stack_t *sigstack, stack_t *old_sigstack);

// sigstack为NULL时，仅通过old_sigstack返回上一备选栈的信息
```
其中`stack_t`结构体如下：
```c
typedef struct {
    void *ss_sp; /* Starting address of alternate stack */
    int ss_flags; /* Flags: SS_ONSTACK, SS_DISABLE */
    size_t ss_size; /* Size of alternate stack */
} stack_t;
```

`SS_ONSTACK` :  如果从old_sigstack中此项被设置了，说明当前正运行在备选栈上。
`SS_DISABLE` :  如果此项被设置了说明当前没有建立备选栈。

使用步骤：
1. 分配一块内存作为备选栈。
2. 调用`signalstack()`告知内核备选栈的存在。
3. 创建信号处理函数时指定SA_ONSTACK标志，

## SA_SIGINFO Flag

在调用sigaction()时使用SA_SIGINFO可以获得额外的信息，当然handler也得相应的改成下面这个形式：

```c
void handler(int sig, siginfo_t *siginfo, void *ucontext);
```
三个参数分别为：  
sig:信号编号  
siginfo：附加信息结构体，见下方  
ucontext： 包含kernel保存在用户空间栈内的信号上下文信息，一般来说信号处理函数中用不到这个。  

`siginfo`参数里提供了额外信息,handler的函数原型变了，那么前面说的sigaction结构体中的handler函数原型也就不适用了，实际上，<signal.h>中定义的sigaction结构体原型是下面这样的，

```c
struct sigaction {
    union {
    void (*sa_handler)(int);
    void (*sa_sigaction)(int, siginfo_t *, void *);
    } __sigaction_handler;
    sigset_t sa_mask;
    int sa_flags;
    void (*sa_restorer)(void);
};
```

这个siginfo_t长这样：
```c
typedef struct {
    int si_signo; /* Signal number */
    int si_code; /* Signal code */
    int si_trapno; /* Trap number for hardware-generated signal
    (unused on most architectures) */
    union sigval si_value; /* Accompanying data from sigqueue() */
    pid_t si_pid; /* Process ID of sending process */
    uid_t si_uid; /* Real user ID of sender */
    int si_errno; /* Error number (generally unused) */
    void *si_addr; /* Address that generated signal
    (hardware-generated signals only) */
    int si_overrun; /* Overrun count (Linux 2.6, POSIX timers) */
    int si_timerid; /* (Kernel-internal) Timer ID
    (Linux 2.6, POSIX timers) */
    long si_band; /* Band event (SIGPOLL/SIGIO) */
    int si_fd; /* File descriptor (SIGPOLL/SIGIO) */
    int si_status; /* Exit status or signal (SIGCHLD) */
    clock_t si_utime; /* User CPU time (SIGCHLD) */
    clock_t si_stime; /* System CPU time (SIGCHLD) */
} siginfo_t;

```

要定义 _POSIX_C_SOURCE 大于等于199309才能使用
> 关于_POSIX_C_SOURCE： _POSIX_C_SOURCE 1 makes the functionality from the POSIX.1 standart available _POSIX_C_SOURCE 2 makes the functionality from the POSIX.2 standart available _POSIX_C_SOURCE 199309L makes the functionality from the POSIX.1b standart available
Higher values like 200809L make more features available. (man 7 feature_test_macros)


## 中断和系统调用

当一个阻塞的系统调用被信号打断，在信号处理函数返回后，默认操作是系统调用会报`EINTR`并失败，但大多数时候，我们希望系统调用继续执行，有两种方法：
1. 将系统调用卸载while循环中：

```c
while ((cnt = read(fd, buf, BUF_SIZE)) == -1 && errno == EINTR)
 continue;
```
2. 在使用`sigaction()`建立信号处理函数时使用SA_RESTART标志，内核会自动帮忙重启系统调用（有些系统调用不支持）。

## 传递、处置及处理特殊情况

SIGKILL 信号默认行为是终止一个进程。SIGSTOP信号默认行为是停止一个进程，二者的默认行为无法改变，也无法阻塞。  
SIGCONT 如果一个进程处于停止状态信号,那么一个SIGCONT信号必然会使其恢复运行。  

每当信号收到SIGCONT信号时，会将处于等待状态的停止信号丢弃，同样的，如果任何停止信号传递给了进程，那么进程将自动丢弃任何处于等待状态的SIGCONT信号，以防止出现反复。

## 硬件产生的信号
硬件异常可以产生信号SIGBUS、 SIGFPE、SIGILL和SIGSEGV,正确处理硬件产生的信号有两种：要么接受信号的默认行为（进程终止），要么为其编写不会正常返回的处理函数。因为代码已经产生错误，无法执行下去，从信号处理函数中返回或忽略信号或阻塞信号都是未定义行为。 

## 信号传递的时机和顺序

传递时机：进程正在执行，且发生由内核到用户态的切换
- 在一个时间片的开始处
- 系统调用完成时

传递顺序： 同时解除多个阻塞信号时，Linux内核按照信号编号的升序来传递信号（依靠系统实现，不做保证）。多个解除了阻塞的信号正在等待传递时，如果在信号处理函数执行期间发生了内核态和用户态之间的切换，那么将中断此处理函数的执行，转去调用第二个信号处理器函数（如此递进）。



## 实时信号

与标准信号相比的区别：

- 可以自定义
- 队列化管理，实时信号多次发送给一个进程，将会多次传递信号。
- 发送实时信号时，可以指定伴随数据（一个整数或指针）供信号处理函数使用
- 不同实时信号的顺序得到保障，在等待序列中，将率先传递最小编号的信号。

### 发送实时信号

```c
#define _POSIX_C_SOURCE 199309
#include <signal>

int sigqueue(pid_t pid, int sig, const union sigval value);

```


参数 sigval 指定了伴随数据

```c
union sigval {
    int sigval_int;
    void *sigval_ptr;
}
```

### 处理实时信号

实时信号的处理和标准信号一样，可以使用常规单参数的信号处理函数，也可以使用带有3个参数的信号处理函数。  
采用了SA_SIGINFO标志后，传给信号处理函数的第二个参数将是一个siginfo_t，内含附加信息。其中si_value字段即sigqueue()中参数
`union sigval value`指定的伴随数据。

## 等待信号

### sigsuspend()

此系统调用将解除信号阻塞和挂起进程这个两个动作封装成一个操作，以防止在解除信号阻塞和挂起进程之间被信号打断，违背程序本意。

```c
#include <signal.h>
int sigsuspend(const sigset_t *mask);
```

`sigsuspend()`将以mask所指向的信号集来替换进程的信号掩码，然后挂起进程的执行，直到其捕捉到信号，并从信号处理函数中返回，一旦返回，sigsuspend()会将进程信号掩码恢复为调用前的值。

### 同步方式

```c
#define _POSIX_C_SOURCE 199309
#include <signal.h>

int sigwaitinfo(const sigset_t *set, siginfo_t *info);
```
sigwaitinfo()会挂起进程，直到收到set中的某一信号,返回值为信号编号，info参数如果不为空的话，则会包含额外信息。
使用sigwaitinfo()前应首先阻塞所有信号(即便信号阻塞，仍然可以使用sigwaitinfo()来获取等待信号)。

## 通过文件描述符获取信号

