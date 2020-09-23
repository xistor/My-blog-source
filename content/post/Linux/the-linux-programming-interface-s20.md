---
title: "Linux/Unix系统编程手册-笔记20.定时器与休眠"
date: 2020-09-02T18:21:00+08:00
tags: ["Linux"]
categories: ["Linux系统编程手册阅读"]
---



## 间隔计时器
使用setitimer()，一个进程可以创建三种不同类型的定时器。
```c
#include <sys/time.h>
int setitimer(int which, const struct itimerval *new_value,struct itimerval *old_value);
                    // Returns 0 on success, or –1 on error
```

参数`which`指定不同的timer类型：
- ITIMER_REAL：计时器以真实时间计时
- ITIMER_VIRTUAL： 计时器以进程使用的用户态时间计时
- ITIMER_PROF： 计时器以进程使用的用户态和内核态时间计时

`itimerval`结构体如下：

```c
struct itimerval {
    struct timeval it_interval; /* Interval for periodic timer */
    struct timeval it_value; /* Current value (time until next expiration) */
};

struct timeval {
 time_t tv_sec; /* Seconds */
 suseconds_t tv_usec; /* Microseconds (long int) */
};
```

`new_value`中`it_value`指定计时器到期的延时，`it_interval`指定当前计时器是否是周期计时器，如果`it_interval`都为0则计时器只发生一次，否则就是一个周期计时器。  
一个进程只能有三种计时器中的一种，第二次调用`setitimer()`会改变现有的计时器。  
若`old_value`非`NULL`,那么它将返回之前一个计时器的的信息。  

计时器时间到了，进程会收到SIGALRM信号，在信号处理函数中做想做的操作。

##  POSIX Interval Timers

前面提到的经典的UNIX间隔计时器，存在几个限制：
- 只能设置三种类型
- 获得定时时间到的通知的唯一方法就是接收信号。
- 如果信号被屏蔽了，而这之间产生了几次定时器到期，在信号解除屏蔽后，只会收到一次通知
- 精度只到微秒级

使用POSIX定时器分三步：
1. 使用`timer_create()`创建定时器。
2. 使用`timer_settime()`来开始或停止一个定时器。
3. 使用`timer_delete()`删除定时器。

### 创建定时器

```c
#define _POSIX_C_SOURCE 199309
#include <signal.h>
#include <time.h>
int timer_create(clockid_t clockid, struct sigevent *evp, timer_t *timerid);
```

clockid的取值如下表

|Clock ID| Description|
|---------------|-------|
|CLOCK_REALTIME|系统级的实时时钟，可以通过改变系统时间而改变|
|CLOCK_MONOTONIC|系统启动后到现在的时间|
|CLOCK_PROCESS_CPUTIME_ID|进程消耗的用户和系统CPU时间|
|CLOCK_THREAD_CPUTIME_ID|线程消耗的CPU时间|

sigevent这个结构体还挺TM复杂，描述如下：

```cpp
union sigval {
 int sival_int; /* Integer value for accompanying data */
 void *sival_ptr; /* Pointer value for accompanying data */
};
struct sigevent {
    int sigev_notify; /* Notification method */
    int sigev_signo; /* Timer expiration signal */
    union sigval sigev_value; /* Value accompanying signal or
    passed to thread function */
    union {
        pid_t _tid; /* ID of thread to be signaled */
        struct {
        void (*_function) (union sigval);
                        /* Thread notification function */
        void *_attribute; /* Really 'pthread_attr_t *' */
        } _sigev_thread;
    } _sigev_un;
};
#define sigev_notify_function _sigev_un._sigev_thread._function
#define sigev_notify_attributes _sigev_un._sigev_thread._attribute
#define sigev_notify_thread_id _sigev_un._tid
```

sigev_notify指定通知方法，
|sigev_notify value| Notification method|
|---------------|-------|
|SIGEV_NONE|No notification; monitor timer using timer_gettime()|
|SIGEV_SIGNAL|Send signal sigev_signo to process|
|SIGEV_THREAD|Call sigev_notify_function as start function of new thread|
|SIGEV_THREAD_ID|Send signal sigev_signo to thread sigev_notify_thread_id|

前两种比较好理解，第三种SIGEV_THREAD