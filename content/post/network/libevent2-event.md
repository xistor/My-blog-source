---
title: "libevent 学习（二)-event"
date: 2024-05-12T12:12:16+08:00
tags: ["network"]
draft: true
categories: ["network"]
---

写过两个例子之后，libevent的基本操作就是以下几行代码。下面深入到libevent代码中，以event为线索，看看是如何存储以及操作event的。

```c
    struct event_base *base = event_base_new();
    struct event *ev;

    ev = event_new(base, listen_fd, EV_READ | EV_PERSIST, on_accept, base);

    event_add(ev, NULL);
    event_base_dispatch(base);
```

## event结构体

一个event结构体，在我们看来最基本保存的就是两个：event的触发来源-文件描述符`ev_fd`，和处理event的操作-`event_callback`。

```c
struct event {
	struct event_callback ev_evcallback;

	/* for managing timeouts */
	union {
		TAILQ_ENTRY(event) ev_next_with_common_timeout;
		size_t min_heap_idx;
	} ev_timeout_pos;
	evutil_socket_t ev_fd;

	short ev_events;
	short ev_res;		/* result passed to event callback */

	struct event_base *ev_base;

	union {
		/* used for io events */
		struct {
			LIST_ENTRY (event) ev_io_next;
			struct timeval ev_timeout;
		} ev_io;

		/* used by signal events */
		struct {
			LIST_ENTRY (event) ev_signal_next;
			short ev_ncalls;
			/* Allows deletes in callback */
			short *ev_pncalls;
		} ev_signal;
	} ev_;


	struct timeval ev_timeout;
};


```
在event中的fd、callback等内容则会从event_new()的参数中得来。
其他的还有一个尾队列ENTRY指针和一个双向链表ENTRY指针，可以看出多个event是会放在一个链表中管理。找到这个链表需要去看`event_add()`函数。

```

```





## 参考

https://www.cnblogs.com/fuzidage/p/14482501.html