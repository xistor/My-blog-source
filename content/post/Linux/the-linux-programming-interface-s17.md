---
title: "Linux/Unix系统编程手册-笔记17.目录和链接"
date: 2020-08-20T18:25:00+08:00
tags: ["Linux"]
categories: ["Linux系统编程手册阅读"]
---

## 目录和硬链接

目录和文件在文件系统中大抵按照相同的方式存储，主要有两个不同点：
- 目录在i-node信息中会被标注成"目录"
- 目录是一个包含文件名和i-node号的表。

![目录和i-node之间的关系](/img/the-linux-programming-interface-s17/relationship_of_dir_inode.png)

由图可见以及文件系统章节关于i-node的内容可知，文件名并不存在于i-node中，而是保存在目录文件中。这样就允许不同的文件名映射到同一个i-node。使用`ln`创建硬链接就是如此。

```sh
$ touch abc
$ ln  abc xyz
$ ls -li abc xyz
2814749767789161 -rw-rw-rw- 2 wsl wsl 29 Aug 20 16:39 abc
2814749767789161 -rw-rw-rw- 2 wsl wsl 29 Aug 20 16:39 xyz
```
`ls -i` 可以显示文件的i-node。如上，两个文件的i-node号是一样的，因此它们对应的是同一个文件。i-node信息中还有一条记录了硬链接数，此时它的数目为2。`stat abc`命令可看

```
$ stat abc
  File: 'abc'
  Size: 29              Blocks: 0          IO Block: 4096   regular file
Device: 2h/2d   Inode: 2814749767789161  Links: 2
Access: (0666/-rw-rw-rw-)  Uid: ( 1000/     wsl)   Gid: ( 1000/     wsl)
Access: 2020-08-20 16:39:05.415709000 +0800
Modify: 2020-08-20 16:39:05.415709000 +0800
Change: 2020-08-20 16:43:03.445908500 +0800
```
使用`rm`命令删除abc文件后，再使用`stat xyz`看下, `Links`已变成了1。

```
k$ stat xyz
  File: 'xyz'
  Size: 29              Blocks: 0          IO Block: 4096   regular file
Device: 2h/2d   Inode: 2814749767789161  Links: 1
Access: (0666/-rw-rw-rw-)  Uid: ( 1000/     wsl)   Gid: ( 1000/     wsl)
Access: 2020-08-20 16:39:05.415709000 +0800
Modify: 2020-08-20 16:39:05.415709000 +0800
Change: 2020-08-20 17:05:25.726910300 +0800
 Birth: -
```

当rm从目录中删除一个文件名后，其对应的i-node如果Links变成0了，那么这个i-node和其指向的文件数据块就会被回收。  

所有的文件名（链接）都是平等的。只要有一个文件名存在，i-node就存在。

硬链接有两个限制：
- 因为目录条目使用i-node号区分文件，而i-number只有在同一个文件系统中才唯一，所以一个硬链接和它对应的文件必须在同一个文件系统内。
- 硬链接不能指向一个目录，这是为了防止无休止的循环。

## 符号链接Symbolic link（软连接）

先看张图：

![软链接](/img/the-linux-programming-interface-s17/soft_link.png)

符号链接是一种特殊的文件类型,它的数据是另外一个文件的**文件名**。软连接并不会包括在i-node中的Links数中，所以软连接指向的**文件名**删除后，软连接依然存在，只是在通过软连接访问时会提示`No such file or directory`。

因为符号链接是指向一个文件名，而不是一个i-number，所以它可以链接到一个在不同文件系统的文件。软连接也允许指向一个目录。

## 创建和移除硬链接

```cpp
#include <unistd.h>
int link(const char *oldpath, const char *newpath);
int unlink(const char *pathname);
```

`link()`系统调用将使用newpath指定的文件名创建一个硬链接指向oldpath文件名指向的文件。`link()`并不会解引用符号链接，所以如果oldpath是一个符号链接，会创建一个此符号链接文件的硬链接。  
`unlink()`用于删除一个链接，`unlink()`也不会解引用符号链接。

### 一个打开的文件只有在所有文件描述符关闭后才会被删除

如果有一个文件描述符还打开着，那这个实际上就不会被删除（当然链接数为0的打开着的文件，我们也不能通过文件名访问了）。这可以用在一个场景下：创建一个临时文件并打开，持有文件描述符，然后unlink()文件，这样这个临时文件就只能被我们自己进程使用了。当我们使用完后，close文件描述符后，文件会被删除。