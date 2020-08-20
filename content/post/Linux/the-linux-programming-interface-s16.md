---
title: "Linux/Unix系统编程手册-笔记16.访问控制列表"
date: 2020-08-19T22:40:00+08:00
tags: ["Linux"]
categories: ["Linux系统编程手册阅读"]
---

访问控制列表即 ACL(access control list)是对传统UNIX文件权限模型的扩展，籍此可以在每个用户或每组的基础上来控制对文件的访问。  

一个ACL的结构如下图：



在shell中运行`getfacl`命令，可查看应用于文件的ACL。会得到类似如下输出：

```
getfacl README.md 
# file: README.md
# owner: x
# group: x
user::rw-
group::r--
other::r--
```

`setfacl`命令可用来修改文件的ACL。`setfacl -m`可修改现有ACE,或者当给定标记类型和限定符的ACE不存在时，会追加新的ACE记录。

```
#命令格式
setfacl -m u:[user]:[permission],g[group]:[permission] file

#执行效果
setfacl -m u:paulh:rx,g:teach:x tfile
getfacl --omit-hrader tfile
user::rwx
user:paulh:r-x
group::r-x
group:teach:--x
mask::r-x
other::--x
```

