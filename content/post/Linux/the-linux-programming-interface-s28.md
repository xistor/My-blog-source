---
title: "Linux/Unix系统编程手册-笔记28. 管道"
date: 2020-11-27T18:30:00+08:00
tags: ["Linux"]
categories: ["Linux系统编程手册阅读"]
---

## 管道的概念

管道一般用于在shell中将一个进程的输出作为另一个进程的输入。比如

```sh
$ ls | wc -l
```

管道是一个字节流，只能按顺序读取任意大小的数据，不能使用fseek()随机读取。  

从当前是空的管道内读取数据会阻塞，直到有数据写入管道，如果管道的写入端已关闭，读管道的进程读完管道内剩余的数据后将会读到end of file。  

管道的数据传输是单向的。  

往管道内写数据到PIPE_BUF个字节前肯定是原子操作。当写入数据大于PIPE_BUF时，内核会把数据分割成任意大小写入管道，当多个进程同时写入时，数据就可能混合。  

管道写满后（Linux 2.6.11 后管道大小为65536），`write()`会阻塞，直到`read()`从管道删除部分数据。  

管道可以让两个相关的进程进行通信，相关指的是两个进程有同一个祖先，因为管道的文件描述符是通过`pipe()`返回的，通过继承在各个进程中传递。一个普遍的场景是，父进程创建管道，之后兄弟进程利用管道通信。

## 创建和使用管道

下面是例子，使用`pipe()`创建管道后，调用`fork()`,在父进程中往管道写数据，在另一端的子进程从管道中读出数据。父进程往管道写完数据后，会关闭管道，子进程在循环读出的过程中碰到EOF就退出循环。

```cpp
int pfd[2];                             /* Pipe file descriptors */
char buf[BUF_SIZE];
ssize_t numRead;

if (pipe(pfd) == -1)                    /* Create the pipe， pfd[0] 读管道， pfd[1] 写管道*/
        errExit("pipe");

switch (fork()) {
    case -1:
        errExit("fork");

    case 0:             /* Child  - reads from pipe */
        if (close(pfd[1]) == -1)            /* Write end is unused */
            errExit("close - child");

        for (;;) {              /* Read data from pipe, echo on stdout */
            numRead = read(pfd[0], buf, BUF_SIZE);
            if (numRead == -1)
                errExit("read");
            if (numRead == 0)
                break;                      /* End-of-file */
            if (write(STDOUT_FILENO, buf, numRead) != numRead)
                fatal("child - partial/failed write");
        }

        write(STDOUT_FILENO, "\n", 1);
        if (close(pfd[0]) == -1)
            errExit("close");
        _exit(EXIT_SUCCESS);

    default:            /* Parent - writes to pipe */
        if (close(pfd[0]) == -1)            /* Read end is unused */
            errExit("close - parent");

        if (write(pfd[1], "hello", 5) != 5)
            fatal("parent - partial/failed write");

        if (close(pfd[1]) == -1)            /* Child will see EOF */
            errExit("close");
        wait(NULL);                         /* Wait for child to finish */
        exit(EXIT_SUCCESS);
    }
```

实现shell中的连接符，比如`ls | wc -l`这样，主要操作就是将标准输入输出冲定向到管道，使用`dup2()`复制文件描述符实现，然后在两端分别关闭重复的文件描述符。

```c
if (pfd[1] != STDOUT_FILENO) {
    dup2(pfd[1], STDOUT_FILENO);
    close(pfd[1]);
}
```

### popen

`popen()`用来执行一个shell命令，并且可以通过管道读取其输出或发给它一个输入。

```c
#include <stdio.h>

FILE *popen(const char *command, const char *mode);

int pclose(FILE *stream);
```

`command`为想要执行的命令，`mode`为`r`或`w`。`popen()`返回的描述符只能用`pclose()`关闭，而不能用`fclose()`关闭，因为`popen()`在执行过程中会创建两个进程，`pclose()`会wait子进程，而使用`fclose()`会产生僵尸进程。


## 管道和stdio缓冲

文件流是块缓冲的，所以stdio对待`popen()`返回的管道也是块缓冲的，一般向管道写入时问题不大，我们可以fflush()刷新缓冲区，或者setbuf(fp, NULL)关闭stdio缓冲。但当我们作为管道的读取端时，就不太好控制了，只能看写入端的代码怎么写的了。这种情况可以使用伪终端，在伪终端的情况下，stdio缓冲为行缓冲。

## FIFO 命名管道

```c
#include <sys/stat.h>

int mkfifo(const char *pathname, mode_t mode);
// mode 指定权限
```

FIFO和管道的类似，只不过FIFO在文件系统中会有一个路径，所以也被叫做命名管道。  

一般来说FIFO只会一个读进程和一个写进程，因此，当以只读打开一个FIFO时会阻塞，直到另一端的进程以只写打开FIFO。同样的，以只写打开一个FIFO也会阻塞至另一端以只读打开。  

想要在open的时候不阻塞，最好使用`O_NONBLOCK`标志。以此标志读FIFO时，没有进程在写端打开，则马上成功结束调用，以此标志写FIFO时，没有进程在对端读，则返回错误ENXIO。


