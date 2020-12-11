---
title: "Linux/Unix系统编程手册-笔记29. System V IPC"
date: 2020-12-2T18:30:00+08:00
tags: ["Linux"]
categories: ["Linux系统编程手册阅读"]
---


System V IPC 代表了三种进程间通信机制，他们在同一时期开发，拥有相似的API。
- 消息队列(Message queues)：用于在进程间传递消息，和管道类似，区别在于信息传递以消息为单位而不是无界限的字节流，而且消息内有类型，可以以类型选择消息，而不是只能按顺序读取。
- 信号量(Semaphores)：用于进程间同步，一个信号量是一个由内核维护的整数值，对任何有合适的权限的进程都可见。
- 共享内存(Shared memory): 多个进程可以访问内存的同一区域。

![System V IPC API](/img/the-linux-programming-interface-s29/system_v_ipc_api.png)
上面是三种IPC的API,每种IPC都有对应的get系统调用，和打开文件的`open()`类似，返回一个IPC对象的标识符。IPC标识符和文件描述符的不同在于，IPC标识符是IPC对象的属性，对整个系统可见，所有的进程都使用同样的标识符访问IPC对象。  

## 创建IPC

以创建一个消息队列为例：

```c
id = msgget(key, IPC_CREAT | S_IRUSR | S_IWUSR);
if (id == -1)
 errExit("msgget");
```

get 的第一个参数`key`的类型为`key_t`，实际是一个`int`值，传入同一个key,将得到同一个IPC对象的标识符。
如果没有对应的IPC对象存在，且指定了`IPC_CREAT` flag,将会新建一个IPC对象。  
若key为`IPC_PRIVATE`则总是会创建一个新的IPC对象。`IPC_PRIVATE`适合于父进程fork进程前创建IPC对象，子进程继承标识符，以实现子进程和父进程以及兄弟进程之间通信。  
若同时指定了`IPC_PRIVATE`和`IPC_EXCL`, 如果key对应的IPC对象已经存在，则报错`EEXIST`

### IPC key

一个key对应一个IPC对象，为保证key唯一，一般使用`IPC_PRIVATE` flag或用`ftok`生成一个key。

- `IPC_PRIVATE`: 如前面所说，一般适用于`fork()`之后的相关进程之间的通信，当然也可以用于不相关进程间的通信，这就需要客户端使用某种方法获取到IPC对象的标识符，比如服务端将标识符写到指定文件中供客户端读取。
- `ftok()`:传入路径名和以一个整数值`proj`，返回一个`key`，`key`值的生成和pathname的i-node号以及proj的低8位有关，所以这两个相同，将会生成同一个key值。`proj`通常是一个字符。
```c
#include <sys/ipc.h>

key_t ftok(char *pathname, int proj);
```


## C/S应用和IPC标识符

由于IPC对象并不会随着创建进程的退出而消失，而是会已知保留在系统中。所以当服务端进程重启的时候，前一个服务端留下的IPC就需要清理，而客户端也需要方法知道服务端重启了。一般做法，服务端get时加上`IPC_EXCL`,若捕获到`EEXIST`错误，就删除掉原有的IPC对象。

```cpp
while ((msqid = msgget(key, IPC_CREAT | IPC_EXCL | MQ_PERMS)) == -1) {
    if (errno == EEXIST) { /* MQ with the same key already
        exists - remove it and try again */
        msqid = msgget(key, 0);
        if (msqid == -1)
        errExit("msgget() failed to retrieve old queue ID");
        if (msgctl(msqid, IPC_RMID, NULL) == -1)
        errExit("msgget() failed to delete old queue");
        printf("Removed old message queue (id=%d)\n", msqid);
    } else { /* Some other error --> give up */
        errExit("msgget() failed");
    }
 }

```

那在写一个System V IPC 服务端时，`ftok()`的`pathname`该指定为什么呢，毕竟客户端想要链接，需要同样使用`ftok()`来产生一个同样的key。一般来说指定一个由服务端程序自己创建的文件比较好，因为可以保证文件存在且能避免和其他程序产生冲突。如果程序没有自己创建文件，可以使用程序可执行文件本身，也就是`argv[0]`，如果创建了多个IPC，就通过`ftok()`的`proj`参数来区分。客户端连接服务端时，只要知道服务端的可执行文件所在路径，以及对应的`proj`，这需要两端协商好。

