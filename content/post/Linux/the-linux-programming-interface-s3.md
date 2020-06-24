---
title: "Linux/Unix系统编程手册-笔记3.文件I/O"
date: 2020-06-23T16:01:00+08:00
tags: ["Linux"]
categories: ["Linux系统编程手册阅读"]
---

文件描述符：通常是一个小的非负整数，可以用来表示所有打开的文件。每个程序都会有三个标准文件描述符:

|File descriptor|Purpose|POSIX name|stdio stream|
|---------------|-------|-----------|------------|
|0|standard input|STDIN_FILENO|stdin|stdin|
|1|standard output|STDOUT_FILENO|stdin|stdout|
|2|standard error|STDERR_FILENO|stdin|stderr|
