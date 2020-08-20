---
title: "Linux/Unix系统编程手册-笔记15.扩展属性"
date: 2020-08-17T15:25:00+08:00
tags: ["Linux"]
categories: ["Linux系统编程手册阅读"]
---


## 概述
文件的扩展属性(extended attributes, EAs), 允许我们给文件添加自定义的键值对属性，可以以此记录文件版本、MMIE类型信息、或文件字符编码等。  


扩展属性根据功能的不同分为四类，叫做命名空间(namespace)，分别为user、trusted、system和security。
- user 扩展属性：可以被非特权进程操作，但要读取的话需要对文件有读权限，写属性需要对文件有写权限。  
- trusted: 进程要有(CAP_SYS_ADMIN)才能操作
- system: 给内核使用，目前仅支持访问控制列表
- security: 
* 给操作系统的安全模块存储安全标签
* 将可执行文件和能力(capability)关联起来
* 一开始为了支持SELinux而设计的


## 设置和获取扩展属性

在shell中可以使用sefattr和getfattr设置和获取扩展属性，比如我们要给文件test在user空间设置一个x属性，可以使用如下命令：
```sh 
touch test      # 新建文件test
setfattr -n user.x -v "The past is not dead." test  #设置扩展属性 -n 后面跟key, -v 后面跟value
getfattr -n user.x test    #获取user命名空间的扩展属性x
```

其他命令详细--help查看。  
代码中可以使用系统调用

```cpp
#include <sys/types.h>
#include <sys/xattr.h>

/***** 设置扩展属性 *****/
int setxattr(const char *path, const char *name,
                const void *value, size_t size, int flags);
int lsetxattr(const char *path, const char *name,
                const void *value, size_t size, int flags);
int fsetxattr(int fd, const char *name,
                const void *value, size_t size, int flags);

/***** 获取扩展属性 *****/
ssize_t getxattr(const char *path, const char *name,
                void *value, size_t size);
ssize_t lgetxattr(const char *path, const char *name,
                void *value, size_t size);
ssize_t fgetxattr(int fd, const char *name,
                        void *value, size_t size);

/***** 删除扩展属性 *****/
int removexattr(const char *pathname, const char *name);
int lremovexattr(const char *pathname, const char *name);
int fremovexattr(int fd, const char *name);


```

还是常见的系统调用三剑客，使用详见 https://man7.org/linux/man-pages/man2/setxattr.2.html 及 https://man7.org/linux/man-pages/man2/getxattr.2.html  

## 扩展属性的限制

### user EA的限制

user EA 只能用于文件和目录。原因有以下几点：
- 对于符号链接，所有权限对所有用户都是打开的，而user EA需要考虑权限所以相互矛盾
- 对于设备文件、套接字以及FIFO而言，授予用户权限，意在对其针对底层对象所执行的i/o操作加以控制，如果想使用这些权限来控制user EAs的操作，二者可能会出现权限冲突。

### EA在实现方面的限制

Linux VFS 针对所有文件系统上的EA均施以如下限制
- EA名称的长度不能超过255个字节
- EA值的容量为64KB
在莫写文件系统对可与文件挂钩的EA数量及其大小还有更为严格的限制。
- 在ext2、ext3及ext4文件系统上，与一文件关联的所有EA命名和EA值的总字节数不会超过单个逻辑磁盘块的大小：1024字节、2048字节、或4096字节。
- 在JFS上，为某一文件所使用的所有EA名和EA值的总字数上限为128KB。