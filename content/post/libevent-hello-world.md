---
title: "libevent 学习（一)-Hello world"
date: 2024-03-16T16:43:16+08:00
tags: ["network"]
categories: ["network"]
---

记录下libevent的学习过程，主要是想理解下其设计思想。源码在[Github](https://github.com/libevent/libevent
)上, 下载后编译安装即可。


```sh
$ git clone git@github.com:libevent/libevent.git
$ cd libevent && mkdir build && cd build
$ cmake ..
$ make 
$ sudo make install
```

## 简单的例子

理解原理还为时过早，先从如何使用入手, 创建一个简单timer的例子，每1秒打印`Hello world！`，打印3次后退出：

``` c
#include <stdio.h>
#include <event2/event.h>
#include <time.h>

static int n_calls = 0;

void hello_cb(int fd, short event, void *arg)    //回调函数
{
    printf("Hello World! %d\n", n_calls++);
    struct event *me = (struct event*)arg;

    if (n_calls > 3)
       event_del(me);
}


int main()
{
    struct event_base *base = event_base_new();  //初始化libevent库
    struct timeval one_sec = { 1, 0 };
    struct event *ev;

    ev = event_new(base, -1, EV_PERSIST, hello_cb, event_self_cbarg());
    event_add(ev, &one_sec);
    event_base_dispatch(base);

    return 0;
}

```


编译：
```
$ gcc hello_world.c -levent
```

主要流程：

- 首先需要调用event_base_new创建event_base，event_base里可以包括多个event，通过poll来监控哪一个被激活。
- event_new() 创建了一个新的event，其函数原型如下：
    ```c
        struct event *event_new(struct event_base *base, evutil_socket_t fd, short events, event_callback_fn callback, void *callback_arg);
    ```
    fd在上面代码中是`-1`，代表event只能timeout激活或者手动调用event_active()激活，在这里就是timeout  
    events参数是`EV_PERSIST`, 代表除非调用`event_del()`删除event, 否则它会一直在  
    callback 参数是 `hello_cb()`,主要打印输出  
    callback_arg 参数传入的是`event_self_cbarg()`, 其作用是将event本身作为event_new()的参数，这是如何实现的？
        在源码中`event_self_cbarg()`定义为
    ```c
    void *
    event_self_cbarg(void)
    {
        return &event_self_cbarg_ptr_;
    }
    ```
    这个`event_self_cbarg_ptr_`指针是一个全局静态变量。
    ```c
    static void *event_self_cbarg_ptr_ = NULL;
    ```
    event_new()函数中为新的event分配了空间，没有看到`arg`指针的操作。
    ```c
    struct event *
    event_new(struct event_base *base, evutil_socket_t fd, short events, void (*cb)(evutil_socket_t, short, void *), void *arg)
    {
        struct event *ev;
        ev = mm_malloc(sizeof(struct event));
        if (ev == NULL)
            return (NULL);
        if (event_assign(ev, base, fd, events, cb, arg) < 0) {
            mm_free(ev);
            return (NULL);
        }

        return (ev);
    }
    ```
    进入event_assign()中发现会判断`arg`指针是否是`event_self_cbasrg_ptr_`，如果是就将`arg`赋值为ev。

    ```c
    event_assign(struct event *ev, struct event_base *base, evutil_socket_t fd, short events, void (*callback)(evutil_socket_t, short, void *), void *arg)
    {
        if (!base)
            base = current_base;
        if (arg == &event_self_cbarg_ptr_)
            arg = ev;
    ...
    ```

## event_base

event_base 结构体：保存了libevent的loop过程中的状态以及所需的各种信息, 定义在event-internal.h中。
每个成员变量都有注释。
```c

struct event_base {

    const struct eventop *evsel;    // 描述和后端的结构体(poll, epoll等)
...
	/** List of changes to tell backend about at next dispatch.  Only used
	 * by the O(1) backends. */
	struct event_changelist changelist; // 暂存add/delete等操作，减少系统调用

    /** Function pointers used to describe the backend that this event_base
	 * uses for signals */
	const struct eventop *evsigsel;
	/** Data to implement the common signal handler code. */
	struct evsig_info sig;


};

```

较为重要的是`evsel`, 它是一个`eventop`结构体，描述backend的行为，这个结构体类似C++中的基类。无论是poll或epoll只要实现接口，实现对应的函数就可以作为一个backend。

## socket echo server  

大概熟悉后，第二个例子稍微复杂点，使用libevent实现一个echo server。

```c
// server.c
#include <stdio.h>
#include <event2/event.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <errno.h>
#include <err.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>

#define SERVER_PORT 50000 // port

void client_cb(int fd, short event, void *arg)    //回调函数
{
    struct event *me = (struct event*)arg;
    printf("Got an event on socket %d:%s%s%s%s",
        (int) fd,
        (event&EV_TIMEOUT) ? " timeout" : "",
        (event&EV_READ)    ? " read" : "",
        (event&EV_WRITE)   ? " write" : "",
        (event&EV_SIGNAL)  ? " signal" : "");

    char buffer[1024];
    int len;

    if(event & EV_READ) {
        len = read(fd, buffer, sizeof(buffer));
        if(len == 0) {
            printf("Client disconnected.\n");
            close(fd);
            event_del(me);
            return;
        }
        len = write(fd, buffer, sizeof(buffer));
        printf(" rcv: %s\n", buffer);
    }
    

}

void on_accept(int fd, short event, void *arg)    //回调函数
{
    int client_fd;
    struct sockaddr_in client_addr;
    socklen_t client_len = sizeof(client_addr);
    struct event *ev;

    struct event_base *base = (struct event_base *) arg;

    /* Accept the new connection. */
	client_fd = accept(fd, (struct sockaddr *)&client_addr, &client_len);

    if(fcntl(client_fd, F_SETFL, fcntl(client_fd, F_GETFL) | O_NONBLOCK) < 0) {
            err(-1, " set O_NONBLOCK failed\n");
    }


	if (client_fd == -1) {
		warn("accept failed");
		return;
	}

    ev = event_new(base, client_fd, EV_READ | EV_WRITE | EV_ET | EV_PERSIST, client_cb, event_self_cbarg());

    event_add(ev, NULL);

    printf("new client\n");

}


int main()
{

    int listen_fd;
    struct sockaddr_in listen_addr;

    // 创建socket
    listen_fd = socket(AF_INET, SOCK_STREAM | SOCK_NONBLOCK, 0);

    if(listen_fd == -1) {
        err(-1, "socket");
    }

    memset(&listen_addr, 0, sizeof(struct sockaddr_in));

    listen_addr.sin_family = AF_INET;
	listen_addr.sin_addr.s_addr = INADDR_ANY;
	listen_addr.sin_port = htons(SERVER_PORT);

    // 监听端口
    if(bind(listen_fd, (struct sockaddr *) &listen_addr, sizeof(struct sockaddr_in)) == -1)
        err(-1, "bind");

    if (listen(listen_fd, SOMAXCONN) < 0) {
        err(-1, "listen");
    }

    struct event_base *base = event_base_new();
    struct event *ev;

    ev = event_new(base, listen_fd, EV_READ | EV_PERSIST, on_accept, base);

    event_add(ev, NULL);
    event_base_dispatch(base);

    return 0;
}
```


```c
// client.c
#include <stdio.h>
#include <sys/socket.h>
#include <stdlib.h>
#include <netinet/in.h>
#include <string.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <errno.h>
#include <err.h>

#define PORT 50000
 
int main(int argc, char const *argv[])
{
    struct sockaddr_in address;
    int sock = 0, valread;
    struct sockaddr_in serv_addr;
    char buffer[4096] = {0};

    if ((sock = socket(AF_INET, SOCK_STREAM, 0)) < 0)
    {
        err(-1, "\n Socket creation error \n");
        return -1;
    }
 
    memset(&serv_addr, '0', sizeof(serv_addr));
 
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(PORT);
     
    // Convert IPv4 and IPv6 addresses from text to binary form
    if(inet_pton(AF_INET, "127.0.0.1", &serv_addr.sin_addr)<=0)
    {
        err(-1, "\nInvalid address/ Address not supported \n");
        return -1;
    }

    if (connect(sock, (struct sockaddr *)&serv_addr, sizeof(serv_addr)) < 0)
    {
        err(-1, "\nConnection Failed \n");
        return -1;
    }



    send(sock , "hello world!" , strlen("hello world!") + 1, 0 );
    valread = read( sock , buffer, 1024);
    printf("%s\n",buffer );
    return 0;
}

```


运行效果大概是server会返回收到的字符，client会发送`hello world`后退出。

server代码中，首先创建了一个socket， 设置为NONBLOCK，这是I/O多路复用的常规操作。 
```c
    listen_fd = socket(AF_INET, SOCK_STREAM | SOCK_NONBLOCK, 0);
```

然后bind到指定的端口，并设置全连接队列长度为SOMAXCONN, 这个值可以通过`cat /proc/sys/net/core/somaxconn`查看。
```c
    if(bind(listen_fd, (struct sockaddr *) &listen_addr, sizeof(struct sockaddr_in)) == -1)
        err(-1, "bind");

    if (listen(listen_fd, SOMAXCONN) < 0) {
        err(-1, "listen");
    }
```

之后创建一个event_base，将server的socket fd放入event中监控， 我们只关心客户端发来的请求，所以只注册EV_READ。

```c
    struct event_base *base = event_base_new();
    struct event *ev;

    ev = event_new(base, listen_fd, EV_READ | EV_PERSIST, on_accept, base);

    event_add(ev, NULL);
    event_base_dispatch(base);
```

同时，event_new最后的自定义参数， 把base指针传进去，这样可以在`on_accept()`中把client的fd也add到同一个event_base中，一起监控起来。accept的client的fd也记得要设置成nonblock。

## 参考
https://libevent.org/