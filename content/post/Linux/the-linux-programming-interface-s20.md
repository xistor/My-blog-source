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
//  Returns 0 on success, or –1 on error
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

前两种比较好理解，第三种SIGEV_THREAD，当定时器到期时，会启动一个新线程调用sigev_notify_function指定的函数来处理信号，sigev_value 保存了传入 sigev_notify_function 的参数。sigev_notify_attributes 如果非空，则应该是一个指向 pthread_attr_t 的指针，用来设置线程的属性（比如 stack 大小,detach 状态等）。  
SIGEV_THREAD_ID和SIGEV_SIGNAL类似，定时器到期将会向指定的sigev_notify_thread_id线程发送信号。  


### 设置定时器

```c
#define _POSIX_C_SOURCE 199309
#include <time.h>

int timer_settime(timer_t timerid, int flags, const struct itimerspec *value,
 struct itimerspec *old_value);
// Returns 0 on success, or –1 on error
```

timerid 是前一步timer_create时返回的那个，flag value和old_value分别上新旧定时器设置值。想要停止定时器，就把value.it_value设为0即可。  

### 删除定时器

```c
#define _POSIX_C_SOURCE 199309
#include <time.h>

int timer_delete(timer_t timerid);
// Returns 0 on success, or –1 on error
```


下面是一个使用timer的例子，通知方法使用SIGEV_THREAD：

```c
#include <stdio.h>  /* for puts() */
#include <string.h> /* for memset() */
#include <unistd.h> /* for sleep() */
#include <stdlib.h> /* for EXIT_SUCCESS */

#include <signal.h> /* for `struct sigevent` and SIGEV_THREAD */
#include <time.h>   /* for timer_create(), `struct itimerspec`,
                     * timer_t and CLOCK_REALTIME 
                     */

void thread_handler(union sigval sv) {
        char *s = sv.sival_ptr;

        /* Will print "5 seconds elapsed." */
        puts(s);
}

int main(void) {
        char info[] = "5 seconds elapsed.";
        timer_t timerid;
        struct sigevent sev;
        struct itimerspec trigger;

        /* Set all `sev` and `trigger` memory to 0 */
        memset(&sev, 0, sizeof(struct sigevent));
        memset(&trigger, 0, sizeof(struct itimerspec));

        /* 
         * Set the notification method as SIGEV_THREAD:
         *
         * Upon timer expiration, `sigev_notify_function` (thread_handler()),
         * will be invoked as if it were the start function of a new thread.
         *
         */
        sev.sigev_notify = SIGEV_THREAD;
        sev.sigev_notify_function = &thread_handler;
        sev.sigev_value.sival_ptr = &info;

        /* Create the timer. In this example, CLOCK_REALTIME is used as the
         * clock, meaning that we're using a system-wide real-time clock for
         * this timer.
         */
        timer_create(CLOCK_REALTIME, &sev, &timerid);

        /* Timer expiration will occur withing 5 seconds after being armed
         * by timer_settime().
         */
        trigger.it_value.tv_sec = 5;

        /* Arm the timer. No flags are set and no old_value will be retrieved.
         */
        timer_settime(timerid, 0, &trigger, NULL);

        /* Wait 10 seconds under the main thread. In 5 seconds (when the
         * timer expires), a message will be printed to the standard output
         * by the newly created notification thread.
         */
        sleep(10);

        /* Delete (destroy) the timer */
        timer_delete(timerid);

        return EXIT_SUCCESS;
}

```

### Timer Overruns

使用信号来获取定时器到期信息存在一个问题：标准信号在信号阻塞期间，多次到达也只会通知一次，无法知道定时器到期了几次，而实时信号会把信号队列化，我的理解是在信号不再阻塞后，这些阻塞序列中的定时器到期信号也就没用了，因为它们已经是过去式了。我们只需要知道在阻塞期间定时器到期了几次即可。可以使用下面两种方式获取到期次数：
- 调用 timer_getoverrun()
- 使用siginfo_t结构体中的si_overrun字段获取。  

overrun次数在每次收到计时器信号时会重置。

##  timerfd定时器

timerfd是Linux为用户程序提供的一个定时器接口。这个接口基于文件描述符，通过文件描述符的可读事件进行超时通知，一般是被用于select/poll/epoll的应用场景。

使用的API和前面的类似
```c
#include <sys/timerfd.h>
int timerfd_create(int clockid, int flags);
 
int timerfd_settime(int fd, int flags, const struct itimerspec *new_value, struct itimerspec *old_value);
 
int timerfd_gettime(int fd, struct itimerspec *curr_value);
```