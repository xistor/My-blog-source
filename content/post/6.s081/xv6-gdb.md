---
title: "qemu xv6 使用GDB调试"
date: 2022-01-01T21:26:16+08:00
tags: ["6.s081"]
categories: ["6.s081"]
---


Fall 2021 6.S081中视频还是20年的，用的GDB调试的部分没有看到比较清除的配置过程，整理了下配置过程。

## 1. riscv64-unknown-elf-gdb

2020 Lecture 5 视频中用的`riscv64-unknown-elf-gdb`，实际在tool page 中已经安装了`gdb-multiarch`,估计在21课程中已经改用这个了，用这个就好了， 如果没有就`sudo apt install gdb-multiarch`。

## 2. gdbinit

1. gdbinit脚本包含了gdb启动的时候自动运行的命令，lab已经提供了.gdbinit（内容在下面），所以需要在lab目录下执行gdb-multiarch。


```
set confirm off // 关闭确认，就是操作时Y/N的确认提示
set architecture riscv:rv64 // 设置架构为riscv:rv64
target remote 127.0.0.1:26000 // 2600端口号是make qemu-gdb时随机产生并写入.gdbinit的，可能不同，见makefile
symbol-file kernel/kernel // 指定symbol文件
set disassemble-next-line auto  // 自动反汇编后面要执行的代码
set riscv use-compressed-breakpoints yes

```

2. 将xv6-labs-2021/.gdbinit文件所在的路径写到`～/.gdbinit`中
```
	add-auto-load-safe-path /home/x/xv6-labs-2021/.gdbinit
```

## 3. 调试

在一个终端`make qemu-gdb` 之后，在另一个终端的lab目录内执行`gdb-multiarch`，
其他的根据视频中的来就好了。

![gdb-debug](/img/xv6-gdb/gdb-debug.png)

## 参考

https://cs.brown.edu/courses/cs033/docs/guides/gdb.pdf  
https://sourceware.org/gdb/onlinedocs/gdb/TUI-Commands.html