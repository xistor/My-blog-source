---
title: "Linux/Unix系统编程手册-笔记22. 程序的执行"
date: 2020-10-12T23:30:00+08:00
tags: ["Linux"]
categories: ["Linux系统编程手册阅读"]
---

## exec()

系统调用execve()将新程序加载到某一进程的内存空间。

```c
#include <unistd.h>

int execve(const char *pathname, char *const argv[], char *const envp[]);
```

- pathname 为指定程序的路径名
- argv为传递给新进程的命令行参数
- envp指定了新程序的环境列表

除此之外还提供了一系列构建于execve()之上的库函数。

```cpp
#include <unistd.h>

int execle(const char * pathname , const char * arg , ...
    /* , (char *) NULL, char *const envp [] */ );
int execlp(const char * filename , const char * arg , ...
    /* , (char *) NULL */);
int execvp(const char * filename , char *const argv []);
int execv(const char * pathname , char *const argv []);
int execl(const char * pathname , const char * arg , ...
    /* , (char *) NULL */);

// None of the above returns on success; all return –1 on error
```

从命名上可以看出规律
- 后缀有`l`的表示以列表的形式指定参数
- 后缀有`e`的表示参数将指定环境变量
- 后缀有`v`的表示以数组的形式指定参数
- 后缀有`p`的表示允许只提供文件名，系统将会在系统变量PATH指定的路径下寻找指定的可执行文件

## exec() 与信号

exec()在执行时会将现有进程已设的信号处置重置为SIG_DFL。而对所有其他信号（即将处置置为SIG_IGN或SIG_DFL的信号）的处置则保持不变。不过遭忽略的SIGCHLD信号属于特例，Linux在执行exec()后将保持其忽略状态。而其他一些UNIX实现将其重置为SIG_DFL。

## system()

system()使用shell来执行命令，使用system()运行命令需要创建至少两个进程。一个用于运行shell，另外一个或多个则用于shell所执行的命令。  
在set-user-ID和set-group-ID程序中避免使用system(),shell对操作的控制依赖于各种环境变量，所以使用system()会不可避免的给系统带来安全隐患。  

system()实现：  

首先需要使用fork()来创建一个子进程，并调用execl()执行sh命令

```c
execl("/bin/sh", "sh", "-c", command, (char *) NULL)
```

system()实现的时候需要考虑对信号的正确处理，首先是SIGCHLD,若调用system()的程序还直接创建了其他子进程，对SIGCHLD信号处理函数也执行了wait()，那么就会和system()中的waitpid()形成竞争关系。所以system()在运行期间必须阻塞SIGCHLD信号。
只有父进程才需要阻塞SIGCHLD，同时还需要忽略SIGINT和SIGQUIT。  
下图所有进程构成前端进程组的一部分，在输入中断或退出符时，会将相应信号发送给所有3个进程，shell在等待子进程期间会忽略SIGINT和SIGQUIT信号，默认情况下会杀死调用程序与sleep进程。

{{< figure src="/img/the-linux-programming-interface-s21/overview.png" title="执行system(\"sleep 20\")期间的进程情况" class="center"  width="500">}}



对于哪个进程应该收到SIGINT和SIGQUIT，SUSv3规定：  
- 调用进程在执行命令期间应忽略SIGINT和SIGQUIT信号
- 子进程将对上述两信号的处置重置为默认值，而对其他信号的处置则保持不变。
遵守这样的规定后，表现出来的现象是在system()执行时，使用ctrl+c或ctrl+\将只会杀掉system()的子进程，应用程序将会继续运行。

## clone()

```c
#define _GNU_SOURCE
#include <sched.h>

int clone(int (*func)(void *), void *child_stack, int flags, void *func_arg, ...
                 /* pid_t *ptid, struct user_desc *tls, pid_t *ctid */ );

// return process ID of child on success, or -1 on error
```

clone()和fork()的不同之处
- clone()生成的子进程继续运行时不以调用处为起点，转而去调用以参数func所指定的函数。func的参数由func_arg指定。
- 子程序可能和父进程共享内存，所以调用者需要分配一块适当大小的栈供子进程使用, 栈向下增长，child_stack指向内存块的高端。
- clone()的参数flags的低字节存放着子进程的终止信号。可以指定子进程终止时父进程收到的信号（fork只能是SIGCHLD。
- flags剩余字节存放了掩码，用于控制clone()的操作，比如共享文件描述符、共享对信号的处置等。
