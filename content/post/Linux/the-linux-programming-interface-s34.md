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


```cpp
// 信号通知
// 主要代码
mqd = mq_open(argv[1], O_RDONLY | O_NONBLOCK);

...

sev.sigev_notify = SIGEV_SIGNAL;
sev.sigev_signo = NOTIFY_SIG;
if (mq_notify(mqd, &sev) == -1)
    errExit("mq_notify");
sigemptyset(&emptyMask);
for (;;) {
    sigsuspend(&emptyMask); /* Wait for notification signal */
    if (mq_notify(mqd, &sev) == -1)
        errExit("mq_notify");
    while ((numRead = mq_receive(mqd, buffer, attr.mq_msgsize, NULL)) >= 0)
        printf("Read %ld bytes\n", (long) numRead);
    if (errno != EAGAIN) /* Unexpected error */
    errExit("mq_receive");
}

```

sigsuspend的整个原子操作过程为：
1. 设置新的mask阻塞当前进程;
2. 收到信号，调用该进程设置的信号处理函数;
3. 待信号处理函数返回后，恢复原先mask
4. sigsuspend返回

使用sigsuspend而不使用pause是为了避免丢失信号，因为sigsuspend为原子操作，退出时会重新阻塞信号。  

使用O_NONBLOCK打开是为了在while中全部读完消息后退出，而不是阻塞在mq_receive。  

其次在读取消息之前先调用mq_notify重新注册通知，是因为若写在while循环读空消息队列之后，在注册之前的一小段时间内又来了一个通知，那么注册时队列将不为空，之后将再也收不到消息通知。

```cpp
// 线程通知的主要代码

static void /* Thread notification function */
threadFunc(union sigval sv)
{
    ssize_t numRead;
    mqd_t *mqdp;
    void *buffer;
    struct mq_attr attr;
    mqdp = sv.sival_ptr;

    if (mq_getattr(*mqdp, &attr) == -1)
        errExit("mq_getattr");
    buffer = malloc(attr.mq_msgsize);

    if (buffer == NULL)
        errExit("malloc");

    // 在读取前先重新注册
    notifySetup(mqdp);

    while ((numRead = mq_receive(*mqdp, buffer, attr.mq_msgsize, NULL)) >= 0)
        printf("Read %ld bytes\n", (long) numRead);

    if (errno != EAGAIN) /* Unexpected error */
        errExit("mq_receive");

    free(buffer);
    pthread_exit(NULL);
}

notifySetup(mqd_t *mqdp)
{
    struct sigevent sev;
    // 设置为线程通知
    sev.sigev_notify = SIGEV_THREAD; /* Notify via thread */
    sev.sigev_notify_function = threadFunc;
    sev.sigev_notify_attributes = NULL;
    /* Could be pointer to pthread_attr_t structure */
    sev.sigev_value.sival_ptr = mqdp; /* Argument to threadFunc() */
    if (mq_notify(*mqdp, &sev) == -1)
        errExit("mq_notify");
}

int
main(int argc, char *argv[])
{
    mqd_t mqd;
    if (argc != 2 || strcmp(argv[1], "--help") == 0)
    usageErr("%s mq-name\n", argv[0]);
    // NONBLOCK打开
    mqd = mq_open(argv[1], O_RDONLY | O_NONBLOCK);
    if (mqd == (mqd_t) -1)
        errExit("mq_open");
    notifySetup(&mqd);
    pause(); /* Wait for notifications via thread function */
}
```

## POSIX 信号量

相关API和POSIX消息队列的类似，和System V 信号量有点区别：
- `sem_post()`和`sem_wait()`一个只操作一个信号量，而且每次只递增或递减1。
- 没有wait-for-zero操作

对于命名信号量，使用`sem_open()`创建。  

对于无名信号量：  

使用`sem_init()`可以初始化一个无名信号量，并且可以通过参数pshared指定信号量sem是在线程间共享还是在进程间共享。线程间共享时sem一般是一个全局变量或者堆上分配的变量，进程间共享时，sem需要是一个共享内存或者共享内存映射的地址。


```c
#include <semaphore.h>
int sem_init(sem_t *sem, int pshared, unsigned int value);
// Returns 0 on success, or –1 on error
```

## POSIX 共享内存

使用POSIX共享内存分两步：
- 使用`shm_open()`以指定名字打开一个共享内存对象。
- 将上一步打开的文件描述符传递给mmap(),使用 MAP_SHARED flag将其映射到进程的虚拟内存空间。


使用要点：

```cpp
    // 写东西到共享内存中
    fd = shm_open(argv[1], O_RDWR, 0); /* Open existing object */
    if (fd == -1)
        errExit("shm_open");
    len = strlen(argv[2]);
    if (ftruncate(fd, len) == -1) /* Resize object to hold string */
        errExit("ftruncate");
    printf("Resized to %ld bytes\n", (long) len);
    addr = mmap(NULL, len, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    if (addr == MAP_FAILED)
        errExit("mmap");
    if (close(fd) == -1)
        errExit("close"); /* 'fd' is no longer needed */
    printf("copying %ld bytes\n", (long) len);
    memcpy(addr, argv[2], len); /* Copy string to shared memory */
    exit(EXIT_SUCCESS);

    // 从共享内存中读取

    fd = shm_open(argv[1], O_RDONLY, 0); /* Open existing object */
    if (fd == -1)
        errExit("shm_open");
    /* Use shared memory object size as length argument for mmap()
    and as number of bytes to write() */
    if (fstat(fd, &sb) == -1)
        errExit("fstat");
    addr = mmap(NULL, sb.st_size, PROT_READ, MAP_SHARED, fd, 0);
    if (addr == MAP_FAILED)
        errExit("mmap");
    if (close(fd) == -1); /* 'fd' is no longer needed */
        errExit("close");

    // 删除共享内存
    if (shm_unlink(argv[1]) == -1)
        errExit("shm_unlink");
```

