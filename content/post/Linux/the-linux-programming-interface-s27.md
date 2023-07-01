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

{{< figure src="/img/the-linux-programming-interface-s27/resolve_symbol_reference.png" title="解析符号引用" class="center"  >}}



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

主要使用几个API, `dlopen()`、`dlsym()`，用法如下：

```cpp
#include <iostream>
#include <dlfcn.h>

int main() {

    void * libfoo;
    libfoo = dlopen("./libfoo.so", RTLD_NOW);
    
    if (libfoo == NULL) {
        printf("open error"); 
        return -1;
    }
    void (*func_print)();
    func_print = (void (*)())dlsym(libfoo, "xyz");

    if (func_print != NULL)
        (*func_print)();
    else 
        printf("sym error");

    dlclose(libfoo);
}

```

使用`dlopen()`打开对应的库文件，打开过程可以指定flag,打开成功会返回一个指针。之后使用`dlsym()`传入`dlopen()`返回的指针和符号名称，若找到了对应的函数或变量，会返回其地址，然后转换成合适的类型，就可以使用了。
- RTLD_DEFAULT: 默认顺序搜索符号。
- RTLD_NEXT: 根据共享对象的搜索顺序，从“当前对象”后搜索某个符号，返回该符号的地址。“当前对象”指的是，dlsym(RTLD_NEXT, "syscall");代码所在的对象。可以用来wrap系统函数。使用方法：`func = dlsym(RTLD_NEXT, “malloc”)`。这个选项我特么困惑了，一开始是编译出的可执行文件并没有依赖`gcc -l`后添加的库，貌似原因是gcc现在默认开启–as-needed选项，如果没有用到库就不会写到到可执行文件的依赖表中，所以编译时加了` -Wl,--no-as-needed`，之后程序可以运行，但表现和使用RTLD_DEFAULT选项并无不同，原因还待查。  

`dlopen()`和`dlclose()`在打开和关闭库的时候会有类似引用计数的机制，直到一个库的的handle计数为0才会实际上unload库。

## 控制符号可见性


一个设计良好的库应该只将其ABI中指定的符号（函数或变量）可见，因为:
- 若开放的未指明的接口被用户使用了，在库升级的时候带来兼容性问题。
- 在符号解析时，开放的符号可能会影响其他库。
- 开放没必要的符号会增大在必须在运行时载入的动态符号表。

以下几种方法可以用来控制符号的输出：

- 在C程序中，可以使用`static`关键字将符号限制在同一个源码文件中。
- GNU C 编译器提供了编译器特性属性可以实现和`static`类似的效果

```c
void
__attribute__ ((visibility("hidden")))
func(void) {
 /* Code */
}
```

- 版本脚本可以用来精确的控制符号的可见性。
- 当动态载入共享库时，`dlopen() RTLD_GLOBAL` flag可以用来指定共享库内定义的符号对其后来载入的库可见，`––export–dynamic`链接器选项可以用来将主程序中的全局变量对其动态载入的库可见。

## 链接器版本脚本

比如一个c文件
```c
// foo.c
#include <stdio.h>

void xyz(){
 printf("foo-xyz\n");
}

void func() {
 xyz();
}
```

有两个全局函数，编译后查看公开的动态符号如下：
```sh
$ readelf --syms --use-dynamic libfoo.so 

Symbol table of `.gnu.hash' for image:
  Num Buc:    Value          Size   Type   Bind Vis      Ndx Name
    7   0: 0000000000201028     0 NOTYPE  GLOBAL DEFAULT  22 _edata
    8   0: 0000000000201030     0 NOTYPE  GLOBAL DEFAULT  23 _end
    9   1: 0000000000201028     0 NOTYPE  GLOBAL DEFAULT  23 __bss_start
   10   1: 0000000000000520     0 FUNC    GLOBAL DEFAULT   9 _init
   11   2: 0000000000000670     0 FUNC    GLOBAL DEFAULT  13 _fini
   12   2: 000000000000065d    17 FUNC    GLOBAL DEFAULT  12 func
   13   2: 000000000000064a    19 FUNC    GLOBAL DEFAULT  12 xyz
```

如果我们只想公开func符号，可以像下面这样写版本脚本，global后的符号会被处理为可见，local后面的符号会被对外隐藏。

```
// foo.map

VER_1 {
    global:
        func;
    local:
        *;
};
```

编译命令：

```sh
$ gcc -g -c -fPIC -Wall foo.c
$ gcc -g -shared -o libfoo.so foo.o -Wl,--version-script,foo.map
```

查看新的so的符号，xyz已不可见：

```sh
$ readelf --syms --use-dynamic libfoo.so 

Symbol table of `.gnu.hash' for image:
  Num Buc:    Value          Size   Type   Bind Vis      Ndx Name
    7   0: 0000000000000000     0 OBJECT  GLOBAL DEFAULT ABS VER_1
    8   1: 00000000000005dd    17 FUNC    GLOBAL DEFAULT  13 func
```

若新的库内重新定义了func函数，但希望老的程序依然使用ver1版本的函数，需要如下定义，`@@`符号表示默认定义，一个符号只能有一个。

```
#include <stdio.h>

__asm__(".symver func_old,func@VER_1");
__asm__(".symver func_new,func@@VER_2");



void xyz() {
    printf("foo-xyz\n");
}


void func_old() {
    printf("func_old");
}

void func_new() {
    printf("func_new");
}
```

相应的版本脚本修改为：

```
VER_1 {
    global:
        func;
    local:
        *;
};

VER_2 {
    global: func;
            xyz;
    local:
        *;

} VER_1;
```

编译后查看符号有两个func，应该是两个不同的版本。不过好像没发现在编译时指定使用哪个版本的方法，只能在源文件中使用`@@`指定。VER_2最后的VER_1后缀表示：VER_2中没有定义的，会继承VER_1中的定义。

```sh
$ readelf --syms --use-dynamic libfoo.so 

Symbol table of `.gnu.hash' for image:
  Num Buc:    Value          Size   Type   Bind Vis      Ndx Name
    8   0: 0000000000000000     0 OBJECT  GLOBAL DEFAULT ABS VER_1
    9   1: 0000000000000000     0 OBJECT  GLOBAL DEFAULT ABS VER_2
   10   2: 00000000000006d5    24 FUNC    GLOBAL DEFAULT  13 func
   11   2: 00000000000006bd    24 FUNC    GLOBAL DEFAULT  13 func
   12   2: 00000000000006aa    19 FUNC    GLOBAL DEFAULT  13 xyz

```

## 初始化和析构函数

初始化函数在库被加载的时候被调用，析构函数在库被卸载的时候被调用。

有两种方式：
- 一种是使用gcc 构造和析构属性
```c
void __attribute__ ((constructor)) some_name_load(void)
{
 /* Initialization code */
}

void __attribute__ ((destructor)) some_name_unload(void)
{
 /* Finalization code */
}
```

- 第二种也是比较老的一种是使用`_init()`和`_fini()`函数。使用这两个函数需要在编译库时指定`gcc -nostartfiles`选项，以避免链接器生成默认的函数。

