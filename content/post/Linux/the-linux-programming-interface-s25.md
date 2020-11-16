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

SUSv3提供了两种实时调度策略SCHED_RR和SCHED_FIFO。处于这两种策略的进程优先级总高于其他处于标准策略的进程。每种调度策略内又有不同的优先级，Linux提供了99个实时优先级（1（低）~99（高）），