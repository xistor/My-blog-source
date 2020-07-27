---
title: "Linux/Unix系统编程手册-笔记7.用户和组"
date: 2020-07-27T10:53:00+08:00
tags: ["Linux"]
categories: ["Linux系统编程手册阅读"]
---


本章没啥好说的，比较有意思的的是crypt()函数，可以用来做用户身份鉴别：

```cpp
#define _XOPEN_SOURCE
#include <unistd.h>
char *crypt(const char *key, const char *salt);
    /* Returns pointer to statically allocated string containing
    encrypted password on success, or NULL on error */
```