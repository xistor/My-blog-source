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

void hello_cd(int fd, short event, void *arg)    //回调函数
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

    ev = event_new(base, -1, EV_PERSIST, hello_cd, event_self_cbarg());
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
    callback 参数是 `hello_cd()`,主要打印输出  
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



## 参考
https://libevent.org/