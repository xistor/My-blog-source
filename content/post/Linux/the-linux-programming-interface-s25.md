---
title: "Linux/Unix系统编程手册-笔记25. 进程优先级和进程调度"
date: 2020-11-05T18:30:00+08:00
tags: ["Linux"]
categories: ["Linux系统编程手册阅读"]
---

## nice值

nice值是进程的一个属性，可以间接影响内核调度，其值的范围为-20到+19，默认值为0。只有特权用户可以设置更高的优先级，其他用户只能设置更低的优先级。  

调用fork()后，子进程会继承其父进程的nice值，exec()后，nice值会保留。


```c
#include <sys/resource.h>
int getpriority(int which, id_t who);
// Returns (possibly negative) nice value of specified process on success, or –1 on error
int setpriority(int which, id_t who, int prio);
// Returns 0 on success, or –1 on error
```

参数who代表取或设置的对象，which参数指定如何理解who,由which的不同，who可以理解为进程号、进程组号或用户id。  
`getpriority()`系统调用返回值范围实际为1-40(通过20-nice转换),为了避免返回负值，因为负值在系统调用中通常代表出错。我们实际调用的`getpriority()`函数经过了c函数库的包装，返回值为-20到+19。正因为返回负值是合法的所以判断函数是否出错，通常需要像下面这样：

```c
    errno = 0; /* Because successful call may return -1 */
    prio = getpriority(which, who);
    if (prio == -1 && errno != 0)
        errExit("getpriority");
```

`setpriorty()`参数中`prio`的范围为-20到+19，就算给的值不在这个范围内，给的值大于19的也会被置为19，小于-20的会被置为-20。


## 实时进程调度

实时进程对调度有更严格的要求，须满足：
- 在指定时间内必须对外部输入做出响应。
- 一个高优先级进程可以保持对CPU的独占，直到它运行结束或者自愿放弃CPU。
- 实时进程应该能够精确的控制其组件进程的调度顺序。

SUSv3提供了两种实时调度策略SCHED_RR和SCHED_FIFO。处于这两种策略的进程优先级总高于其他处于标准策略(SCHED_OTHER)的进程。每种调度策略内又有不同的优先级，Linux提供了99个实时优先级（1（低）~99（高）），不同调度策略里的相同优先级的进程是平等的。

### SCHED_RR和SCHED_FIFO策略

这两种策略比较相似，区别在于SCHED_RR策略中，同优先级的进程调度采用时间片轮转，而SCHED_FIFO中，当一个进程获得了CPU,除非它自己释放或终止，或者被I/O阻塞，或者被其他高优先级进程抢占，否则将一直占有CPU。

SCHED_RR策略又和普通的进程调度类似，但是实时调度策略的区别在于，高优先级的进程总是会抢占CPU。其时间片轮转也仅仅是在同优先级的进程内。
### SCHED_BATCH 和 SCHED_IDLE策略

这两个并非实时调度策略，和SCHED_OTHER类似。

- SCHED_BATCH：适用于非交互、CPU密集型的批处理进程，此策略下，时间片更长(默认1.5s)，可以更好的利用缓存。nice值决定了进程（非kernel-mode）相对的CPU占用时间百分比。
- SCHED_IDLE：优先级最低（比+19还低），nice值对这些进程来说也没有意义，只有在系统空闲的时候才会分到CPU。


## 进程调度相关API


```c
#include <sched.h>
int sched_setscheduler(pid_t pid, int policy, const struct sched_param *param);
//  Returns 0 on success, or –1 on error

int sched_setparam(pid_t pid, const struct sched_param *param);
// Returns 0 on success, or –1 on error


struct sched_param {
 int sched_priority; /* Scheduling priority */
};
```

`sched_setscheduler()`能同时改变调度策略和进程优先级。`sched_setparam()`只改变进程优先级。  

修改进程nice值需要合适的权限，特权进程(CAP_SYS_NICE)可以设置调度策略优先级，或者EUID相同可以设置SCHED_OTHER策略。Linux2.6.12之后，资源限制RLIMIT_RTPRIO定义了进程修改优先级的上限，若RLIMIT_RTPRIO非0，则优先级不能高于当前优先级且不能高于RLIMIT_RTPRIO，RLIMIT_RTPRIO为0，进程只能降低其实时调度优先级，或者切换到非实时调度策略。  

```c
#include <sched.h>
int sched_yield(void);
```

`sched_yield()`可以主动放弃CPU,若在同一优先级队列内没有其他运行的进程，那么进程就当什么都没发生，继续运行，否则将被放到队列最后。这只适用于处于实时调度策略的进程。

```c
#include <sched.h>
int sched_rr_get_interval(pid_t pid, struct timespec *tp);
// Returns 0 on success, or –1 on error
```

`sched_rr_get_interval()`能获取一个SCHED_RR进程的时间片大小，Linux 3.9之后写`/proc/sys/kernel/sched_rr_timeslice_ms`可以修改。


## CPU Affinity

