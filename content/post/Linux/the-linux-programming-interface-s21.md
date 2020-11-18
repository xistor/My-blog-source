---
title: "Linux/Unix系统编程手册-笔记21. 进程的创建、终止"
date: 2020-09-25T18:21:00+08:00
tags: ["Linux"]
categories: ["Linux系统编程手册阅读"]
---

进程这部分主要有几个点：进程的创建、进程的终止、父进程和子进程之间的关系等。开局先来一张图

![overview](/img/the-linux-programming-interface-s21/overview.png)

图中把fork()、exec()、wait()、exit()的功能展示的很清楚了，有几个点需要注意:

## fork()

```c
#include <unistd.h>
pid_t fork(void);
/* In parent: returns process ID of child on success, or –1 on error;
in successfully created child: always returns 0*/

// 常用的使用模板如下：

pid_t childPid; /* Used in parent after successful fork() 
to record PID of child */
switch (childPid = fork()) {
case -1: /* fork() failed */
 /* Handle error */
case 0: /* Child of successful fork() comes here */
 /* Perform actions specific to child */
default: /* Parent comes here after successful fork() */
 /* Perform actions specific to parent */
}
```

fork的时候会把父进程的内存复制一份给子进程，两个进程从fork()调用处继续运行。为了避免内存浪费，在复制进程内存的时候还会采取两种措施：1.共享代码段。2.对于数据段、堆、栈则采用copy-on-write方式。  
而且由于内存复制，子进程拥有父进程的文件描述符副本，指向同一个打开文件句柄。所以共享打开文件的offset等信息。  
其次还需要考虑竞态，毕竟两个进程同时从fork()之后运行，为避免竞争，可以使用信号来同步。

## _exit()和exit()

```c
#include <unistd.h>
void _exit(int status);


#include <stdlib.h>
void exit(int status);

// status只有低8位有用
```

第一个是系统调用，第二个库函数，区别在于exit()会多做一些善后操作
1. 调用exit handler
2. flush stdio buffer
3. 最后调用 _exit()

在进程退出时可以注册exit handler（可以注册多个，调用顺序与注册顺序相反）做一些清理操作。有以下两种方式：

```c
#include <stdlib.h>
int atexit(void (*func)(void));


#define _BSD_SOURCE /* Or: #define _SVID_SOURCE */
#include <stdlib.h>
int on_exit(void (*func)(int, void *), void *arg);
```

第二个为库函数，相比atexit(), on_exit()通过func的第一个参数可以获得进程退出的status,其次arg也会被传入func,可以灵活指定参数。


## wait()

```c
#include <sys/wait.h>
pid_t wait(int *status);
// Returns process ID of terminated child, or –1 on error
```

wait()函数由父进程调用，会阻塞至有一个子进程终止。如果想等待所有子进程终止，可以像下面这样：

```c
while ((childPid = wait(NULL)) != -1)
    continue;

// 当没有子进程需要等待的时候，errno=ECHILD
if (errno != ECHILD) /* An unexpected error... */
    errExit("wait");
```

需要等待特定子进程的，用waitpid()就行。

```c
#include <sys/wait.h>
pid_t waitpid(pid_t pid, int *status, int options);
// Returns process ID of child, 0 (see text), or –1 on error
```

pid的取值和信号那一章的kill()的参数类似：
1. 如果pid大于0，会等待对应pid的进程。
2. pid = 0, 会等待同一个进程组的任一进程。
3. pid < -1, 会取绝对值。
4. pid = -1, 等待任一子进程，和wait()一样了。  

options可以指定flags，在子进程被信号SIGSTOP或SIGCONT，终止或继续时返回状态。也可以不阻塞等待子进程，采用轮询的方式。  

还有一个名字很容易搞混的系统调用waitid(),

```c
#include <sys/wait.h>
int waitid(idtype_t idtype, id_t id, siginfo_t *infop, int options);
/* Returns 0 on success or if WNOHANG was specified and
there were no children to wait for, or –1 on error */
```

参数idtype和id用来指定等待的子进程。
1. idtype为P_ALL，等待任意一个子进程，id被忽略
2. idtype为P_PID，等待PID为id的子进程。
3. idtype为p_PGID,等待任意一个进程组id为id的子进程。

waitid()的options参数可使用值更丰富些：
1. WEXITED： 等待子进程终止
2. WSTOPPED： 等待子进程被信号stop
3. WCONTINUED: 等待子进程被信号SIGCONT继续
4. WNOHANG： 如果符合参数指定id的子进程没有需要返回的状态信息,则立刻返回。如果没有符合指定id的子进程，则error为ECHLD。
5. WNOWAIT： 如果设置了此flag, 子进程状态返回后，子进程依然在可等待状态中，可以再次取得同样的进程信息。

## 孤儿进程和僵尸进程

- 当父进程先于子进程终止，子进程就变为孤儿进程，孤儿进程会被init进程收养，调用getppid()会返回init的pid 1。
- 子进程在父进程wait()之前就终止了，内核会把子进程转为僵尸进程，子进程所持有的大多数资源已经还给了系统，只在内核的进程表中保留了记录。  

## SIGCHLD

为了不阻塞父进程，或者浪费CPU去轮询子进程状态，可以为SIGCHLD信号创建信号处理函数，当有子进程终止的时候，父进程会收到SIGCHLD信号，在信号处理函数中调用wait()收割僵尸子进程。由于在信号处理函数内会阻塞信号，所以多个子进程在同一时间终止会导致父进程只收到一个SIGCHLD信号，为解决这种情况可以循环检测是否有其他僵尸子进程：

```cpp
while (waitpid(-1, NULL, WNOHANG) > 0)
    continue;
```


父进程将SIGCHLD设置为SIG_IGN,会使子进程立即终止，而不是转为僵尸进程。类似的在sigaction()中使用SA_NOCLDWAIT flag也可以实现同样的效果。

## Exercises

2. 在父进程终止之后。

4. make_zombie.c 使用信号替代sleep()

```c
#include <signal.h>
#include <libgen.h>             /* For basename() declaration */
#include "tlpi_hdr.h"

#define CMD_SIZE 200
#define SYNC_SIG SIGUSR1                /* Synchronization signal */

static void             /* Signal handler - does nothing but return */
handler(int sig)
{
}

int
main(int argc, char *argv[])
{
    char cmd[CMD_SIZE];
    pid_t childPid;
    sigset_t blockMask, origMask, emptyMask;
    struct sigaction sa;

    setbuf(stdout, NULL);       /* Disable buffering of stdout */

    sigemptyset(&blockMask);
    sigaddset(&blockMask, SYNC_SIG);    /* Block signal */
    if (sigprocmask(SIG_BLOCK, &blockMask, &origMask) == -1)
        errExit("sigprocmask");

    sigemptyset(&sa.sa_mask);
    sa.sa_flags = SA_RESTART;
    sa.sa_handler = handler;
    if (sigaction(SYNC_SIG, &sa, NULL) == -1)
        errExit("sigaction");

    printf("Parent PID=%ld\n", (long) getpid());

    switch (childPid = fork()) {
    case -1:
        errExit("fork");

    case 0:     /* Child: immediately exits to become zombie */
        printf("Child (PID=%ld) exiting!!\n", (long) getpid());

        sleep(3);
        if (kill(getppid(), SYNC_SIG) == -1)
            errExit("kill");

        _exit(EXIT_SUCCESS);

    default:    /* Parent */

        sigemptyset(&emptyMask);
        if (sigsuspend(&emptyMask) == -1 && errno != EINTR)
            errExit("sigsuspend");

        printf("got signal\n");

        if (sigprocmask(SIG_SETMASK, &origMask, NULL) == -1)
            errExit("sigprocmask");

        snprintf(cmd, CMD_SIZE, "ps | grep %s", basename(argv[0]));
        system(cmd);            /* View zombie child */

        /* Now send the "sure kill" signal to the zombie */

        if (kill(childPid, SIGKILL) == -1)
            errMsg("kill");
        sleep(3);               /* Give child a chance to react to signal */
        printf("After sending SIGKILL to zombie (PID=%ld):\n", (long) childPid);
        system(cmd);            /* View zombie child again */

        exit(EXIT_SUCCESS);
    }
}
```