---
title: "Linux/Unix系统编程手册-笔记11.系统和进程信息"
date: 2020-08-03T14:53:00+08:00
tags: ["Linux"]
categories: ["Linux系统编程手册阅读"]
---

现代Linux提供了/proc虚拟文件系统，开放了许多内核信息，允许进程方便的读取信息或者在某些情况下修改一些信息。/proc并不存在于磁盘上，存在于内存里。  

### /proc/PID

对于每个进程，根据其PID可在/proc找到对应的文件夹，其中文件status提供了进程许多信息，=以init为例，如下：

```sh
Name:   init
State:  S (sleeping)
Tgid:   1                               Thread group ID (traditional PID, getpid())
Pid:    1                               Actually, thread ID (gettid())
PPid:   0                               Parent process ID
TracerPid:      0                       PID of tracing process (0 if not traced)
Uid:    0       0       0       0       Real, effective, saved set, and FS UIDs
Gid:    0       0       0       0       Real, effective, saved set, and FS GIDs
FDSize: 10                              # of file descriptor slots currently allocated
Groups:                                 Supplementary group IDs
VmPeak: 0 kB                            Peak virtual memory size
VmSize: 8892 kB                         Current virtual memory size
VmLck:  0 kB                            Locked memory
VmHWM:  0 kB                            Peak resident set size
VmRSS:      308 kB                          Current resident set size
VmData: 0 kB                            Data segment size
VmStk:  0 kB                            Stack size
VmExe:  412 kB                          Text (executable code) size
VmLib:  0 kB                            Shared library code size
VmPTE:  0 kB                            Size of page table (since 2.6.10)
Threads:        2                       # of threads in this thread’s thread group
SigQ:   0/0                             Current/max. queued signals (since 2.6.12)
SigPnd: 0000000000000000                Signals pending for thread
ShdPnd: 0000000000000000                Signals pending for process (since 2.6)
SigBlk: 0000000000000000                Blocked signals
SigIgn: 0000000000000000                Ignored signals
SigCgt: 0000000000000000                Caught signals
CapInh: 0000000000000000                Inheritable capabilities
CapPrm: 0000001fffffffff                Permitted capabilities
CapEff: 0000001fffffffff                Effective capabilities
CapBnd: 0000001fffffffff                Capability bounding set (since 2.6.26)
Cpus_allowed:   f                       CPUs allowed, mask 0xf = 1111 (since 2.6.24)
Cpus_allowed_list:      0-3             Same as above, list format (since 2.6.26)
Mems_allowed:   1                       Memory nodes allowed, mask (since 2.6.24)
Mems_allowed_list:      0               Same as above, list format (since 2.6.26)
voluntary_ctxt_switches:        150     Voluntary context switches (since 2.6.23)
nonvoluntary_ctxt_switches:     545     Involuntary context switches (since 2.6.23)

```

/proc文件夹下的其他文件：

```sh
wsl@x:~$ ll /proc/1/
total 0
dr-xr-xr-x 7 root root 0 Aug  3 14:20 ./
dr-xr-xr-x 9 root root 0 Aug  3 14:20 ../
dr-x------ 2 root root 0 Aug  3 14:20 attr/
-r-------- 1 root root 0 Aug  3 14:20 auxv
-r--r--r-- 1 root root 0 Aug  3 14:20 cgroup
-r--r--r-- 1 root root 0 Aug  3 14:20 cmdline
-rw-r--r-- 1 root root 0 Aug  3 14:20 comm
lrwxrwxrwx 1 root root 0 Aug  3 14:20 cwd -> //         当前工作目录的符号链接
-r-------- 1 root root 0 Aug  3 14:20 environ           环境变量
lrwxrwxrwx 1 root root 0 Aug  3 14:20 exe -> /init*     被执行文件的符号链接
dr-x------ 2 root root 0 Aug  3 14:20 fd/               包含被此进程打开的文件的符号链接的文件夹
-rw-r--r-- 1 root root 0 Aug  3 14:20 gid_map           
-r--r--r-- 1 root root 0 Aug  3 14:20 limits
-r--r--r-- 1 root root 0 Aug  3 14:20 maps              memory mappings
-r--r--r-- 1 root root 0 Aug  3 14:20 mountinfo         
-r--r--r-- 1 root root 0 Aug  3 14:20 mounts            
-r-------- 1 root root 0 Aug  3 14:20 mountstats
dr-xr-xr-x 3 root root 0 Aug  3 14:20 net/              网络状态和socket信息
dr-x--x--x 2 root root 0 Aug  3 14:20 ns/
-rw-r--r-- 1 root root 0 Aug  3 14:20 oom_adj
-rw-r--r-- 1 root root 0 Aug  3 14:20 oom_score_adj
lrwxrwxrwx 1 root root 0 Aug  3 14:20 root -> //
-r--r--r-- 1 root root 0 Aug  3 14:20 schedstat
-rw-r--r-- 1 root root 0 Aug  3 14:20 setgroups
-r--r--r-- 1 root root 0 Aug  3 14:20 smaps
-r--r--r-- 1 root root 0 Aug  3 14:20 stat
-r--r--r-- 1 root root 0 Aug  3 14:20 statm
-r--r--r-- 1 root root 0 Aug  3 14:20 status
dr-xr-xr-x 4 root root 0 Aug  3 14:20 task/
-rw-r--r-- 1 root root 0 Aug  3 14:20 uid_map
```
关于/proc目录下每个项目的详细作用可以参考这个[文档](https://man7.org/linux/man-pages/man5/proc.5.html)。  



**/proc/PID/task**

此进程的所有线程会在此目录下拥有一个子目录/proc/PID/task/TID,里面的内容和/proc/PID内的类似。

其他的就用到时再详细了解了，/proc下的文件一般普通用户可读，但要改就需要root权限了。

### Exercise

