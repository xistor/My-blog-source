---
title: "Linux/Unix系统编程手册-笔记35. Sockets"
date: 2021-01-12T18:05:00+08:00
tags: ["Linux"]
categories: ["Linux系统编程手册阅读"]
---

Socket系统调用：
- `socket()`创建一个新的socket
- `bind()`将socket绑定到一个地址。
- `listen()` 系统调用允许一个流socket接受来的链接。
- `accept()` 接受从listen的socket发来的链接请求
- `connect()` 与其他socket建立链接。


## 流式socket

![overview](/img/the-linux-programming-interface-s35/overview_socket.png)

类似我们生活中的电话通信。
一个典型的服务应用会创建一个监听socket, 绑定到一个地址上，然后通过accept socket的链接来处理客户端的请求。

## 数据报socket

类似我们生活中的信件通信。
![overview](/img/the-linux-programming-interface-s35/datagram_socket.png)

数据报socket虽然是无连接的，但是也可以使用connect(),连接后的socket可以使用write()或者send()直接发送到对端。只有会收到远端连接的那个socket发送的数据。connect()的作用是非对称的，只会影响调用connect()的socket,不会影响远端的socket。

## UNIX 域socket

socket 的域决定了其通信范围，有UNIX(AF_UNIX), IPv4(AF_INET), IPv6(AF_INET6)几种通信域。

UNIX域的socket地址数据结构如下：

```c
struct sockaddr_un {
    sa_family_t sun_family; /* Always AF_UNIX */
    char sun_path[108]; /* Null-terminated socket pathname */
};
```
sun_path指定socket名，bind()的时候会在此路径下创建一个文件。可以利用路径文件夹的权限来限制谁可以访问socket, 避免一些安全问题。  

`socketpair()`可以创建一个socket对,和管道类似，一般用于父子进程间的通信，socket对于其他进程并不可见。

```cpp
#include <stdio.h>
#include <unistd.h>
#include <sys/socket.h>
#include <string.h>

#define CHAR_BUFSIZE 50

int main(int argc, char **argv) {
    int fd[2], len;
    char message[CHAR_BUFSIZE];
    
    if(socketpair(AF_LOCAL, SOCK_STREAM, 0, fd) == -1) {
        return 1;
    }
    
    /* If you write to fd[0], you read from fd[1], and vice versa. */
    
    /* Print a message into one end of the socket. */
    snprintf(message, CHAR_BUFSIZE, "A message written to fd[0]");
    write(fd[0], message, strlen(message) + 1);
    
    /* Print a message into the other end of the socket. */
    snprintf(message, CHAR_BUFSIZE, "A message written to fd[1]");
    write(fd[1], message, strlen(message) + 1);
    
    /* Read from the first socket the data written to the second. */
    len = read(fd[0], message, CHAR_BUFSIZE-1);
    message[len] = '\0';
    printf("Read from fd[0]: %s \n", message); 
    
    /* Read from the second socket the data written to the first. */
    len = read(fd[1], message, CHAR_BUFSIZE-1);
    message[len] = '\0';
    printf("Read from fd[1]: %s \n", message);

}
```

Linux 抽象Socket命名空间：将sun_path字段的第一个字节指定为null字节（\0）就可创建抽象绑定，不会在文件系统上创建该名字。


## Internet域socket

Internet域的流式socket基于TCP,数据报socket基于UDP。

这段直接看书，不想写了...