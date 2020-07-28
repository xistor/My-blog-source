---
title: "Linux/Unix系统编程手册-笔记8.进程信任状态"
date: 2020-07-28T10:53:00+08:00
tags: ["Linux"]
categories: ["Linux系统编程手册阅读"]
---

每个进程有一组相关的UID和GID:
- 真实用户ID(real user ID)和组ID
- 有效用户ID(effective user ID)和组ID
- 保存的set-user-ID(saved set-user-ID)和set-group-ID
- 文件系统用户ID(file-system user ID)和组ID
- 补充组ID(supplementary group ID)

### real user ID 和 real group ID

ruid、rgid 由启动进程的用户决定，通常是登录用户，或者是继承自父进程。


### Effective User ID 和 Effective Group ID
在访问文件时，系统会检查进程的euid和egid，以判断进程是不是有权限访问。在一般情况下和ruid、rgid是一样的。但再两种情况下可以不一样,一种是使用setuid()函数改变uid,一种是执行set-user-id和set-group-id程序。


### Set-User-ID 和 Set-Group-ID Programs

一个set-user-ID 程序可以在其运行时以它的可执行文件的uid和gid运行。
比如典型的passwd,一般用户没有权限修改/etc/shadow, 所以passwd需要以root运行：

```
$ll /usr/bin/passwd
-rwsr-xr-x 1 root root 54256 Mar 27  2019 /usr/bin/passwd*
```

代表可执行权限的'x'变成了's'。然后我们执行'passwd'，并用'ps -al' 看一下'passwd'进程的uid。

```
F S   UID   PID  PPID  C PRI  NI ADDR SZ  WCHAN TTY          TIME CMD
0 S  1000     7     6  1  80   0 -  3782      - tty1     00:00:00 bash
0 S     0    34     7  0  80   0 -  3527      - tty1     00:00:00 passwd
...

```

可见'passwd'的uid并不是去执行它的用户的uid,而是其可执行文件的uid,也就是root。  
可以通过'chmod'给可执行文件设置  set-user-ID 和 set-group-ID bits

```sh
chmod u+s xxx       // Turn on set-user-ID permission bit        
chmod g+s xxx       // Turn on set-group-ID permission bit
```

### Saved Set-User-ID 和 Saved Set-Group-ID