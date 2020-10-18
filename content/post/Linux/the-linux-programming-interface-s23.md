---
title: "Linux/Unix系统编程手册-笔记23. 线程"
date: 2020-10-18T18:30:00+08:00
tags: ["Linux"]
categories: ["Linux系统编程手册阅读"]
---

本章主要介绍了POSIX线程， 也就是pthread。一个进程可以包含多个线程，线程之间共享全局内存。  
多线程相比多进程有几个优点：
- 共享信息简单快速
- 线程创建速度快

线程的缺点：
- 多线程编程时，需要考虑线程安全。
- 某个线程存在BUG,可能危及该进程的所有线程。
- 每个线程争用宿主进程的有限的虚拟地址空间。
- 多线程中处理信号需要特别小心
- 除了数据，还可以共享文件描述符、信号处理、当前工作目录、以及用户ID和组ID，优劣视应用而定。

线程在进程中执行时的内存布局如下图：
![线程在进程内运行时内存布局](/img/the-linux-programming-interface-s23/four_threads_in_a_process.png)

线程之间共享的信息：

- 进程ID和父进程ID
- 进程组ID和会话ID
- 控制终端
- 用户和组ID
- 文件描述符
- 使用fcntl()创建的record locks
- 信号配置
- 文件系统相关信息：umask，当前工作目录，根目录
- 间隔计时器 (setitimer())和POSIX计时器 (timer_create())
- System V信号量
- 资源限制
- CPU 时间消耗(times())
- 资源消耗(getrusage())
- nice值

线程之间不共享的属性：

- 线程id
- 信号mask
- 线程相关的数据
- 备用信号栈
- errno 变量
- 浮点数环境（fenv(3)）
- 实时调度策略和优先级
- CPU affinity
- capabilities
- 栈

## 创建线程

```c
#include <pthread.h>

int pthread_create(pthread_t *thread, const pthread_attr_t *attr, void *(*start)(void *), void *arg);

// Return 0 on success, or a positive error number on error
```

新线程从函数start以参数arg开始执行，参数thread指向的缓冲区，在此保存一个该进程的唯一标识， 供后续使用。start函数的返回值不应指定在线程栈中，因为线程终止后无法确定线程栈的内容是否有效。  
参数attr是指向pthread_attr_t对象的指针，指定新线程的属性。

## 终止进程

```c
#include <pthread.h>

void pthread_exit(void *retval);

```

相当于在start函数内return。retval指定了线程返回值，举个使用的栗子:

```c
void *thread(void *arg) {
  char *ret;

  if ((ret = (char*) malloc(20)) == NULL) {
    perror("malloc() error");
    exit(2);
  }
  strcpy(ret, "This is a test");
  pthread_exit(ret);
}

```

## pthread_join()

函数thread_join()等待由thread标识的线程终止（若已经终止会立即返回）

```c
#include <pthread.h>

int pthread_join(pthread_t thread, void **retval);

// Return 0 on sucess, or a positive error number on error
```

若线程未分离（detached），而又没有使用pthread_join(),那么线程终止后将产生僵尸线程。  
pthread_join()和waitpid()之间的区别：

- 一个进程内的线程之间的关系是对等的，进程中的任意线程都可以调用pthread_join()与该进程的任何其他线程连接起来。
- pthread_join()无法连接任意线程，必须指定特定的线程ID。也不能以非阻塞方式连接。

## 线程的分离

如果不关心进程的返回状态，只希望进程在线程终止的时候能自动清理并移除之，可以使用pthread_detach()

```c
#include <pthread.h>

int pthread_detach(pthread_t thread);

// Return 0 on sucess, or a positive error number on error
```

但是当其他线程调用了exit(), 或主进程执行return时，即使分离的线程也会收到影响，进程的所有线程将会立即停止。

