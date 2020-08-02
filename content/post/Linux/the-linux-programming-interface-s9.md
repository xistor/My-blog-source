---
title: "Linux/Unix系统编程手册-笔记9.时间"
date: 2020-07-30T10:53:00+08:00
tags: ["Linux"]
categories: ["Linux系统编程手册阅读"]
---


直接看书上的例子吧：获取时间并转换成多种形式显示。加了些注释。

```cpp
#include <locale.h>
#include <time.h>
#include <sys/time.h>
#include "tlpi_hdr.h"

#define SECONDS_IN_TROPICAL_YEAR (365.24219 * 24 * 60 * 60)

int
main(int argc, char *argv[])
{
    time_t t;
    struct tm *gmp, *locp;
    struct tm gm, loc;
    struct timeval tv;

    /* Retrieve time, convert and display it in various forms */

    /*
        time()函数返回一个time_t类型的值，是一个有符号整数，值代表当前时间离1970.1.1零点过去的秒数
    */
    t = time(NULL);
    printf("Seconds since the Epoch (1 Jan 1970): %ld", (long) t);
    printf(" (about %6.3f years)\n", t / SECONDS_IN_TROPICAL_YEAR);

    /* 
    timeval 结构体
      struct timeval {
          time_t tv_sec;            // 从1970.1.1 零点开始的秒数
          suseconds _t tv_usec;     // 附加的毫秒 (long int)
      }
    */
    if (gettimeofday(&tv, NULL) == -1)
        errExit("gettimeofday");
    printf("  gettimeofday() returned %ld secs, %ld microsecs\n",
            (long) tv.tv_sec, (long) tv.tv_usec);

    /*
    gmp是一个指向tm结构体的指针，
        struct tm {
            int tm_sec;
            int tm_min;
            int tm_hour;
            int tm_mday;
            int tm_mon;
            int tm_year;
            int tm_wday;    // 一周内的第几天（星期天=0）
            int tm_yday;    // 一年内的第几天
            int tm_isdst;   // > 0 : 夏令时
                               = 0 : 非夏令时
                               < 0 : 尝试判定DTS在每年的这一时间是否生效
        }

        gmtime()函数能够把日历时间转换为一个对应于UTC的分解时间，也就是上面的结构体

    */
    gmp = gmtime(&t);
    if (gmp == NULL)
        errExit("gmtime");

    gm = *gmp;          /* Save local copy, since *gmp may be modified
                           by asctime() or gmtime() */
    printf("Broken down by gmtime():\n");
    printf("  year=%d mon=%d mday=%d hour=%d min=%d sec=%d ", gm.tm_year,
            gm.tm_mon, gm.tm_mday, gm.tm_hour, gm.tm_min, gm.tm_sec);
    printf("wday=%d yday=%d isdst=%d\n", gm.tm_wday, gm.tm_yday, gm.tm_isdst);

    /* The TZ environment variable will affect localtime().
       Try, for example:

                TZ=Pacific/Auckland calendar_time
    */

    /* localtime()函数会考虑时区和夏令时设置，其他和gmtime()一致 */
    locp = localtime(&t);
    if (locp == NULL)
        errExit("localtime");

    loc = *locp;        /* Save local copy */

    printf("Broken down by localtime():\n");
    printf("  year=%d mon=%d mday=%d hour=%d min=%d sec=%d ",
            loc.tm_year, loc.tm_mon, loc.tm_mday,
            loc.tm_hour, loc.tm_min, loc.tm_sec);
    printf("wday=%d yday=%d isdst=%d\n\n",
            loc.tm_wday, loc.tm_yday, loc.tm_isdst);

    /* asctime()函数 将分解后的时间格式化为字符串,参数为一个tm结构体 */
    printf("asctime() formats the gmtime() value as: %s", asctime(&gm));
    /* ctime()函数 将time_t转换为可打印格式 */
    printf("ctime() formats the time() value as:     %s", ctime(&t));

    /* mktime()函数 将一个本地时区的分解时间解释为time_t值 */
    printf("mktime() of gmtime() value:    %ld secs\n", (long) mktime(&gm));
    printf("mktime() of localtime() value: %ld secs\n", (long) mktime(&loc));

    exit(EXIT_SUCCESS);
}

```


### 时区  

时区信息保存在文件中，位于/usr/share/zoneinfo中。为运行中的程序指定一个时区，需要设置TZ环境变量。设置方法：

```sh
TZ=":Pacific/Aukland" ./show_time
```

### 软件时钟 (jiffies)

在本书中所描述的时间相关的各种系统调用的精度是受限于系统软件时钟的分辨率，他的度量单位被称为jiffies。jiffies的大小是定义在内核源代码的常量HZ。这是内核按照round-robin的分时调度算法分配CPU进程的单位。  

更高的软件时钟速率意味着定时器可以有更高的操作精度和时间精度，然而因为每个时钟中断会消耗少量的CPU时间，，这部分时间CPU无法执行任何操作。  

最终，软件时钟频率成为一个可配置的内核选项。内核可以设置为100、250（默认）、1000、300。

### 进程时间

进程时间是进程创建后使用的CPU时间数量。内核把CPU时间分以下两部分：
- 用户CPU时间：在用户模式下执行所花费的时间数量
- 系统CPU时间：在内核模式中执行所花费的时间数量。
有时候，进程时间是指出现处理过程中所消耗的总CPU时间。


还是直接看例程：

```cpp

#include <sys/times.h>
#include <time.h>
#include "tlpi_hdr.h"

static void             /* Display 'msg' and process times */
displayProcessTimes(const char *msg)
{
    struct tms t;
    clock_t clockTime;
    static long clockTicks = 0;

    if (msg != NULL)
        printf("%s", msg);

    if (clockTicks == 0) {      /* 获取每秒包含的时钟计时单元 */
        clockTicks = sysconf(_SC_CLK_TCK);
        if (clockTicks == -1)
            errExit("sysconf");
    }

    /*
        clock()返回调用进程使用的总的CPU时间，包括用户和系统。
    */
    clockTime = clock();
    if (clockTime == -1)
        errExit("clock");

    printf("        clock() returns: %ld clocks-per-sec (%.2f secs)\n",
            (long) clockTime, (double) clockTime / CLOCKS_PER_SEC);

    /* times()返回一个tms结构体
        struct tms {
            clock_t tms_utime;  // 调用者使用的用户CPU时间
            clock_t tms_stime;  // 调用者使用的系统CPU时间
            clock_t tms_cutime; // 调用者执行了系统调用wait()的所有已经终止的子进程使用的用户CPU时间
            clock_t tms_cstime; // 调用者执行了系统调用wait()的所有已经终止的子进程使用的系统CPU时间
        }

        
    */
    if (times(&t) == -1)
        errExit("times");
    printf("        times() yields: user CPU=%.2f; system CPU: %.2f\n",
            (double) t.tms_utime / clockTicks,
            (double) t.tms_stime / clockTicks);
}

int
main(int argc, char *argv[])
{
    int numCalls, j;

    printf("CLOCKS_PER_SEC=%ld  sysconf(_SC_CLK_TCK)=%ld\n\n",
            (long) CLOCKS_PER_SEC, sysconf(_SC_CLK_TCK));

    displayProcessTimes("At program start:\n");

    /* Call getppid() a large number of times, so that
       some user and system CPU time are consumed */

    numCalls = (argc > 1) ? getInt(argv[1], GN_GT_0, "num-calls") : 100000000;
    for (j = 0; j < numCalls; j++)
        (void) getppid();

    displayProcessTimes("After getppid() loop:\n");

    exit(EXIT_SUCCESS);
}

```
