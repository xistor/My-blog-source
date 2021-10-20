---
title: "Linux/Unix系统编程手册-笔记16.访问控制列表"
date: 2020-08-19T22:40:00+08:00
tags: ["Linux"]
categories: ["Linux系统编程手册阅读"]
---

访问控制列表即 ACL(access control list)是对传统UNIX文件权限模型的扩展，籍此可以在每个用户或每组的基础上来控制对文件的访问。访问控制列表是利用文件扩展属性实现的。  

## ACL的结构
如下图:
![acl](/img/the-linux-programming-interface-s16/an_access_control_list.png)

每条记录由3部分组成：
- 标记类型：标记该记录作用于一个用户、组，还是其他类别的用户。
- 标记限定符号： 某个用户的ID或组ID
- 权限

**带OBJ后缀的项**
对于应传统的用户权限  

**ACL_MASK**
该值记录了可由 ACL_USER、ACL_GROUP_OBJ以及ACL_GROUP型ACE所能授予的最高权限。这一项设立的目的在于即使运行并无ACL概念的应用程序，也能保障其行为的一致性。当ACL包含标记类型为ACL_MASK的ACE时：
- 调用chmod()对传统组权限所做的变更，会改变ACL_MASK（而非ACL_GROUP_OBJ）
- 调用stat(),在st_mode字段的组权限位中会返回ACL_MASK权限。

但是正是如此，假设某文件设置了如下ACL

```
user::rw-, group::---,mask::---,other::r--
```
若针对该文件执行chmod g+rw，则ACL将会变为

```
user::rw-, group::---,mask::rw-,other::r--
```
这时组用户仍无法访问该文件。迂回策略是修改针对组的ACE,赋予其所有权限，那么组用户将会获得和ACL_MASK一样的权限。


## shell 命令
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
setfacl -m u:[user]:[permission],g:[group]:[permission] file

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

## 默认型ACL

前面讨论的都是访问型ACL，即当进程访问与该ACL相关的文件时，将使用访问型ACL来判定进程对文件的访问权限。针对目录有第二种ACL：默认型ACL。

访问目录时，默认型ACL并不参与判定所授予的权限，相反，默认型ACL的存在与否决定了在目录下所创建的文件或子目录的ACL和权限。
比如创建一个文件夹，并给设置默认型ACL，然后在此文件夹下新建文件

```
mkdir test
setfacl -d -m u::rwx,u:x:rx,g::rx,g:x:rwx,o::- test
getfacl test
# file: test
# owner: x
# group: x
user::rwx
group::rwx
other::r-x
default:user::rwx
default:user:x:r-x
default:group::r-x
default:group:x:rwx
default:mask::rwx
default:other::---

cd test
echo "hello" >> hello

getfacl hello     
# file: hello
# owner: root
# group: root
user::rw-
user:x:r-x			#effective:r--
group::r-x			#effective:r--
group:x:rwx			#effective:rw-
mask::rw-
other::---

```
可以看到新建的文件继承了目录的默认ACL。一旦目录拥有默认型ACL,那么对于新创建于该目录下的文件来说，进程的unmask并不参与判定文件访问型ACL中所记录的权限。

## ACL API

![ACL库函数及数据结构](/img/the-linux-programming-interface-s16/ACL_library_function_and_datastrures.png)

ACL 的API比较繁杂，如图，需要首先从文件中将ACL读入内存中，然后从中依次get每条记录，从ACE中读取/修改标记类型、获取/修改标记限定符、获取/修改权限分别对应下面三组函数。

