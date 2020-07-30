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


### Set-User-ID 和 Set-Group-ID 程序

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

saved set-user-id 是设计来给set-user-ID程序使用的。当一个suid程序执行时，会发生以下几个步骤：
1.  如果可执行程序的set-user-id位为enble,然后effective user id 会被设置为可执行程序的拥有者的id。如果可执行程序的set-user-id位没有设置，就什么也不做。
2.  saved set-user-ID值是从程序的euid那拷贝过来的。无论set-user-id设置没。

set-user-ID程序可以设置它的euid在ruid和saved set-user-id之间切换。在这种情况下，程序可以暂时的drop(通过将euid切换到ruid)和regain(通过将euid切换到saved set-user-id)权限。


### 获取和修改进程信任状态

对于任何一个进程，可以在/proc/PID/status文件中查看今晨Uid,Gid和Groups.
举个栗子，如下。uid和gid行四个数字依次为real,effective,saved set和file system。

```
Name:   su
State:  S (sleeping)
Tgid:   71
Pid:    71
PPid:   36
TracerPid:      0
Uid:    1000    0       0       0
Gid:    0       0       0       0
FDSize: 3
Groups:
...
```

获取进程uid和gid的系统调用如下：

```cpp
#include <unistd.h>
uid_t getuid(void);
    // Returns real user ID of calling process
uid_t geteuid(void);
    // Returns effective user ID of calling process
gid_t getgid(void);
    // Returns real group ID of calling process
gid_t getegid(void);
    // Returns effective group ID of calling process
```

**修改effective id**

```cpp
#include <unistd.h>
int setuid(uid_t uid);
int setgid(gid_t gid);
    // Both return 0 on success, or –1 on error
```

setuid()分两种情况:
1. 当进程为非特权进程(effective user ID 不是0)，只有effective user ID会被setuid()改变,而且值只能在real user Id和saved set-user-ID之间切换。也就说对于非特权用户，setuid()仅在执行一个set-user-id程序时才有用。
2. 当一个特权进程执行setuid()并传入一个非0参数，real、effective、saved user ID都会设为参数值。一旦设置成功，进程将不能再把uid设置回0。

其他的就不详细做笔记了，主要看下面这个总结表格：
![总结](/img/the-linux-programming-interface-s8/summary_of_change_process_credentials.png)

