---
title: "Linux/Unix系统编程手册-笔记24. 进程组、会话和作业控制"
date: 2020-10-31T17:40:00+08:00
tags: ["Linux"]
categories: ["Linux系统编程手册阅读"]
---

进程组是一组相关进程的集合，会话是一组相关进程组的集合。  
进程组和会话是用来支持shell作业控制的，创建进程组的目的是方便管理，可以向一个进程组内的进程同时发送信号。

## 进程组

进程组ID是创建进程组的进程的PID,新进程会继承其父进程所属的进程组ID。

可以使用`setpgid()`设置进程组id

```c
#include <unistd.h>

int setpgid(pid_t pid, pid_t pgid);
```
使用`setpgid()`的限制：
- 参数pid仅可以指定为调用进程或其子进程，否则报错ESRCH
- 在组间移动进程时，调用进程、由pid指定的进程、以及目标进程组必须在同属于一个会话，否则报错EPERM
- pid参数所指定的不能是会话首进程（可能原因：？会话首进程总是一个进程组首进程，使用`setsid()`在创建新会话的同时总是会创建一个新的进程组，若它被移动到其他一个进程组，那么它将不再是一个进程组组长，和规则冲突）
- 一个进程在其子进程已经执行了exec()后就无法修改该子进程的进程组ID了。违反了这条规则会导致EACESS。
正因为此，shell在为子进程创建新的进程组的时候，父进程和子进程都需要调用setpgid()，因为不能确定父进程和子进程的调度顺序，若不这样做将导致在进程组创建之前有一个未知的时间窗口，无法准确知道进程组何时创建。

## 会话

会话是一组进程组集合。

```c
#include <unistd.h>

pid_t setsid(void);

// 返回新会话的ID, -1为出错
```

setsid()创建一个新会话，调用进程成为新会话的首进程和该会话中新进程组的首进程。所有之前的控制终端的连接都会断开。一个和会话可以有一个控制终端，建立与控制终端连接的会话首进程被称为控制进程。 

在一个会话中，在同一时刻只有一个进程组能成为前台进程组。  

终端在断开的时候，内核会向控制进程发送SIGHUP信号，通常控制进程是shell进程它会转发给前台进程组的所有成员。


在shell正常退出时，也会发送SIGHUP信号，这种情况经常遇到，比如在ssh登录到远程服务器执行长耗时的任务时，ssh终端退出后，远程任务也被终止了。
这种情况可以使用nohup命令可以使一个命令对SIGHUP免疫，在终端退出后，依然可以继续执行。  


## 作业控制
输入命令以 `&` 结尾，该命令会转入后台运行。  

```sh
sleep 60 &
```

`jobs`命令能查看当前后台运行的作业，方括号内为作业号,`+`标记当前作业，`-`标记上一个当前作业。(当前作业为在前台为最先被停止的作业)

```sh
$ jobs
[1]-  Running                 sleep 60 &
[2]+  Running                 sleep 120 &
```

`fg`将后台作业移动到前台，`%num`代表作业号。

```sh
$ fg %1
```

当作业在前台运行时，可以使用（ctrl+z）挂起作业，如需要，可以使用`fg`或`bg`在前台或后台恢复作业。  

只有前台作业的进程才能从控制终端读取输入，如果后台作业尝试从控制终端中读取输入，就会收到一个SIGTTIN信号。（SIGTTIN信号的默认操作是停止作业）
默认情况下（终端未设置TOSTOP标记），后台作业是可以通过终端输出内容的。


书中给了一段处理作业控制信号的例子，关于SIGTSTP的信号处理函数部分在下面，来看他是怎么做的，首先在（1）处将SIGTSTP信号的处置方式设为默认，(2)重新发送SIGTSTP，因为进入了信号处理函数会阻塞SIGTSTP（除非指定了SA_NODEFER标记），所以在（3）处解除阻塞信号后，会收到（2）处发出的SIGTSTP，进程会被挂起。在收到SIGCONT信号后（如使用fg调进程至前台），程序从（4）处继续执行，又将SIGTSTP信号阻塞，然后在（5）处重新注册本身。  
看完流程，有两个疑问，（4）处将信号重新阻塞，那么之后还能收到SIGTSTP信号吗，又是在那里解除阻塞的呢？别忘了，当前这段程序是在信号处理函数内，在信号处理函数内部的信号阻塞和解除阻塞都只在内部有效，所以跳出信号处理函数后，信号mask就会被恢复原状。另一个问题是为什么需要重新注册本身为信号处理函数呢？之前好像没有看到需要每次注册呀，这个是因为（1）处把SIGTSTP信号的处置方式设为默认了，所以需要在退出信号处理函数之前重新注册自身。


```c

static void                             /* Handler for SIGTSTP */
tstpHandler(int sig)
{
    sigset_t tstpMask, prevMask;
    int savedErrno;
    struct sigaction sa;

    savedErrno = errno;                 /* In case we change 'errno' here */

    printf("Caught SIGTSTP\n");         /* UNSAFE (see Section 21.1.2) */

    if (signal(SIGTSTP, SIG_DFL) == SIG_ERR) //（1）
        errExit("signal");              /* Set handling to default */

    raise(SIGTSTP);     // (2)                    /* Generate a further SIGTSTP */

    /* Unblock SIGTSTP; the pending SIGTSTP immediately suspends the program */

    sigemptyset(&tstpMask);
    sigaddset(&tstpMask, SIGTSTP);
    if (sigprocmask(SIG_UNBLOCK, &tstpMask, &prevMask) == -1)   // （3）
        errExit("sigprocmask");

    /* Execution resumes here after SIGCONT */

    printf("continue handler\n");   // (4)
    if (sigprocmask(SIG_SETMASK, &prevMask, NULL) == -1)
        errExit("sigprocmask");         /* Reblock SIGTSTP */

    sigemptyset(&sa.sa_mask);           /* Reestablish handler */
    sa.sa_flags = SA_RESTART;
    sa.sa_handler = tstpHandler;
    if (sigaction(SIGTSTP, &sa, NULL) == -1) // （5）
        errExit("sigaction");

    printf("Exiting SIGTSTP handler\n");
    errno = savedErrno;
}

```

