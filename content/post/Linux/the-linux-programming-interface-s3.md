---
title: "Linux/Unix系统编程手册-笔记3.通用文件I/O"
date: 2020-07-05T16:01:00+08:00
tags: ["Linux"]
categories: ["Linux系统编程手册阅读"]
---

文件描述符：通常是一个小的非负整数，可以用来表示所有打开的文件。每个程序都会有三个标准文件描述符:

|File descriptor|Purpose|POSIX name|stdio stream|
|---------------|-------|-----------|------------|
|0|standard input|STDIN_FILENO|stdin|
|1|standard output|STDOUT_FILENO|stdout|
|2|standard error|STDERR_FILENO|stderr|

### 文件I/O操作的四个主要系统调用：

- fd = open(pathname, flags, mode), 函数打开pathname所标识的文件，并返回文件描述符， flags指定文件的打开方式。mode 指定创建文件的访问权限
如果open函数并未创建文件，则忽略mode参数。
- numread = read(fd, bnuffer, count), 从fd所指代的文件中读取count个字节到buffer中，numread为实际读取的字节数。
- numwritten = write(fd, buffer, count), 将buffer中的count个字节写入到fd所指代的文件中，numwritten为实际写入的字节数。
- status = close(fd), 释放文件描述符，以及与之相关的内核资源。


### 改变文件偏移量：

```cpp
#include<unistd.h>
off_t lseek(int fd, off_t offset, int whence);
```

whence参数表明应参照哪个基点来解释offset参数：
- SEEK_SET 将文件偏移量设置为从文件头部起始点开始的offset个字节。
- SEEK_CUR 相对于当前文件偏移量，将文件偏移量调整offset个字节。
- SEEK_END 将文件偏移量设置为起始于文件尾部的offset个字节

除了SEEK_SET情况下，offset必须为非负数，其他两种情况offset可正可负。 
lseek调用例子
```cpp
lseek(fd, 0, SEEK_SET); // start of file
lseek(fd, 0, SEEK_END); // Next byte after the end of the file
lssek(fd, -1, SEEK_END); // Last byte after the end
lseek(fd, -10, SEEK_CUR); //Ten bytes prior to current location
lseek(fd, 10000, SEEK_END); //10001 bytes past last byte of file
```

### 文件空洞 

看过之前的操作，会有一个疑问，文件偏移量跨越了文件末尾，再执行I/O操作，会发生什么？
read()将会返回0，write()函数可以在文件末尾写入数据。从原文件末尾到新写入数据之间的这段空间称为文件空洞。
读取空洞将会返回0填充的缓冲区。但是文件空洞不占用磁盘空间（对于大多数系统而言）。


通用I/O模型以外的操作： ioctl()

```cpp
#include<sys/ioctl.h>
int ioctl(int fd, int request, .../* argp */);

```

fd 参数为某个设备或文件已打开的描述符， request参数指定了将在fd上执行的控制操作。ioctl根据request的参数值来确定argp所期望的类型。通常情况下，argp是指向整数或结构的指针。对于未纳入标准I/O模型的所有设备和操作而言，ioctl()系统调用是个百宝箱。

### Exercise

1. 实现 tee -a file:

```cpp
#include <fcntl.h>
#include <unistd.h>
#include <iostream>

#define MAX_BUFFER 256
int tee(bool append, char* file_name);

int main(int argc, char* argv[]) {

    char opt;
    bool append = 0;
    char* file;


    while ((opt = getopt(argc, argv, "a:")) != -1) {
        switch (opt) {
        case 'a':
            append = 1;
            file = optarg;
            break;
        default: /* '?' */
            fprintf(stderr, "Usage: %s [-a file]\n",
                    argv[0]);
            exit(EXIT_FAILURE);
        }
    }

    tee(append, file);


}

int tee(bool append, char* file_name) {
    char buffer[MAX_BUFFER];
    int numread = 0;
    int outputfd;
    int openflag;

    if (append) {
        openflag = O_CREAT | O_WRONLY | O_APPEND;
    }
    else
    {
        openflag = O_CREAT | O_WRONLY;
    }
    
    outputfd = open(file_name, openflag, S_IRUSR | S_IWUSR);

    std::cout << "append:" << append << " file:" << file_name << std::endl;
    while ((numread = read(STDIN_FILENO, buffer, 256)) > 0)
    {
        buffer[numread] = '\0';
        printf("%s", buffer);
        write(outputfd, buffer, numread);
    }
    
}

```