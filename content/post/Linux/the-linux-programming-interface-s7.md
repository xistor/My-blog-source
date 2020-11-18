---
title: "Linux/Unix系统编程手册-笔记7.用户和组"
date: 2020-07-27T10:53:00+08:00
tags: ["Linux"]
categories: ["Linux系统编程手册阅读"]
---


本章没啥好说的，比较有意思的的是crypt()函数，可以用来做用户身份鉴别：

```cpp
#define _XOPEN_SOURCE
#include <unistd.h>
char *crypt(const char *key, const char *salt);
    /* Returns pointer to statically allocated string containing
    encrypted password on success, or NULL on error */
```

crypt接受一个最长8个字符的密钥，并施之以数据加密算法（DES）的一种变体。slat参数指向一个两字符的字符串，用来扰动DES算法。。函数会返回一个指针，指向长度为13个字节的字符串。该字符串为静态分配的。内容即为经过加密处理的密码。和/etc/shadow里的加密密码记录做比较就能知道当前输入的密码是否正确。  

加密候选密码时，能够从已加密的密码(/etc/shadow)中获取salt值。在crypt()函数的参数salt只有前两位有意义。因此，可以直接将已加密密码指定为salt参数。

 ### Exercises
2. 首先这三个函数的功能是：  
- setpwent(): 重返密码文件起始处
- getpwent(): 从密码文件中逐条返回记录，当不再有记录或出错时，返回NULL。
- endpwent(): 关闭密码文件。

需要实现的getpwnam()的功能是: 获取指定登录名的密码记录信息。
其实就是书中介绍这几个函数的时给的示例代码段。

```cpp
#include <sys/stat.h>
#include <fcntl.h>
#include <pwd.h>
#include "tlpi_hdr.h"

struct passwd* ex_getpwnam(const char * name);

int main(int argc, char* argv[]) {
    struct passwd *pwd;
    if(argc != 2){
        printf("usage: %s login_name \n", argv[0]);
        errExit("arg");
    }

    pwd = ex_getpwnam(argv[1]);

    printf("%s\n", pwd->pw_name);
    printf("%s\n", pwd->pw_passwd);
    printf("%d\n", pwd->pw_uid);
    printf("%d\n", pwd->pw_gid);
    
}

struct passwd* ex_getpwnam(const char * name) {
    struct passwd *pwd;

    printf("name :%s", name);
    while((pwd = getpwent()) != NULL) {
        if(!strcmp(name, pwd->pw_name))
            return pwd;
    }
    endpwent();
    return NULL;
}
```