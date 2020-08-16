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
`lstat()`和`stat()`的区别在于如果文件属于符号链接， 那么所返回的信息针对的是符号链接本身（而非符号链接指向的文件）
