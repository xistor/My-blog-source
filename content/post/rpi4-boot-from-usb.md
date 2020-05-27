---
title: "树莓派4B USB硬盘启动"
date: 2020-03-08T22:03:16+08:00
tags: ["raspberrypi"]
categories: ["raspberrypi"]
---

树莓派4B 官方还没有提供从USB启动的方案，不过可以使用之前更改cmdline.txt的方法实现。  
我的树莓派已经使用SD卡跑了挺长时间了，不想重新配置，下面这种方式可以无痛将系统从SD卡迁移到硬盘（迁移之后SD卡依然需要）。
系统运行速度上会有提升，感觉不错，当然据说可能会有莫名其妙的问题，用全新系统更稳一些，这个看个人选择了。

### 1.硬盘分区格式化
将有系统的SD卡和要用到的硬盘都插到树莓派上。我的是一枚4T WD紫盘。  
由于MBR不支持超过2T的硬盘，使用GPT分区, /dev/sda为此硬盘
```bash
$sudo parted

Using /dev/sda
Welcome to GNU Parted! Type 'help' to view a list of commands.、

(parted) print

Model: External USB 3.0 (scsi)
Disk /dev/sda: 4001GB
Sector size (logical/physical): 512B/4096B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system  Name  Flags

(parted) rm 1 # 若之前存在分区，且可以删掉，则使用rm [Number] 命令删除

(parted) mkpart LVM ext4 0% 10% # 作为树莓派系统分区

(parted) mkpart LVM ext4 10% 100%  # 留作其他

(parted) print

Model: External USB 3.0 (scsi)
Disk /dev/sda: 4001GB
Sector size (logical/physical): 512B/4096B
Partition Table: gpt
Disk Flags

1      1049kB  400GB   400GB   ext4         LVM
2      400GB   4001GB  3601GB  ext4         LVM

(parted) quit
```
分给树莓派10%的空间作为系统分区，大概360G。parted 使用quit命令退出后分区即生效。  

还需要格式化分区：

```bash
$sudo mkfs.ext4 /dev/sda1

$sudo mkfs.ext4 /dev/sda2

$sudo fdisk -l

Disk /dev/sda: 3.7 TiB, 4000787030016 bytes, 7814037168 sectors
Disk model: USB 3.0         
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: gpt
Disk identifier: C492CEA3-9C21-40D4-AFB0-23F887E360E9

Device         Start        End    Sectors   Size Type
/dev/sda1       2048  781404159  781402112 372.6G Linux filesystem
/dev/sda2  781404160 7814035455 7032631296   3.3T Linux filesystem
```

### 2.系统迁移
挂载硬盘：
```bash
$sudo mkdir /media/sys
$sudo mount /dev/sda1 /media/sys
```

拷贝系统文件：
```bash
$sudo rsync -avx / /media/sys
```

查看sda1分区PARTUUID:
```bash
$blkid

/dev/sda1: UUID="ec074432-b020-44a7-8cb2-ce84d45296ab" TYPE="ext4" PARTLABEL="LVM" PARTUUID="18deaf7e-9704-44d2-9e63-a2c60358b26b"
```
记下PARTUUID。  

修改cmdline.txt文件
```bash
$sudo nano /boot/cmdline.txt

root=PARTUUID=18deaf7e-9704-44d2-9e63-a2c60358b26b # 此处修改为sda1的PARTUUID
```
保存退出。重启。  

挂载sda2分区，4T硬盘还有另外一个分区需要开机挂载。
```bash
$sudo mkdir /media/disk
$sudo nano /etc/fstab
```
最后新加一行  
UUID为sda2的UUID, blkid可看， nofail最好加上，否则挂载失败会导致启动不起来。
```bash
UUID=fdddd773-1ce7-430f-b4ff-3df3ac26494a /media/disk ext4 defaults,auto,users,rw,nofail 0 0
```
保存退出，重启。

```bash
$df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/root       366G  7.9G  340G   3% /
devtmpfs        1.6G     0  1.6G   0% /dev
tmpfs           1.7G     0  1.7G   0% /dev/shm
tmpfs           1.7G  9.5M  1.7G   1% /run
tmpfs           5.0M  4.0K  5.0M   1% /run/lock
tmpfs           1.7G     0  1.7G   0% /sys/fs/cgroup
/dev/mmcblk0p1  253M   53M  200M  21% /boot
/dev/sda2       3.3T   89M  3.1T   1% /media/disk
tmpfs           348M     0  348M   0% /run/user/1000
```
可见系统目录 / 已经是366G了，/dev/sda2也自动挂载到/media/disk了。

若/media/disk目录无权限,则
```bash
$sudo chown pi:pi /media/disk
```

### 3.参考
https://www.youtube.com/watch?v=FM9wuFLufyA  
https://blog.hqcodeshop.fi/archives/273-GNU-Parted-Solving-the-dreaded-The-resulting-partition-is-not-properly-aligned-for-best-performance.html