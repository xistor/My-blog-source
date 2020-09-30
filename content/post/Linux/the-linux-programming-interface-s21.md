---
title: "Linux/Unix系统编程手册-笔记21. 进程"
date: 2020-09-25T18:21:00+08:00
tags: ["Linux"]
categories: ["Linux系统编程手册阅读"]
---

进程这部分主要有几个点：进程的创建、进程的终止、父进程和子进程之间的关系等。开局先来一张图

![overview](/img/the-linux-programming-interface-s21/overview.png)

图中把fork()、exec()、wait()、exit()的功能展示的很清楚了，有几个点需要注意:

- fork(): 
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

- _exit()和exit()

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


- wait()

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
Returns process ID of child, 0 (see text), or –1 on error
```

pid的取值和信号那一章的kill()的参数类似：
1. 如果pid大于0，会等待对应pid的进程。
2. pid = 0, 会等待同一个进程组的任一进程。
3. pid < -1, 会取绝对值。
4. pid = -1, 等待任一子进程，和wait()一样了。  

options可以指定flags，在子进程被信号SIGSTOP或SIGCONT，终止或继续时返回状态。也可以不阻塞等待子进程，采用轮询的方式。  

还有一个名字很像的系统调用waitid(),

```c
#include <sys/wait.h>
int waitid(idtype_t idtype, id_t id, siginfo_t *infop, int options);
/* Returns 0 on success or if WNOHANG was specified and
there were no children to wait for, or –1 on error */
```