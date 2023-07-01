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
除了`long`型的`mtype`的，结构体内的内容和长度都是自定义的，不一定非得是个字符数组。  

参数`msgsz`指定了`mtext`域的大小（bytes）。

### msgrcv

```c
#include <sys/types.h> /* For portability */
#include <sys/msg.h>
ssize_t msgrcv(int msqid, void *msgp, size_t maxmsgsz, long msgtyp, int msgflg);
// Returns number of bytes copied into mtext field, or –1 on error
```
参数`maxmsgsz`指定了`mtext`域最大可用空间大小，这点和`msgsnd()`的参数`msgsz`有点区别。默认情况下，如果消息体的大小超过了`maxmsgsz`,将产生`E2BIG`错误。

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

{{< figure src="/img/the-linux-programming-interface-s30/one_message_queue.png"  title="使用一个消息队列" class="center" width="600" >}}

结构如上图，这种方式存在两个问题，一个是消息比较多的时候，容易造成消息队列阻塞，造成死锁。
第二个是，一个恶意的客户端如果往消息队列填入大量垃圾消息，很容易造成服务端不响应。

2.每个客户端使用一个消息队列，一般来说，服务端会先创建一个消息队列，供客户端将自己创建的消息队列标识符发送给服务端。可能存在的问题就是系统中消息队列的数量是有限制的。

{{< figure src="/img/the-linux-programming-interface-s30/per_mq.png"  title="每个客户端使用一个消息队列" class="center" width="600" >}}

## Exercises

2. 使用system V 消息队列实现管道章节的sequence-number程序：

```c
// mq_seqnum_server.h

#include <sys/types.h>
#include <sys/msg.h>
#include <sys/stat.h>
#include <stddef.h>                     /* For definition of offsetof() */
#include <limits.h>
#include <fcntl.h>
#include <signal.h>
#include <sys/wait.h>
#include "tlpi_hdr.h"

#define MQ_KEY 0x1aaaaaa1           /* Key for server's message queue */

struct requestMsg {                /* Request (client --> server) */
    long mtype; 
    pid_t pid;                  /* PID of client */
    int seqLen;                 /* Length of desired sequence */
};

struct responseMsg {               /* Response (server --> client) */
    long mtype;
    int seqNum;                 /* Start of sequence */
};

#define REQ_MSG_SIZE (offsetof(struct requestMsg, seqLen) - \
                      offsetof(struct requestMsg, pid) + sizeof(int))

#define RESP_MSG_SIZE sizeof(int)

```

```c
// mq_seqnum_server.c

#include "mq_seqnum_server.h"

int
main(int argc, char *argv[])
{
    struct requestMsg req;
    struct responseMsg rsp;

    ssize_t msgLen;
    int serverId;
    int seqNum = 0;

    /* Create server message queue */

    serverId = msgget(MQ_KEY, IPC_CREAT | IPC_EXCL |
                            S_IRUSR | S_IWUSR | S_IWGRP);
    if (serverId == -1)
        errExit("srv msgget");

    for (;;) {
        msgLen = msgrcv(serverId, &req, REQ_MSG_SIZE, 1, 0);
        if (msgLen == -1) {
            if (errno == EINTR)         /* Interrupted by SIGCHLD handler? */
                continue;               /* ... then restart msgrcv() */
            errMsg("srv msgrcv");           /* Some other error */
            break;                      /* ... so terminate loop */
        }

        rsp.seqNum = seqNum;
        rsp.mtype = req.pid;
        printf("rcv msg from %d\n", req.pid);
    
        if (msgsnd(serverId, &rsp, RESP_MSG_SIZE, 0) == -1) {
            errMsg("srv msgsnd");
            break;            
        }

        seqNum += req.seqLen;
    }

    /* If msgrcv() or fork() fails, remove server MQ and exit */

    if (msgctl(serverId, IPC_RMID, NULL) == -1)
        errExit("srv msgctl");
    exit(EXIT_SUCCESS);
}
```

```c
// mq_seqnum_client.c
#include "mq_seqnum_server.h"


int
main(int argc, char *argv[])
{
    struct requestMsg req;
    struct responseMsg resp;
    int mqId;
    ssize_t msgLen;

    if (argc != 2 || strcmp(argv[1], "--help") == 0)
        usageErr("%s num\n", argv[0]);

    mqId = msgget(MQ_KEY, S_IWUSR);
    if (mqId == -1)
        errExit("msgget - server message queue");
    req.mtype = 1;                      /* send to server */
    req.pid = getpid();
    req.seqLen = atoi(argv[1]);


    if (msgsnd(mqId, &req, REQ_MSG_SIZE, 0) == -1)
        errExit("msgsnd");
    /* Get first response, which may be failure notification */

    msgLen = msgrcv(mqId, &resp, RESP_MSG_SIZE, getpid(), 0);
    if (msgLen == -1)
        errExit("msgrcv");

    printf("Receive seqNum %d\n", resp.seqNum);

    exit(EXIT_SUCCESS);
}

```


5. 给svmsg_file_client.c中的客户端在send和recv消息的时候添加timeout,避免被永久阻塞。

```c

#include "svmsg_file.h"

#define TIMEOUT 10
static int clientId;

static void /* SIGALRM handler: interrupts blocked system call */
handler(int sig)
{
    printf("msg Send timeout\n"); /* UNSAFE (see Section 21.1.2) */
}

static void
removeQueue(void)
{
    if (msgctl(clientId, IPC_RMID, NULL) == -1)
        errExit("msgctl");
}



void set_timeout(int num_secs) {
    struct sigaction sa;
    sa.sa_flags = 0;
    sigemptyset(&sa.sa_mask);   
    sa.sa_handler = handler;
    if (sigaction(SIGALRM, &sa, NULL) == -1)
        errExit("sigaction");
    alarm(num_secs);
}

void cancel_timeout() {
    int savedErrno;
    savedErrno = errno;
    alarm(0); /* Ensure timer is turned off */
    errno = savedErrno;
}


int
main(int argc, char *argv[])
{
    
    struct requestMsg req;
    struct responseMsg resp;
    int serverId, numMsgs;
    ssize_t msgLen, totBytes;


    if (argc != 2 || strcmp(argv[1], "--help") == 0)
        usageErr("%s pathname\n", argv[0]);

    if (strlen(argv[1]) > sizeof(req.pathname) - 1)
        cmdLineErr("pathname too long (max: %ld bytes)\n",
                (long) sizeof(req.pathname) - 1);

    /* Get server's queue identifier; create queue for response */

    serverId = msgget(SERVER_KEY, S_IWUSR);
    if (serverId == -1)
        errExit("msgget - server message queue");

    clientId = msgget(IPC_PRIVATE, S_IRUSR | S_IWUSR | S_IWGRP);
    if (clientId == -1)
        errExit("msgget - client message queue");

    if (atexit(removeQueue) != 0)
        errExit("atexit");

    /* Send message asking for file named in argv[1] */

    req.mtype = 1;                      /* Any type will do */
    req.clientId = clientId;
    strncpy(req.pathname, argv[1], sizeof(req.pathname) - 1);
    req.pathname[sizeof(req.pathname) - 1] = '\0';
                                        /* Ensure string is terminated */
    
    set_timeout(TIMEOUT);

    if (msgsnd(serverId, &req, REQ_MSG_SIZE, 0) == -1)
        errExit("msgsnd");    

    cancel_timeout();

    /* Get first response, which may be failure notification */

    set_timeout(TIMEOUT);

    msgLen = msgrcv(clientId, &resp, RESP_MSG_SIZE, 0, 0);

    cancel_timeout();

    if (msgLen == -1)
        errExit("msgrcv");

    if (resp.mtype == RESP_MT_FAILURE) {
        printf("%s\n", resp.data);      /* Display msg from server */
        exit(EXIT_FAILURE);
    }

    /* File was opened successfully by server; process messages
       (including the one already received) containing file data */

    totBytes = msgLen;                  /* Count first message */
    for (numMsgs = 1; resp.mtype == RESP_MT_DATA; numMsgs++) {
        msgLen = msgrcv(clientId, &resp, RESP_MSG_SIZE, 0, 0);
        // printf(resp.data);
        if (msgLen == -1)
            errExit("msgrcv");

        totBytes += msgLen;
    }

    printf("Received %ld bytes (%d messages)\n", (long) totBytes, numMsgs);

    exit(EXIT_SUCCESS);
}

```