---
title: "Linux/Unix系统编程手册-笔记17.目录和链接"
date: 2020-08-20T18:25:00+08:00
tags: ["Linux"]
categories: ["Linux系统编程手册阅读"]
---

## 目录和硬链接

目录和文件在文件系统中大抵按照相同的方式存储，主要有两个不同点：
- 目录在i-node信息中会被标注成"目录"
- 目录是一个包含文件名和i-node号的表。

![目录和i-node之间的关系](/img/the-linux-programming-interface-s17/relationship_of_dir_inode.png)

由图可见以及文件系统章节关于i-node的内容可知，文件名并不存在于i-node中，而是保存在目录文件中。这样就允许不同的文件名映射到同一个i-node。使用`ln`创建硬链接就是如此。

```sh
$ touch abc
$ ln  abc xyz
$ ls -li abc xyz
2814749767789161 -rw-rw-rw- 2 wsl wsl 29 Aug 20 16:39 abc
2814749767789161 -rw-rw-rw- 2 wsl wsl 29 Aug 20 16:39 xyz
```
`ls -i` 可以显示文件的i-node。如上，两个文件的i-node号是一样的，因此它们对应的是同一个文件。i-node信息中还有一条记录了硬链接数，此时它的数目为2。`stat abc`命令可看

```
$ stat abc
  File: 'abc'
  Size: 29              Blocks: 0          IO Block: 4096   regular file
Device: 2h/2d   Inode: 2814749767789161  Links: 2
Access: (0666/-rw-rw-rw-)  Uid: ( 1000/     wsl)   Gid: ( 1000/     wsl)
Access: 2020-08-20 16:39:05.415709000 +0800
Modify: 2020-08-20 16:39:05.415709000 +0800
Change: 2020-08-20 16:43:03.445908500 +0800
```
使用`rm`命令删除abc文件后，再使用`stat xyz`看下, `Links`已变成了1。

```
k$ stat xyz
  File: 'xyz'
  Size: 29              Blocks: 0          IO Block: 4096   regular file
Device: 2h/2d   Inode: 2814749767789161  Links: 1
Access: (0666/-rw-rw-rw-)  Uid: ( 1000/     wsl)   Gid: ( 1000/     wsl)
Access: 2020-08-20 16:39:05.415709000 +0800
Modify: 2020-08-20 16:39:05.415709000 +0800
Change: 2020-08-20 17:05:25.726910300 +0800
 Birth: -
```

当rm从目录中删除一个文件名后，其对应的i-node如果Links变成0了，那么这个i-node和其指向的文件数据块就会被回收。  

所有的文件名（链接）都是平等的。只要有一个文件名存在，i-node就存在。

硬链接有两个限制：
- 因为目录条目使用i-node号区分文件，而i-number只有在同一个文件系统中才唯一，所以一个硬链接和它对应的文件必须在同一个文件系统内。
- 硬链接不能指向一个目录，这是为了防止无休止的循环。

## 符号链接Symbolic link（软连接）

先看张图：

![软链接](/img/the-linux-programming-interface-s17/soft_link.png)

符号链接是一种特殊的文件类型,它的数据是另外一个文件的**文件名**。软连接并不会包括在i-node中的Links数中，所以软连接指向的**文件名**删除后，软连接依然存在，只是在通过软连接访问时会提示`No such file or directory`。

因为符号链接是指向一个文件名，而不是一个i-number，所以它可以链接到一个在不同文件系统的文件。软连接也允许指向一个目录。

**系统调用对符号链接的解释**  
这个表可在使用时查询下，其中记录了系统调用是否会去对符号链接解引用。
![系统调用是否解引用](/img/the-linux-programming-interface-s17/follow_link.png)

## 创建和移除硬链接

```cpp
#include <unistd.h>
int link(const char *oldpath, const char *newpath);
int unlink(const char *pathname);
```

`link()`系统调用将使用newpath指定的文件名创建一个硬链接指向oldpath文件名指向的文件。`link()`并不会解引用符号链接，所以如果oldpath是一个符号链接，会创建一个此符号链接文件的硬链接。  
`unlink()`用于删除一个链接，`unlink()`也不会解引用符号链接。

### 一个打开的文件只有在所有文件描述符关闭后才会被删除

如果有一个文件描述符还打开着，那这个实际上就不会被删除（当然链接数为0的打开着的文件，我们也不能通过文件名访问了）。这可以用在一个场景下：创建一个临时文件并打开，持有文件描述符，然后unlink()文件，这样这个临时文件就只能被我们自己进程使用了。当我们使用完后，close文件描述符后，文件会被删除。

## rename()

```c
#include <stdio.h>

int rename(const char *oldpath, const *newpath)
```
rename()既可以重命名文件，也可以将文件移至同一文件系统的另一目录。rename()调用仅操作目录条目（不会改变i-node），而不移动文件数据,这也是它仅能在同一文件系统移动文件的原因。


## 使用符号链接

```c
#include <unistd.h>

int symlink(const char *filepath, const char *linkpath);
ssize_t readlink(const char *pathname, char *buffer, size_t bufsiz);
```

symlink()系统调用会针对第一个参数filepath创建一个新的符号链接--linkpath。 由filepath所命名的目录或文件调用时无需存在。  
readlink():获取链接的本身内容，即其所指向的路径名，符号链接字符串的副本将放置于buffer所指向的字符数组中。

## 创建和移除目录

```c
#include <sys/stat.h>
#include <unistd.h>

int mkdir(const char *pathname, mode_t mode);
int rmdir(const char *pathname);
```

mkdir()系统调用所创建的仅仅是路径名中的最后一部分，换言之，`mkdir("aaa/bbb/ccc", mode)`仅当目录aaa和aaa/bbb已经存在的情况下才会成功。  
要是rmdir()调用成功，则要删除的目录必须为空。

## 移除一个文件或目录

```c
#include <stdio.h>

int remove(const char *pathname);
```

如果pathname是一个文件，那么remove()去调用unlink();如果pathname为一目录，那么remove()去调用rmdir()。  


## 读目录

```cpp
#include <dirent.h>

DIR *opendir(const char *dirpath);
DIR *fdopendir(int fd);
struct dirent *readdir(DIR *dirp);
void rewinddir(DIr *dirp);
int closedir(DIR *dirp);

int readdir_r(DIR *dirp, struct dirent *entry, struct dirent **result); // 可重入版的readdir()
```

opendir()和fdopendir()都会但会指向DIR类型结构的指针。该结构即所谓目录流。每调用readdir()一次，就会从dirp所指代的目录流读取下一目录条目。并返回一枚指针指向静态分配而得的dirent类型结构，每次调用readdir()都会覆盖该结构,读取完毕后返回NULL。  
rewinddir()函数可将目录流回移到起点。  
closedir()函数将dirp指代的目录流关闭。


## 文件树遍历

这个很实用。
```c
#define _XOPEN_SOURCE 500

#include <ftw.h>

int nftw(const char *dirpath, int (*func) (const char *pathname, const struct stat *statbuf, int typeflag, struct FTW *ftwbuf), int nopenfd, int flags );
```

`nftw()`遍历目录树时，最多会为树的每一层级打开一个文件描述符。参数`nopenfd`指定了`nftw()`可使用文件描述符数量的最大值。如果目录树深度超过这一最大值，那么nftw()会在做好记录的前提下，关闭并重新打开描述符。  

nftw()函数的flags参数修正函数的一些行为，详见 https://man7.org/linux/man-pages/man3/nftw.3.html  

nftw()为每个文件调用func时传递4个参数，pathnames是文件的路径名，是绝对路径还是相对路径取决与参数dirpath是何种路径。第二个参数statbuf是一枚指向stat结构的指针，内含该文件的相关信息。第三个参数typeflag提供了有关该文件的深入信息，详见上一个链接。第四个参数ftwbuf一枚结构体指针，所指向的结构体为

```c
struct FTW {
  int base;
  int level;
}
```
结构体中的`base`字段是指func函数中pathname参数内文件名部分的（最后一个“/”字符之后的部分）的整型偏移量。`level`字段是指该条目想对于遍历起点（其level为0）的深度。  

每次调用`func`必须返回一个整型值，若返回非0值，则`nftw()`立即停止遍历，对调用者返回相同的非0值。  
由于`nftw()`使用的数据结构是动态分配的，故而应用程序提前终止目录树遍历的唯一方法就是让`func`调用返回一个非0值，否则可能会引起内存泄漏等不可预期结果。  



## 进程的当前工作目录

进程的当前工作目录定义了该进程解析相对路径名的起点。新进程的当前工作目录继承自其父进程。

可以使用`getcwd()`获取进程的当前工作目录，一旦调用成功，当前路径会保存在cwdbuf所指向的缓冲区中，如果当前路径名长度超过size个字节，那么getcwd()会返回NULL。

```c
#include <unistd.h>
char *getcwd(char *cwdbuf, size_t size);
```

有两种用法：

```c
#define MAX_SIZE 255

int main(int argc, const char* argv[]){
    char path[MAX_SIZE];
    getcwd(path,sizeof(path));
    puts(path);  // puts is equal to print. In c++ we can use: cout << path << endl;
    return 0;
}
```
还有一种是cwdbuf参数为NULL,size参数传入0,则glibc会为getcwd()按需分配一个缓冲区，使用完我们需要自己释放掉

```c
int main(int argc, const char* argv[]){
    char* path;
    path = getcwd(NULL, 0);
    puts(path);
    free(path);
    return 0;
}

```

### 改变当前工作目录

```c
#include <unistd.h>

int chdir(const char *pathname);
```

```c
#define _XOPEN_SOURCE 500
#include <unistd.h>

int fchdir(int fd);
```

## 针对目录文件描述的相关操作

![使用文件描述符的系统调用](/img/the-linux-programming-interface-s17/use_fd.png)

这些系统调用参数中都有一个文件描述符，当打开相对路径时，会以传入的文件描述符为参照点，之所以要支持这些系统调用，原因有二：

- 当调用open()或其他使用路径的系统调用的同时，如果pathname目录前缀的某些部分发生了改变，就可能导致竞争，想要避免可以针对目标目录打开一个文件描述符，然后将该描述符传给上述的`xxxat()`。
- 工作目录是进程的属性之一，为进程的所有线程共享，而对某些应用程序而言，需要针对不同线程拥有不同的‘虚拟’工作目录。将`xxxat()`与应用所维护的目录文件描述符号，就可以模拟出这一功能。


## 改变进程的根目录

每个进程都有一个根目录，该目录是解释绝对路径(即那些以/开始的目录)时的起点。默认情况，这是文件系统的根目录。想改变需要为特权(CAP_SYS_CHROOT)进程调用chroot()。

```c
#define _BSD_SOURCE

#include <unistd.h>

int chroot(const char* pathname);
```

调用`chroot()`后，用户将受困于文件系统中新根目录下的子树中，`/..`是`/`的一个链接，所以目录到`/`后再执行命令`cd ..`时，用户依然会待在同一目录下。  

对于无特权程序，需要注意：
- 调用`chroot()`后，通常应调用一次`chdir("/")`改变当前工作目录，否则进程就可能使用相对路径访问外面的文件。

- 如果进程针对监禁区的某一目录持有一打开的文件描述符，那么结合fchdir()和chroot()即可越狱成功，比如

```c
int fd;

fd = open("/", O_RDONLY);
chroot("/home/xx");
fchdir(fd);
chroot("."); /*out of jail*/
```

## 解析路径名

```c
#include <stdlib.h>

char *realpath(const char *pathname, char *resolved_path);
```

`realpath()`将对pathname中的所有符号链接和`/.``/..`解析,得到的绝对路径放置于resolved_path指向的缓冲区中。

## 解析路径名字字符串

```c
#include <libgen.h>

char *dirname(char *pathname);
char *basename(char *pathname);
```

`dirname()`和`basename()`函数将一个路径名字符串分解成目录和文件名两部分。比如给定路径名`/home/britta/prog.c`,`dirname()`将返回`/home/britta`,而`basename()`将返回`prog.c`。

## Exercise

1.根据提示，在编译前后使用`ls -li`查看，发现可执行文件的i-number改变了，所以实际行为为原可执行文件的在目录中的记录被删除了，但其i-node在其执行期间还存在，只不过新建了一个新文件以及i-node信息，文件名指向了新的i-number。  
3.Github上找到了一个小哥写的[程序](https://github.com/timjb/tlpi-exercises/blob/master/realpath_clone.c)，花了点时间看懂了，加了些注释。

```cpp
#include <limits.h>
#include <unistd.h>
#include <libgen.h>
#include <sys/types.h>
#include <sys/stat.h>
#include "tlpi_hdr.h"

char *my_realpath_relative (const char *, char *);
char *my_realpath(const char *, char *);

char *my_realpath_relative (const char *path, char *relative_path) {
  
  // path为绝对路径的话， relative_path先赋值为“/”
  // 这个relative_path也就是上层调用传入的resolved_path，也就是最终调用者拿到的真实路径所在的缓冲区指针
  // 所以对其修改都会影响到最终解析出的路径

  if (path[0] == '/') {
    relative_path[0] = '/';
    relative_path[1] = '\0';
  }

  int i = 0, j = 0;
  // 遍历path
  while (path[i] != '\0') {

    /*
      i 指向一层目录的起始处，j 指向一层目录结尾 '/'或 '\0'
      比如传入路径为"/home/x/work/a/b/c"， 在第一次循环时i指向'h', j 指向"home"之后的'/'字符
      这样就把一层目录分割出来，每次循环前进一层目录，直至字符串末尾
    */

    while (path[i] == '/') { i++; }
    j = i;
    while (path[j] != '/' && path[j] != '\0') {
      j++;
    }


    // 当 path 为 "/"时会出现 j == i
    if (j > i) {

      /*
        接下来处理单层目录，分了三种情况
      */
      if (j - i == 1 && path[i] == '.') {
        /*  当前处理目录字符串是 "."， 什么也不做 */
      } else if (j - i == 2 && path[i] == '.' && path[i+1] == '.') {
        /* 当前处理的目录字符串是 ".."， 调用dirname()，取得上层目录路径，并取代relative_path中原先的路径 */

        char *dirn = dirname(relative_path);
        strncpy(relative_path, dirn, PATH_MAX);
      } else {
        if (relative_path[strlen(relative_path)-1] != '/') {
          strncat(relative_path, "/", PATH_MAX);
        }

        /* 
          记录当前relative_path内路径字符串的长度，后面有用
          比如传入路径为"/home/x/work/a/b/c"， 当前处理到"work"这层，此时"work"还没有被添加到relative_path中，
          所以得到的为"/home/x/"的长度， relative_path_len 为8
          后续都以"work"为例
        */
        size_t relative_path_len = strlen(relative_path);

        /* 把当前处理的目录append到relative_path的末尾，那么relative_path执行完下面这句后为
          "/home/x/work/"
        */
        strncat(relative_path, &path[i], j-i+1);
        struct stat stat_buf; 
        if (lstat(relative_path, &stat_buf) == -1) { return NULL; }
        /*
          如果当前是一个符号链接的话，则调用readlink()读取符号链接内的字符串，
          并将relative_path恢复，比如当前处理的"work"是一个符号链接，则将"work"从relative_path中删除，
          也就是说relative_path变成了"/home/x/", 当然从符号链接中取出的路径还是要继续解析的，所以还需要递归的
          调用my_realpath_relative。这里又有两种情况：
          1. 符号链接里的是绝对路径：那么my_realpath_relative函数一开始会判断并将relative_path重置为"/",
          使用符号链接里的路径继续解析，比如"work"指向的路径为"/a/b/c"，一个前缀完全不一样的绝对路径，则再次进入
          my_realpath_relative后，relative_path被重置为"/",然后按a->b->开始重新解析。
          2. 符号链接里的是相对路径：那么relative_path中保存路径还是有用的，则my_realpath_relative会在relative_path
          后面添加新的解析后的路径。
        */
        if (S_ISLNK(stat_buf.st_mode)) {
          char buf[PATH_MAX];
          ssize_t len = readlink(relative_path, buf, PATH_MAX);
          if (len == -1) { return NULL; }
          relative_path[relative_path_len] = '\0';
          if (my_realpath_relative(buf, relative_path) == NULL) {
            return NULL;
          }
          // 至此，递归结束后，relative_path中已经为"work"在文件系统中的绝对路径。
        }
      }
    }
    /*
      进入下一层目录
    */
    i = j;
  }

  return relative_path;
}

char *my_realpath(const char *path, char *resolved_path) {
  if (path == NULL) {
    errno = EINVAL;
    return NULL;
  }

  if (resolved_path == NULL) {
    resolved_path = (char *) malloc(PATH_MAX);
  }

  // 如果是相对路径的话就先获取当前工作目录，作为前缀加到resolved_path中
  if (path[0] != '/') {
    if (getcwd(resolved_path, PATH_MAX) == NULL) {
      return NULL;
    }
    strncat(resolved_path, "/", PATH_MAX);
  }

  /* 
    解析路径 
    当参数path是相对路径，resolved_path已经有了其绝对路径前缀，后续解析在它之后添加就可以了
    当参数path是绝对路径，则resolved_path内还是空的，继续看my_realpath_relative函数
  */
  return my_realpath_relative(path, resolved_path);
}

int main (int argc, char *argv[]) {
  if (argc != 2) {
    usageErr("%s path", argv[0]);
  }
  char *path = argv[1];
  char resolved_path[PATH_MAX];
  if (my_realpath(path, resolved_path) == resolved_path) {
    printf("%s", resolved_path);
  } else {
    errExit("my_realpath\n");
  }
  exit(EXIT_SUCCESS);
}
```

7.实现nftw()

```



```