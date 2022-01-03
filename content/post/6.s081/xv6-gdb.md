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

ex.

在 Lab: page tables 中一开始报错如下：

```
panic: freewalk: leaf
```

这句报错出自vm.c 中的freewalk()函数：
```
// Recursively free page-table pages.
// All leaf mappings must already have been removed.
void
freewalk(pagetable_t pagetable)
{
// there are 2^9 = 512 PTEs in a page table.
for(int i = 0; i < 512; i++){
	pte_t pte = pagetable[i];
	if((pte & PTE_V) && (pte & (PTE_R|PTE_W|PTE_X)) == 0){
	// this PTE points to a lower-level page table.
	uint64 child = PTE2PA(pte);
	freewalk((pagetable_t)child);
	pagetable[i] = 0;
	} else if(pte & PTE_V){
	panic("freewalk: leaf");
	}
}
kfree((void*)pagetable);
}
```

注释中已经说了所有的叶子映射必须已经被移除。再搜一下，发现这个函数只在`uvmfree()`中被调用。由此，我们可以 `b uvmfree`设置断点,执行到断点后用`n` 不进入函数的单步执行。发现第一次调用到`freewalk()`就painc了。 `bt` 看一下backtrace,如下：

```
gdb) bt

|#0  uvmfree (pagetable=pagetable@entry=0x87f75000,
|    sz=sz@entry=4096) at kernel/vm.c:290
│#1  0x0000000080001044 in proc_freepagetable (
│    pasz=sz@entry=4096)
│    at kernel/proc.c:227
│#2  0x00000000800042c6 in exec (
│    path=path@einit",
│    argv=argv@entry=0x3fffffce00)
│    at kernel/exec.c:117
│#3  0x0000000080004e7e in sys_exec ()
│    at kernel/sysfile.c:444
│#4  0x000000008000203e in syscall ()
│    at kernel/syscall.c:154
│#5  0x0000000080001d28 in usertrap ()
│    at kernel/trap.c:67
│#6  0x0505050505050505 in ?? ()

```
显然是`proc_freepagetable()`函数中调用的`uvmfree()`，去看一下为什么会报错

```
void
proc_freepagetable(pagetable_t pagetable, uint64 sz)
{
  uvmunmap(pagetable, TRAMPOLINE, 1, 0);
  uvmunmap(pagetable, TRAPFRAME, 1, 0);
  uvmfree(pagetable, sz);
}
```
当看到`proc_freepagetable()`函数的内容时，原因已经昭然若揭了，之前添加map的USYSCALL没有unmap， 同样的`uvmunmap`一下，问题解决。

```
void
proc_freepagetable(pagetable_t pagetable, uint64 sz)
{
  	uvmunmap(pagetable, TRAMPOLINE, 1, 0);
  	uvmunmap(pagetable, TRAPFRAME, 1, 0);
	uvmunmap(pagetable, USYSCALL, 1, 0);
  	uvmfree(pagetable, sz);
}
```


## 参考

https://cs.brown.edu/courses/cs033/docs/guides/gdb.pdf  
https://sourceware.org/gdb/onlinedocs/gdb/TUI-Commands.html