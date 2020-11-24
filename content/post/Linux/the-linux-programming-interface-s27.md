---
title: "Linux/Unix系统编程手册-笔记27. 共享库"
date: 2020-11-23T18:30:00+08:00
tags: ["Linux"]
categories: ["Linux系统编程手册阅读"]
---


静态库是一个归档(archives),可以使用`ar`命令生成：

```sh
$ ar options archive object-file...
```

静态库会被整个拷贝到可执行文件中。


## 创建动态库

书中给的命令分两步

```sh
$ gcc -g -c -fPIC -Wall mod1.c mod2.c mod3.c
$ gcc -g -shared -o libfoo.so mod1.o mod2.o mod3.o
```

`-g`参数在可执行文件中保留源码级的debug和symbol信息。  
`-c`编译源码而不链接。
`-Wall`打开所有编译器警告。 
`-shared` 生成共享库。
`-fPIC` PIC意味着编译器生成Position Independent Code，f没有啥意义是历史遗留产物。位置无关代码意味着产生的机器码不会依赖一个特定地址才能正常工作，比如jump会使用相对地址而不是绝对地址。这样就允许产生的机器码可以被放到虚拟内存的任意位置。

## Real names, sonames, and linker names

共享库有时需要升级，升级就会引入不兼容等问题。所以需要使用版本号来管理，同一个主版本号内的库是兼容的，不同主版本号之间不兼容。
|名称|格式|描述|
|----|---|------|
|real name|libname.so.maj.min|库文件的真实名字，包含共享库的主次版本号|
|soname|libname.so.maj|包含主版本号，可能存在多个次要版本的real name,一般会被链接到最新版本的real name|
|linker name|libname.so|链接到最新的real name或者通常是链接到最新的soname,以允许在链接的时候，可以版本无关的链接|

## ldconfig

ldconfig 维护着`/etc/ld.so.cache`，它会检查每个主要版本库的小版本变化，并为每个soname更新或创建对应的软连接。一般系统启动的时候执行一次，安装新的库后需要手动执行一下。  

ldconfig 不会自动设置linker name,原因是：虽然一般来说会希望代码跑在最新的库上，但可能还是存在例外，想使用老版本的兼容库。所以ldconfig不会假定你的程序想使用哪个版本的库，安装者需要自己修改linker name的软连接。  

## 运行时符号解析

如果同一个全局符号（函数或变量），在可执行文件那和共享库中多个位置重复定义，或者在多个共享库中重复定义，如何解决符号引用？
书中举了如图的栗子，最终主函数中调到了可执行文件中的xyz()。
![解析符号引用](/img/the-linux-programming-interface-s27/resolve_symbol_reference.png)

- 在主程序中定义的全局符号会override在共享库中的定义
- 在多个共享库中重复定义，将会引用至第一个扫描到的库  

如果想指定调用共享库中的函数，生成共享库时可以使用`–Bsymbolic`参数。
```sh
gcc -g -shared -Wl,-Bsymbolic -o libfoo.so foo.o
```

## 指定使用静态库

链接时使用`-ldemo`时，若同时存在`libdemo.so`和`libdemo.a`，则共享库会被使用，若想指定使用静态库，可以
- 在gcc编译时，将静态库路径(包括.a)加在后面。
- 指定`-static`参数
- 使用`–Wl,–Bstatic`和`–Wl,–Bdynamic`来切换让链接器选择静态或动态库。

## 动态加载库

