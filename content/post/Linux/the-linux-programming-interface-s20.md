---
title: "Linux/Unix系统编程手册-笔记20.定时器与休眠"
date: 2020-09-02T18:21:00+08:00
tags: ["Linux"]
categories: ["Linux系统编程手册阅读"]
---



## 间隔计时器
使用setitimer()
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
