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

也正是因为key是用来区分函数的，所以返回的key可以被进程中的所有线程使用，指向的是一个由我们事先声明的全局变量。

```c
#include <pthread.h>

int pthread_setspecific(pthread_key_t key, const void *value);
// return 0 on success, or a positive error number on error
void *pthread_getspecific(pthread_key_t key);
// return pointer, or a NULL if no thread-specific data is associated with key
```

`pthread_setspecific()`的参数`key`是调用`pthread_key_create()`分配的key,`value`通常指向一块由调用者分配的内存，线程终止时，会自动调用`create`时指定的`destructor()`去释放`value`指向的内存。   
下图是线程特有数据的实现结构，如图，Pthread为每个线程维护一个指针数组，指针数组内每一个成员都指向一个函数的特有数据。key1所对应的函数的特有数据数据保存在tsd[1]指向的内存中。  
![线程特有数据的实现结构](/img/the-linux-programming-interface-s23/tsd.png)
其实根据上面这个图，之前对key的总结有点补充，key是用来区分函数的，但并不代表一个函数内只能用一个key，一个函数中可以创建和指定多个key。但上限有限，linux中key最多可以1024个。通常只用一个key就可以了，将函数要用的多个特有数据值放到一个结构中。

### 线程局部存储(thread-local Storage)

线程局部存储比线程特有数据使用起来更加简单,只要在声明全局或静态变量的时候加上__thread说明符，每个线程就会持有一份此变量的拷贝：

```c
static __thread buf[MAX_ERROR_LEN];
```

## 线程取消

调用下面系统调用可以请求取消掉指定线程(在其他线程中调用)。

```c
#include <pthread.h>
int pthread_cancel(pthread_t thread);
// Returns 0 on success, or a positive error number on error
```

### 线程取消状态
调用如下接口，可以控制本线程对取消请求的应对方式。

```c
#include <pthread.h>
int pthread_setcancelstate(int state, int *oldstate);
int pthread_setcanceltype(int type, int *oldtype);
// Both return 0 on success, or a positive error number on error
```

pthread_setcancelstate()参数state可选状态如下：
- PTHREAD_CANCEL_DISABLE: 线程不可取消，取消状态被挂起，直到状态改为enable
- PTHREAD_CANCEL_ENABLE: 线程可取消

如果线程是可取消的，那线程取消的请求结果由可取消类型决定
- PTHREAD_CANCEL_ASYNCHRONOUS：线程可以在任何时间被取消（不一定是马上被取消）。
- PTHREAD_CANCEL_DEFERRED：请求被挂起，直到下一个取消点。

### 取消点

仅当取消操作安全时才应取消线程。pthreads 标准指定了几个取消点，其中包括：

- 通过 pthread_testcancel 调用以编程方式建立线程取消点。如果当前有一个取消请求在等待，线程就会取消。
- 线程等待 pthread_cond_wait 或 pthread_cond_timedwait(3C) 中的特定条件出现。
- 线程在 pthread_join()等待其他线程结束。
- 被 sigwait(2) 阻塞的线程。
- 一些标准的库调用。通常这些调用可使其线程阻塞。有关列表，请参见 cancellation(5) 手册页。


线程取消的危险性主要与未释放共享资源有关，取消时须十分注意，否则可能导致死锁。为了避免这些，取消点因此选在线程阻塞的时候，减小持有资源的可能，除此之外还可以使用cleanup handler

### cleanup handlers

```c
#include <pthread.h>
void pthread_cleanup_push(void (*routine)(void*), void *arg);
void pthread_cleanup_pop(int execute);
```
清理函数是以栈的形式来管理的，这两个函数很明显用来添加和删除handler，每个push都应该有一个对应的pop。当线程执行到最后退出的话，是不需要调用清理函数的，所以要在适当的时候将handler从栈中pop出来。若`pthread_cleanup_pop()`参数`execute`不为0，弹出的栈顶处理函数的同时会执行清理函数。线程因调用pthread_exit()终止的时候，会自动执行尚未从清理函数栈中弹出的清理函数，线程正常返回(return)时不会执行s清理函数。

### 异步取消

如果设定线程可异步取消时，可以在任何时点将其取消，取消动作不会拖延到下一个取消点才执行。异步取消时虽然清理函数可以执行，但是无法得知线程当前执行到哪一步。所以原则上可异步取消的线程不应该分配资源。

## 线程细节

### 栈

每个线程有自己的固定大小的栈，以下函数可以修改：

```c
#include <pthread.h>
// 可以更改栈大小
int pthread_attr_setstacksize(pthread_attr_t *attr, size_t stacksize);
// 可以更该栈地址和栈大小
int pthread_attr_setstack(pthread_attr_t *attr, void *stackaddr, size_t stacksize);
```

使用栗子如下，可见上面两个函数是将栈size或地址写到属性中，在创建线程的时候再使用指定属性创建线程。

```c
#include <stdio.h>                                                              
#include <pthread.h>                                                            
                                                                                
void *thread1(void *arg)                                                        
{                                                                               
   printf("hello from the thread\n");                                           
   pthread_exit(NULL);                                                          
}                                                                               
                                                                                
int main()                                                                      
{                                                                               
   int            rc, stat;                                                     
   size_t         s1;                                                           
   pthread_attr_t attr;                                                         
   pthread_t      thid;                                                         
                                                                                
   rc = pthread_attr_init(&attr);                                               
   if (rc == -1) {                                                              
      perror("error in pthread_attr_init");                                     
      exit(1);                                                                  
   }                                                                            
                                                                                
   s1 = 4096;                                                                   
   rc = pthread_attr_setstacksize(&attr, s1);                                   
   if (rc == -1) {                                                              
      perror("error in pthread_attr_setstacksize");                             
      exit(2);                                                                  
   }                                                                            
                                                                                
   rc = pthread_create(&thid, &attr, thread1, NULL);                            
   if (rc == -1) {                                                              
      perror("error in pthread_create");                                        
      exit(3);                                                                  
   }                                                                            
                                                                                
   rc = pthread_join(thid, (void *)&stat);                                      
   exit(0);                                                                     
}

```

### 线程和信号

信号的接收：

- 信号动作（signal action）是进程级的，如果一个默认动作是`stop`或`terminate`的信号送达任意一个线程，那么进程中的所有线程都会`stop`或`terminate`。
- 信号处置是进程级的，进程中所有线程对于一个信号共享同样的处置，如果一个线程使用`sigaction()`为信号创建了信号处理函数，当信号到达时，信号处理函数可能在任一进程中执行。
- 在以下情况下信号会被发给特定的线程：
  1. 由于线程执行的硬件指令异常产生的信号(SIGBUS, SIGFPE, SIGILL, and SIGSEGV),只有产生异常的线程收到并处理。
  2. 写一个被破坏的管道时产生的SIGPIPE信号。
  3. pthread_kill() or pthread_sigqueue()发送给指定线程。
- 当信号被发送到一个多线程进程时，内核会任意选择一个线程，把信号发给它，并在那个线程中调用线程处理函数。
- signal mask是线程级的，线程的信号掩码继承自创建它的线程，每个线程可以使用`pthread_sigmask()`单独block或unblock某个信号。
- 内核为进程维护一个挂起信号表，也为每个线程维护一个挂起信号表，sigpending()返回的线程正在挂起的信号和进程正在挂起的信号的并集。
- 当信号处理函数打断了pthread_mutex_lock()和pthread_cond_wait()的调用，会自动重新执行调用。
- 备用信号栈是线程级的。

### 线程和进程控制

- 当某一个线程调用`exec()`后，调用程序会被完全替换掉。线程的特有数据析构函数和清理处理函数都不会被调用。
- 当多线程程序调用`fork()`后，只有调用线程会被复制到子进程中。其他线程都会消失，它们的清理函数也不会被调用，这就带来一些问题：
  1. 除了调用线程，全局变量包括互斥量等也会被复制进子进程，这就导致如果在`fork()`的时候，其他线程锁了一个互斥量，`fork()`完毕后，它消失了..那个互斥量就会被永远的锁住。
  2. 因为清理函数不会被调用，所以还有可能导致内存泄漏。  
基于以上原因，在多线程程序中使用`fork()`最好马上调用`exec()`。如果不能，Pthread提供了pthread_atfork()接口

```c
#include <pthread.h>

int pthread_atfork(void (*prepare_func)(void), void (*parent_func)(void), void (*child_func)(void));
```
prepare_func会被加入一个函数列表中，在子进程创建之前，自动按照注册顺序反序执行。parent_func和child_func在fork()返回之前，分别在父进程和子进程中按注册顺序执行。
下面是栗子:
```cpp
#define _UNIX03_THREADS 1                                                       

#include <pthread.h>                                                            
#include <stdio.h>                                                              
#include <unistd.h>   
#include <fcntl.h>   
#include <sys/types.h>
#include <sys/wait.h>
#include <stdlib.h>
#include <errno.h>                                                              
                                                                                
char fn_c[] = "childq.out";
char fn_p[] = "parentside.out";
int  fd_c;
int  fd_p;

void prep1(void)  {
  char buff[80] = "prep1\n";
  printf("prep1\n");
  write(4,buff,sizeof(buff));
}

void prep2(void)  {
  char buff[80] = "prep2\n";
  printf("prep2\n");
  write(4,buff,sizeof(buff));
}

void prep3(void)  {
  char buff[80] = "prep3\n";
  printf("prep3\n");
  write(4,buff,sizeof(buff));
}

                                                                               
void parent1(void)  {
  char buff[80] = "parent1\n";
  printf("parent1\n");
  write(4,buff,sizeof(buff));
}

void parent2(void)  {
  char buff[80] = "parent2\n";
  printf("parent2\n");
  write(4,buff,sizeof(buff));
}

void parent3(void)  {
  char buff[80] = "parent3\n";
  printf("parent3\n");
  write(4,buff,sizeof(buff));
}


void child1(void)  {
  char buff[80] = "child1\n";
  printf("child1\n");
  write(3,buff,sizeof(buff));
}

void child2(void)  {
  char buff[80] = "child2\n";
  printf("child2\n");
  write(3,buff,sizeof(buff));
}

void child3(void)  {
  char buff[80] = "child3\n";
  printf("child3\n");
  write(3,buff,sizeof(buff));
}

void *thread1(void *arg) {
                                                                                
  printf("Thread1: Hello from the thread.\n");

} 
                                                                                

int main(void)
{                                                                        
  pthread_t thid;
  int       rc, ret;
  pid_t     pid;
  int       status;
  char   header[30] = "Called Child Handlers\n";

  if (pthread_create(&thid, NULL, thread1, NULL) != 0) {
    perror("pthread_create() error");                                           
    exit(3);                                                                    
  }

  if (pthread_join(thid, NULL) != 0) {
    perror("pthread_join() error");
    exit(5);                                                                    
  } else {
    printf("IPT: pthread_join success!  Thread 1 should be finished now.\n");
    printf("IPT: Prepare to fork!!!\n");
  }
 

  /*-----------------------------------------*/
  /*|  Start atfork handler calls in parent  */
  /*-----------------------------------------*/
  /* Register call 1 */
  rc = pthread_atfork(&prep1, &parent2, &child3);
  if (rc != 0) {
     perror("IPT: pthread_atfork() error [Call #1]"); 
     printf("  rc= %d, errno: %d", rc, errno); 
  }
 

  /* Register call 2 */
  rc = pthread_atfork(&prep2, &parent3, &child1);
  if (rc != 0) {
     perror("IPT: pthread_atfork() error [Call #2]"); 
     printf("  rc= %d, errno: %d", rc, errno); 
  }
  

  /* Register call 3 */
  rc = pthread_atfork(&prep3, &parent1, NULL);
  if (rc != 0) {
     perror("IPT: pthread_atfork() error [Call #3]"); 
     printf("  rc= %d, errno: %d", rc, errno); 
  }

  /* Create output files to expose the execution of fork handlers. */
  if ((fd_c = creat(fn_c, S_IWUSR)) < 0)
    perror("creat() error");
  else
    printf("Created %s and assigned fd= %d\n", fn_c, fd_c);
  if ((ret = write(fd_c,header,30)) == -1)
    perror("write() error");
  else
    printf("Write() wrote %d bytes in %s\n", ret, fn_c);

  if ((fd_p = creat(fn_p, S_IWUSR)) < 0)
    perror("creat() error");
  else
    printf("Created %s and assigned fd= %d\n", fn_p, fd_p);
  if ((ret = write(fd_p,header,30)) == -1)
    perror("write() error");
  else
    printf("Write() wrote %d bytes in %s\n", ret, fn_p);

  pid = fork();

  if (pid < 0) 
    perror("IPT: fork() error"); 
  else {
    if (pid == 0) {
      printf("Child: I am the child!\n");
      printf("Child: My PID= %d, parent= %d\n", (int)getpid(), 
              (int)getppid());
      exit(0);

    } else {
      printf("Parent: I am the parent!\n");
      printf("Parent: My PID= %d, child PID= %d\n", (int)getpid(), (int)pid);

      if (wait(&status) == -1)  
        perror("Parent: wait() error");
      else if (WIFEXITED(status))
             printf("Child exited with status: %d\n",WEXITSTATUS(status)); 
           else
             printf("Child did not exit successfully\n");

    close(fd_c);
    close(fd_p);

    }
  }
}

```

```
终端输出：
Thread1: Hello from the thread.
IPT: pthread_join success!  Thread 1 should be finished now.
IPT: Prepare to fork!!!
Created childq.out and assigned fd= 3
Write() wrote 30 bytes in childq.out
Created parentside.out and assigned fd= 4
Write() wrote 30 bytes in parentside.out
Parent: I am the parent!
Parent: My PID= 42349, child PID= 42351
Child: I am the child!
Child: My PID= 42351, parent= 42349
Child exited with status: 0

文件内容：
$ cat parentside.out 
Called Child Handlers
prep3
prep2
prep1
parent2
parent3
parent1

$ cat childq.out 
Called Child Handlers
child3
child1

```
从输出中能看出子进程中调用的处理函数可以通过文件描述符`3`往`childq.out`文件写内容，父进程可以通过文件描述符`4`往`parentside.out`文件中写内容。根据文件内容可以看出函数执行顺序。

 