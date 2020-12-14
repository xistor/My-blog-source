---
title: "Linux/Unix系统编程手册-笔记30. System V 消息队列"
date: 2020-12-14T18:30:00+08:00
tags: ["Linux"]
categories: ["Linux系统编程手册阅读"]
---



## API

### msgsnd

```c
#include <sys/types.h> /* For portability */
#include <sys/msg.h>
int msgsnd(int msqid, const void *msgp, size_t msgsz, int msgflg);
// Returns 0 on success, or –1 on error
```

`msgp`是一个自定义的消息数据结构，这个结构体里一定要定义一个`mtype`域来表示消息类型：
```c
struct mymsg {
    long mtype; /* Message type */
    char mtext[]; /* Message body */
}
```

### msgrcv

```c
#include <sys/types.h> /* For portability */
#include <sys/msg.h>
ssize_t msgrcv(int msqid, void *msgp, size_t maxmsgsz, long msgtyp, int msgflg);
// Returns number of bytes copied into mtext field, or –1 on error
```

`msgtyp`参数可以控制接收的消息类型：
- 如果`msgtyp`等于0，就按队列顺序读取消息。
- 如果`msgtyp`大于0，队列中第一个类型为`msgtyp`的消息将被读取。
- 如果`msgtyp`小于0，消息队列可以看作一个优先级队列，消息类型小于或等于`msgtyp`的绝对值，且其值最小的第一条消息将被返回给调用进程。

`msgflg`参数：

- IPC_NOWAIT: 不阻塞，若无对应消息，返回ENOMSG
- MSG_EXCEPT: 只在`msgtyp`大于0的时候起作用，将按顺序读取类型不等于`msgtyp`的消息。
- MSG_NOERROR： 默认情况下，消息结构体内的`mtext`域的数据大小超过参数`maxmsgsz`,将失败，如果指定了此flag,将会截断数据。

## 使用System V消息队列作为CS结构应用通信方式：
1. 客户端和服务端都使用同一个消息队列，发给不同客户端的消息使用消息类型（messag type）来区分，一般选用客户端的进程ID作为消息类型，客户端选择和自己进程id相同的消息收取，一般使用`1`作为发给服务端的消息类型。
![使用一个消息队列](/img/the-linux-programming-interface-s30/one_message_queue.png)

结构如上图，这种方式存在两个问题，一个是消息比较多的时候，容易造成消息队列阻塞，造成死锁。
第二个是，一个恶意的客户端如果往消息队列填入大量垃圾消息，很容易造成服务端不响应。

2.每个客户端使用一个消息队列，一般来说，服务端会先创建一个消息队列，供客户端将自己创建的消息队列标识符发送给服务端。可能存在的问题就是系统中消息队列的数量是有限制的。

![每个客户端使用一个消息队列](/img/the-linux-programming-interface-s30/per_mq.png)
