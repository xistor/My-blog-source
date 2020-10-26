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

## 线程的同步

因为线程之间能够共享全局变量，所以就存在竞争问题，为了安全的共享变量，不同线程之间需要同步操作。

### 互斥量

互斥量pthread_mutex_t使用前必须初始化

```cpp
#include <pthrad.h>

// 静态互斥量的初始化
static pthread_mutex_t mtx = PTHREAD_MUTEX_INITIALIZER;


// 动态互斥量的初始化

int func() {
  pthread_mutex_t mtx;
  pthread_mutexattr_t mtxAttr;

  int s, type;

  s = pthread_mutexattr_init(&mtxAttr);
  if (s != 0)
    return;
  // 设置属性
  s = pthread_mutexattr_settype(&mtxAttr, PTHREAD_MUTEX_ERRORCHECK);

  if (s ！= 0)
    return;
  s = pthread_mutex_init(mtx, &mtxAttr);

  pthread_mutex_init(mtx, );

}
```


对互斥量加锁和解锁

```c
#include <pthrad.h>

int pthread_mutex_lock(pthread_mutex_t *mutex);

int pthread_mutex_unlock(pthread_mutex_t *mutex);

// Return 0 on success, or a positive error number on error
```

如果互斥量之前未锁定，执行锁定操作将会立即返回，否则将会一直阻塞到互斥量被解锁。  

当不再需要互斥量时，应使用pthread_mutex_destory()将其销毁。

```c
#include <pthread.h>

int pthread_mutex_destory(pthread_mutex_t *mutex);

// Return 0 on success, or a positive error number on error
```

互斥量的属性之一 类型：

- PTHREAD_MUTEX_NORMAL: 该互斥量不具有死锁自检功能，对与已经锁住的互斥量，再次加锁会导致不确定结果。（linux下会成功）
- PTHREAD_MUTEX_ERRORCHECK: 会对加解锁过程做检查。
- PTHREAD_MUTEX_RECURSIVE: 递归维护一个锁计数器，每次加锁会+1，每次解锁会-1，当计数为0时才会释放该互斥量。解锁时和上一个类型一样，若互斥量处于未锁定状态或已由其他线程锁定， 操作都会失败。  
还有其他类型，不赘述。


### 条件变量

条件变量可以在共享变量状态改变的时候通知其他线程，其他线程可以等待通知。

条件变量初始化

```c
#include <pthread.h>

// 静态条件变量初始化
static pthread_cond_t cond = PTHREAD_COND_INITIALIZER;

// 动态条件变量初始化

int pthread_cond_init(pthread_cond_t *cond, const pthread_condattr_t *attr);
// Returns 0 on success, or a positive error number on error
```

通知和等待条件变量操作

```c
#include <pthread.h>
int pthread_cond_signal(pthread_cond_t *cond);
int pthread_cond_broadcast(pthread_cond_t *cond);
int pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex);
int pthread_cond_timedwait(pthread_cond_t *cond, pthread_mutex_t *mutex,
  const struct timespec *abstime);
// All return 0 on success, or a positive error number on error
```

`pthread_cond_signal()`和`pthread_cond_broadcast()`区别在于前者只保证通知至少一个阻塞在`pthread_cond_wait()`的线程，后者会通知所有阻塞的线程。除非只有一个阻塞线程需要唤醒，比如所有线程都执行同一操作，只需要唤醒一个线程做就行。使用`pthread_cond_broadcast()`一般会得到正确结果。**`pthread_cond_signal()`通知的时候若没有线程在等待，就会被忽略。**  

从`pthread_cond_wait()`的参数就能看出，它需要和mutex配合使用，在其内部对mutex会按顺序做如下操作：
1. 释放mutex;
2. 阻塞当前调用线程，直到其他线程通知条件变量改变
3. 再次获取mutex

`pthread_cond_wait()`传入的mutex就是用于控制共享变量访问时用到的mutex, 为什么需要这个mutex呢？ 这个mutex就是为了保护共享变量的，这个共享变量的作用类似“状态(state)”或“标志(flag)”，比如下面这段程序中的`avail`。在检查或修改共享变量之前都需要拥有mutex。这样设计的原因是因为条件变量和互斥量之间存在着天然的关系：
1. 线程在准备检查共享变量状态时锁定互斥量。
2. 检查共享变量状态
3. 如果共享变量未处于预期状态，线程应在等待条件变量并进入休眠前解锁互斥量（以便其他线程能访问该共享变量）
4. 当线程因为条件变量的通知而被再度唤醒时，必须对互斥量再次加锁，因为在典型情况下，线程会立即访问共享变量。

pthread_cond_wait()会自动执行最后两步中对互斥量的解锁和加锁动作，并且第3步的互斥量释放和陷入休眠属于一个原子操作。在这个地方一开始我还存在困扰，我想如果在生产者发条件变量信号之后，消费者才陷入等待，岂不是会丢掉信号。其实我看错了，因为消费者陷入等待之前会首先判断共享变量也就是状态是否满足，不满足要求才会陷入等待。而前述情况既然发了条件变量信号，就说明`avail`已经`++`了，是不会等待的。所以是我之前看的太局部，没有考虑到条件变量是为线程间访问临界资源服务的。  

使用`while()`而不是`if()`来检查判断条件是因为从pthread_cond_wait()返回后需要重新检查判断条件，这是因为：
- 其他线程可能会率先醒来，改变了判断条件的状态。
- 条件变量信号意味着“可能有事情去做”，而不是“一定有事情去做”，接收信号的线程可以通过再次检查条件来确定是否真的需要做什么。
- 可能出现虚假唤醒的情况



```c

static pthread_mutex_t mtx = PTHREAD_MUTEX_INITIALIZER;
static pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
static int avail = 0;

// 生产者
pthread_mutex_lock(&mtx);

avail++;

pthread_mutex_unlock(&mtx);

pthread_cond_signal(&cond);

```

```c
// 消费者
pthread_mutex_lock(&mtx);

while (avail == 0) { /* Wait for something to consume */
  pthread_cond_wait(&cond, &mtx);
}

while (avail > 0) { /* Consume all available units */
/* Do something with produced unit */
  avail--;
}

pthread_mutex_unlock(&mtx);
```


## 线程安全

一个线程安全的函数允许同时被多个线程调用，若一个函数中使用了全局变量，通常需要互斥量来同步以保证线程安全。不用互斥量就可以被多线程安全调用的函数称为可重入函数，可重入函数中不会用到全局和静态变量。  

### One-time 初始化

有时候，可能有一个函数会被每一个线程调用，但只希望第一次调用的时候执行。

```c
#include <pthread.h>

pthread_once_t once_control = PTHREAD_ONCE_INIT;

int pthread_once(pthread_once_t *once_control, void (*init)(void));

// return 0 on success, or a positive error number on error
```

使用栗子：

```cpp
#include <stdio.h>                                                              
#include <errno.h>                                                              
#include <pthread.h>                                                            
                                                                                
#define threads 3                                                               
                                                                                
int             once_counter=0;                                                 
pthread_once_t  once_control = PTHREAD_ONCE_INIT;                               
                                                                                
void  once_fn(void)                                                             
{                                                                               
  puts("in once_fn");                                                            
  once_counter++;                                                                
}                                                                               
                                                                                
void *threadfunc(void *parm)                                         
{                                                                               
  int        status;                                                             
  int        threadnum;                                                          
  int        *tnum;                                                              
                                                                                
  tnum = parm;                                                                   
  threadnum = *tnum;                                                             
                                                                                
  printf("Thread %d executing\n", threadnum);                                    
                                                                                
  status = pthread_once(&once_control, once_fn);                                 
  if ( status <  0)                                                              
    printf("pthread_once failed, thread %d, errno=%d\n", threadnum,             
                                                            errno);             
                                                                                
  pthread_exit((void *)0);                                                     
}                                                                               
                                                                                
main() {                                                                        
  int          status;                                                           
  int          i;                                                                
  int          threadparm[threads];                                              
  pthread_t    threadid[threads];                                                
  int          thread_stat[threads];                                             
                                                                                
  for (i=0; i<threads; i++) {                                                    
    threadparm[i] = i+1;                                                        
    status = pthread_create( &threadid[i],                                      
                              NULL,                                              
                              threadfunc,                                        
                              (void *)&threadparm[i]);                           
    if ( status <  0) {                                                         
        printf("pthread_create failed, errno=%d", errno);                        
        exit(2);                                                                 
    }                                                                           
  }                                                                             
                                                                                
  for ( i=0; i<threads; i++) {                                                   
    status = pthread_join( threadid[i], (void *)&thread_stat[i]);               
    if ( status <  0)                                                           
        printf("pthread_join failed, thread %d, errno=%d\n", i+1, errno);        
                                                                                
    if (thread_stat[i] != 0)                                                    
      printf("bad thread status, thread %d, status=%d\n", i+1,                
                                                    thread_stat[i]);             
  }                                                                             
                                                                                
  if (once_counter != 1)                                                         
    printf("once_fn did not get control once, counter=%d",once_counter);         
  exit(0);                                                                       
}                                                                               

```

```
执行结果：

Thread 1 executing
in once_fn
Thread 2 executing
Thread 3 executing
```

### 线程特有数据

线程特有数据允许每个调用函数的线程持有一份变量的拷贝。  

为了区分不同函数之间的线程特有数据，需要创建一个key来区分它们。记住，这个key是用来区分函数的，而不是线程的。

```c
#include <pthread.h>
int pthread_key_create(pthread_key_t *key, void (*destructor)(void *));
// Returns 0 on success, or a positive error number on error
```

也正是因为key是用来区分函数的，所以返回的key可以被进程中的所有线程使用，指向的是一个全局变量。
