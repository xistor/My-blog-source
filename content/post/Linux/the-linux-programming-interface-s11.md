---
title: "Linux/Unix系统编程手册-笔记11.系统和进程信息"
date: 2020-08-09T16:57:00+08:00
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

2.绘制树状结构，展示系统中所有进程的父子关系。

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <dirent.h>
#include <limits.h>
#include <string.h>
#include <map>

#define MAX_LINE 1000

struct tree_node
{
    pid_t id;
    pid_t ppid;
    std::string cmd;
    std::vector<tree_node*> children;
};


void printTree(tree_node*, int);


int main() {

    DIR *dirp;
    struct dirent *dp;
    char path[PATH_MAX];
    char line[MAX_LINE], cmd_str[MAX_LINE];
    pid_t ppid;
    FILE *fp;
    char *p;
    bool gotname, gotpid, gotppid;

    dirp = opendir("/proc");
    if (dirp == NULL)
        return -1;

    /* Scan entries under /proc directory */


    std::map<pid_t, tree_node*> pmap;    // id > process 便于查找

    gotname = false;
    gotppid = false;

    for (;;) {
        errno = 0;              /* To distinguish error from end-of-directory */
        dp = readdir(dirp);
        if (dp == NULL) {
            if (errno != 0)
                return -1;
            else
                break;
        }

        /* Since we are looking for /proc/PID directories, skip entries
           that are not directories, or don't begin with a digit. */

        if (dp->d_type != DT_DIR || !isdigit((unsigned char) dp->d_name[0]))
            continue;

        snprintf(path, PATH_MAX, "/proc/%s/status", dp->d_name);

        fp = fopen(path, "r");
        if (fp == NULL)
            continue;           /* Ignore errors: fopen() might fail if
                                   process has just terminated */

        while (!gotname || !gotppid) {
            if (fgets(line, MAX_LINE, fp) == NULL)
                break;

            /* The "Name:" line contains the name of the command that
               this process is running */

            if (strncmp(line, "Name:", 5) == 0) {
                for (p = line + 5; *p != '\0' && isspace((unsigned char) *p); )
                    p++;
                // 去除最后的换行符
                char *pt = p;
                while(*pt != '\n' && *pt != '\0')
                    pt++;
                *pt = '\0';

                strncpy(cmd_str, p, MAX_LINE - 1);
                cmd_str[MAX_LINE -1] = '\0';        /* Ensure null-terminated */

            }

            if (strncmp(line, "PPid:", 5) == 0) {
                ppid = strtol(line + 6, NULL, 10);      /* Ensure null-terminated */
            }

        }

        tree_node *pinfo = new tree_node;
        pinfo->id = strtol(dp->d_name, NULL, 10);
        pinfo->cmd = std::string(cmd_str);
        pinfo->ppid = ppid;
        pmap[pinfo->id] = pinfo;

        fclose(fp);
    }

    // 构建进程树
    for (auto item : pmap) {
        if (pmap.count(item.second->ppid) > 0) {
            pmap[item.second->ppid]->children.push_back(item.second);
        }
    }
    // 打印
    printTree(pmap[1], 0);          
}


// 递归的打印出进程的所有子进程

void printTree(tree_node* node, int level) {

    for(int i = 0; i < level; i++)
    {
        printf("|");
        printf("     ");
    }

    printf("|--- ");
    printf("[%d]%s\n", node->id, node->cmd.c_str());
    
    for(auto i : node->children) {
        printTree(i, level + 1);
    }
}

```

运行结果：

```sh
|--- [1]systemd
|     |--- [367]systemd-journal
|     |--- [423]systemd-udevd
|     |--- [895]systemd-resolve
|     |--- [896]systemd-timesyn
|     |--- [985]accounts-daemon
|     |--- [986]acpid
|     |--- [989]avahi-daemon
|     |     |--- [1036]avahi-daemon
|     |--- [992]bluetoothd
|     |--- [994]cron
|     |--- [996]cupsd
|     |--- [997]dbus-daemon
|     |--- [998]NetworkManager
|     |--- [1005]irqbalance
|     |--- [1008]networkd-dispat
|     |--- [1014]polkitd
|     |--- [1018]rsyslogd
|     |--- [1023]nvidia-persiste
|     |--- [1024]snapd
|     |--- [1027]switcheroo-cont
|     |--- [1031]systemd-logind
|     |--- [1032]systemd-machine
|     |--- [1033]thermald
|     |--- [1034]udisksd
|     |--- [1035]wpa_supplicant
|     |--- [1072]cups-browsed
|     |--- [1079]ModemManager
|     |--- [1130]libvirtd
|     |--- [1131]unattended-upgr
|     |--- [1363]dnsmasq

...

```