---
title: "Linux/Unix系统编程手册-笔记34. POSIX IPC"
date: 2020-12-31T00:05:00+08:00
tags: ["Linux"]
categories: ["Linux系统编程手册阅读"]
---

POSIX 提供的三种IPC和System V类似，分别是消息队列、信号量、共享内存。  
和System V 比较优势在于：
- 接口更简单,如下图
![POSIX IPC](/img/the-linux-programming-interface-s34/POSIX_IPC.png)
- 使用名字代替key,使用open、close、以及unlink函数，与传统的的UNIX文件模型更加一致。
- POSIX IPC对象是引用计数的，简化了对象删除，当所有进程都关闭该对象之后对象就会被销毁。

POSIX IPC劣势在于在可移植性上不如System v。  


## POSIX 消息队列

使用流程 `mq_open()` -> `mq_send()` -> `mq_receive()` -> `mq_close()` -> `mq_unlink()`。

### 创建消息队列

```c

#include <fcntl.h>            /* Defines O_* constants */
#include <sys/stat.h>         /* Defines mode constants */
#include <mqueue.h>

mqd_t mq_open(const char *name, int oflag, ... /* mode_t mode, struct mq_attr *attr */);
// Returns a message queue descriptor on success, or (mqd_t) –1 on error
```

打开一个既有的消息队列，只需要name和oflag, 但如果在flag中指定了O_CREAT创建一个消息队列,那么还需要另外两个参数：mode 指定创建的消息队列的权限，attr指定新消息队列的属性，attr为NULL的话，将使用默认属性。

```c
// mq_attr 结构体如下

struct mq_attr {
    long mq_flags;       /* Flags: 0 or O_NONBLOCK */
    long mq_maxmsg;      /* Max. # of messages on queue */
    long mq_msgsize;     /* Max. message size (bytes) */
    long mq_curmsgs;     /* # of messages currently in queue */
};

```
可以使用`mq_setattr()`和`mq_getattr()`来设置和获取队列属性。SUSv3规定`mq_setattr()`只能修改O_NONBLOCK标记的状态。  

`mq_open()`的返回值为mqd_t类型，在linux上是一个int。POSIX消息队列描述符和文件描述符类似，实际上Linux上也是这么做的，用虚拟文件系统中的i-node实现了POSIX队列。



### 交换消息


```c
#include <mqueue.h>
int mq_send(mqd_t mqdes, const char *msg_ptr, size_t msg_len, unsigned int msg_prio);
// Returns 0 on success, or –1 on error

/* 
参数 msg_prio 表示消息优先级, 0表示优先级最低
msg_ptr 指向消息缓冲区
msg_len 消息长度
*/


ssize_t mq_receive(mqd_t mqdes, char *msg_ptr, size_t msg_len, unsigned int *msg_prio);
// Returns number of bytes in received message on success, or –1 on error
```

如果消息队列已经满了，那么后续的`mq_send()`调用会阻塞直到队列中存在的可用空间，或者指定了O_NONBLOCK标记其作用时立即失败并返回EAGAIN。  

`mq_revceive()`会从队列中删除一条优先级最高、存在时间最长的消息，并放置到msg_ptr指向的缓冲区。




### 消息通知

 ```c
 #include <mqueue.h>
 int mq_notify(mqd_t mqdes, const struct sigevent *notification);
 
 // Returns 0 on success, or –1 on error

// sigevent结构体的原型

struct sigevent {    
    int    sigev_notify;          /* Notification method */    
    int    sigev_signo;           /* Notification signal for SIGEV_SIGNAL */    
    union sigval sigev_value;     /* Value passed to signal handler or thread function */    
    void (*sigev_notify_function) (union sigval); /* Thread notification function */    
    void  *sigev_notify_attributes;   /* Really 'pthread_attr_t' */
};

 ```
`mq_notify()`函数注册调用进程在一条消息进入描述符mqdes引用的空队列时接收通知。  

- 任何时候一个消息队列只能有一个进程注册消息通知
- 只有当一条新消息进入之前为空的队列时注册进程才会收到通知
- 当向注册进程发送了一个通知后就会删除消息，之后如果一个进程需要继续接受通知，他就必须要在每次收到通知之后再次调用`mq_notify()`来注册自己
- 注册进程只有在当前不存在其他在该队列上的调用`mq_receive()`而发生阻塞时，才会收到通知。
- 一个进程可以在调用`mq_notify()`时传入NULL的notification参数来撤销自己在消息通知上的注册信息。


sigevent结构体中sigev_notify指定通知方式，当以SIGEV_SIGNAL指定以信号的方式接收通知时，sigev_signo可以指定实时信号。当以SIGEV_THREAD指定以sigev_notify_function函数通知的方式，将会在新线程中启动该函数一样。  


