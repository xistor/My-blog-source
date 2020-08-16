---
title: "Linux/Unix系统编程手册-笔记14.文件属性"
date: 2020-08-15T23:37:00+08:00
tags: ["Linux"]
categories: ["Linux系统编程手册阅读"]
---

## 获取文件信息

```cpp
#include <sys/stat.h>

int stat(const char *pathname, struct stat * statbuf);
int lstat(const char *pathname, struct stat * statbuf);
int fstat(int fd, struct stat *statbuf);
```

通过以上三个系统调用可以获取文件属性，它们大多数提取自i-node。  
`lstat()`和`stat()`的区别在于如果文件属于符号链接， 那么所返回的信息针对的是符号链接本身（而非符号链接指向的文件）,返回的stat结构有如下信息:

```cpp
struct stat {
    dev_t       st_dev;         // 文件所在的设备ID
    ino_t       st_ino;         // i-node number
    mode_t      st_mode;        // 文件类型和权限
    nlink_t     st_nlink;       // 硬链接到文件的个数
    uid_t       st_uid;         // 文件拥有者的用户ID
    gid_t       st_gid;         // 拥有者组ID
    dev_t       st_rdev;        // 设备文件的ID
    off_t       st_size;        // 文件大小（bytes）
    blksize_t   st_blksize;     // 文件进行I/O操作时的最优块大小（bytes）
    blkcnt_t    st_blocks;      // 分配给文件的总块数
    time_t      st_atime;       // 最后一次访问时间
    time_t      st_mtime;       // 最后修改时间
    time_t      st_ctime;       // 文件属性最后修改时间
}
```

- 设备ID和i节点号  
利用以上两者，可在所有文件系统中唯一标识某个文件。`st_dev`保存了文件存储的设备主、辅ID。而`st_rdev`是针对驱动的字符设备和块设备的主、辅ID。

- 文件大小、已分配块、及最优I/O块大小  
对于常规文件，st_size字段表示文件的字节数。对于符号链接，则表示链接所指路径名的长度。  
st_blocks字段表示分配给文件的总块数，块大小为512字节，其中包括了为指针块所分配的空间。  
st_blksize 字段指针对文件系统上文件进行I/O操作时的最优块大小（以字节为单位）。若I/O所采用的块小小于该值，则被视为低效。一般其值为4096。  

## 文件属主

文件创建时其用户ID取自进程的有效用户ID。其组ID则取自进程的有效组ID。有的时候需要某一目录下所有文件都属于某一特定组，可以为该组所有成员访问，这一需求现实中很实用，可以通过以下几种方式满足：

| mount 选项 | 有无设置父目录的Set-group-ID位 | 新建文件的组所有权取自何处
|:-----------:|:-----------:|:----------:|
| -o grpid或 -o bsdgroups| 忽略 | 父目录组ID|
| -o nogrpid或 -o sysgroups| 无 | 父目录组ID|
| （默认）| 有 | 父目录组ID|

改变文件属主时需要注意  
- 只有特权级进程（CAP_CHOWN）才能使用chown()改变文件的用户ID。非特权用户只能改变文件的组ID为进程所属的组ID中的一个，
- 如果文件的属主或属组发生了改变，那么set-user-ID和set-group-ID权限位也会随之关闭。这是为了防止：普通用户若能打开某一可执行文件的set-user-ID(或set-group-ID)位，然后再设法令其为某些特权级用户所拥有，就可能在执行该文件时获得特权用户身份。

## 文件权限

### 目录权限

目录权限对3种权限的含义另有所指

- 读权限： 可列出目录下的内容
- 写权限： 可在目录内创建、删除文件。
- 可执行权限： 可访问目录中的文件。  

若拥有目录的可执行权限，而无读权限，只要知道目录内文件的名称，仍可对其进行访问。

### 权限检查算法

只要在访问文件或目录的系统调用中指定了路径名称，内核就会检查相应文件的权限。如果赋予系统调用的路径名还包含目录前缀时，那么内核除去检查对文件本身所需的权限以外，还会检查前缀所含每个目录的可执行权限。一旦调用open()打开了文件爱你，针对文件描述符的后续系统调用（read()、write()、fstat()、fcntl()、以及mmap()）将不再进行任何权限检查。

### umask()

umask是一种进程属性，在进程创建文件和目录时umask会对这些设置进行修改。比如某进程的umask为八进制022(----w--w-)。其含义为此进程创建的文件或目录，对于同组或其他用户，应总是屏蔽写权限。  

系统调用umask()可以将进程umask改变为umask()参数所指定的值。

```c
#include <sys/stat.h>

mod_t umask(mode_t mask);
```

## Exercise

1. 
a) 根据权限检查的规则，内核会一次执行针对属主、属组以及其他用户的权限检查，进程的有效ID和文件的用户ID匹配则会根据文件的属主权限，授予进程相应的访问权限。所以将文件属主的所有权限剥夺后，即使其他用户可以访问文件，属主也无法访问文件。  
b) 见目录权限部分

