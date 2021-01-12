---
title: "Linux/Unix系统编程手册-笔记35. Sockets"
date: 2020-01-12T18:05:00+08:00
tags: ["Linux"]
categories: ["Linux系统编程手册阅读"]
---

Socket系统调用：
- `socket()`创建一个新的socket
- `bind()`将socket绑定到一个地址。
- `listen()` 系统调用允许一个流socket接受来的链接。
- `accept()` 接受从listen的socket发来的链接请求
- `connect()` 与其他socket建立链接。

