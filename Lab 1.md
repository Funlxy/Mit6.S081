# 笔记

## System Call 系统调用

| **系统调用**                            | **描述**                                                     |
| --------------------------------------- | ------------------------------------------------------------ |
| `int fork()`                            | 创建一个进程，返回子进程的PID。>0为父进程,=0为子进程,-1表示错误 |
| `int exit(int status)`                  | 终止当前进程，并将状态报告给wait()函数。无返回               |
| `int wait(int *status)`                 | 等待一个子进程退出; 将退出状态存入*status; 返回子进程PID。   |
| `int kill(int pid)`                     | 终止对应PID的进程，返回0，或返回-1表示错误                   |
| `int getpid()`                          | 返回当前进程的PID                                            |
| `int sleep(int n)`                      | 暂停n个时钟节拍                                              |
| `int exec(char *file, char *argv[])`    | 加载一个文件并使用参数执行它; 只有在出错时才返回             |
| `char *sbrk(int n)`                     | 按n 字节增长进程的内存。返回新内存的开始                     |
| `int open(char *file, int flags)`       | 打开一个文件；flags表示read/write；返回一个fd（文件描述符）  |
| `int write(int fd, char *buf, int n)`   | 从buf 写n 个字节到文件描述符fd; 返回n                        |
| `int read(int fd, char *buf, int n)`    | 将n 个字节读入buf；返回读取的字节数；如果文件结束，返回0     |
| `int close(int fd)`                     | 释放打开的文件fd                                             |
| `int dup(int fd)`                       | 返回一个新的文件描述符，指向与fd 相同的文件(**最低可用文件描述符**) |
| `int pipe(int p[])`                     | 创建一个管道，把read/write文件描述符放在p[0]和p[1]中。成功返回0，否则-1 |
| `int chdir(char *dir)`                  | 改变当前的工作目录                                           |
| `int mkdir(char *dir)`                  | 创建一个新目录                                               |
| `int mknod(char *file, int, int)`       | 创建一个设备文件                                             |
| `int fstat(int fd, struct stat *st)`    | 将打开文件fd的信息放入*st                                    |
| `int stat(char *file, struct stat *st)` | 将指定名称的文件信息放入*st                                  |
| `int link(char *file1, char *file2)`    | 为文件file1创建另一个名称(file2)                             |
| `int unlink(char *file)`                | 删除一个文件                                                 |

## 文件描述符 file descriptor

Linux 系统中，把一切都看做是文件（一切皆文件），当进程打开现有文件或创建新文件时，内核向进程返回一个文件描述符，文件描述符就是内核为了高效管理已被打开的文件所创建的索引，用来指向被打开的文件，所有执行I/O操作的系统调用都会通过文件描述符。

文件描述符 0、1、2默认被使用,按照惯例，`UNIX` 系统将 `fd 0` 对应进程的标准输入， `fd 1` 对应进程的标准输出， `fd 2` 对应进程的标准错误

1、一个进程能够同时打开多个文件，对应需要多个文件描述符，所以需要用一个文件描述符表对文件描述符进行管理；通常默认大小为1024，也即能容纳1024个文件描述符；

2、文件描述符表中0、1、2三个位置对应的文件描述符固定不变，标准输入、标准输出、标准错误；

3、当打开一个文件时，内核会自动在文件描述符表中寻找一个空闲且最小的文件描述符；

4、同一个文件可以被多次打开，但是每打开一次都需要一个新的文件描述符；

5、已经被占用的文件描述符在被释放后，可以重新被占用；

## 管道 Pipe

> A pipe is a small kernel buffer exposed to processes as a pair of file descriptors, one for reading and one for writing.

管道是一个小的内核缓冲区，作为一对文件描述符暴露给进程，一个用于读取，一个用于写入。

### 无名管道 Anonymous Pipe

无名管道是指以匿名管道的方式连接两个进程，管道在进程中创建，由文件描述符引用。创建管道的系统调用是pipe，管道的读取端和写入端分别对应两个文件描述符。其中，管道的读取端通过文件描述符0（即stdin）读取，而管道的写入端则通过文件描述符（即stdout）写入。这样，父子进程就可以通过无名管道来进行通信。无名管道只能传递**父子进程**间的数据，不能用于非亲缘关系的进程通信。

eg:

```bash
ls -al | grep .txt
```

上述命令中，管道连接了 ls -al 和 grep .txt 两个命令，将 ls -al 命令的输出作为 grep .txt 命令的输入，从而实现文件名搜索的功能。

### 有名管道 Named Pipe

有名管道是一种文件类型，它允许无关进程之间的通信（即非亲缘关系进程），跨越不同进程号和主机之间的通信。有名管道通过文件系统中的某一个文件名来建立，其它进程（不论是父子进程还是非亲缘关系进程）可以通过打开该文件来进行通信。使用有名管道时，需要先用`mkfifo`命令创建管道文件，然后通过文件 I/O 操作来进行数据的读写。有名管道可以在系统中存在较长时间，甚至跨越系统重启的时间，因此在使用完毕后，需要手动删除该文件。

eg:

```bash
// 创建有名管道
mkfifo mypipe

// 在一个进程中向管道中写入数据
echo "hello, world" > mypipe

// 在另一个进程中从管道中读取数据
cat < mypipe
```

上述代码中，通过 mkfifo 命令创建了一个名为 mypipe 的有名管道文件，在一个进程中向管道中写入了字符串 "hello, world"，而在另一个进程中从管道中读取了这个字符串。需要注意，这两个进程是没有任何父子关系的，它们是通过打开有名管道文件来进行通信的。

## I/O 重定向 Redirect

### 1 三种标准输入输出

1. 标准输入（STDIN），文件描述符号为：0，默认从键盘获取输入；
2. 标准输出（STDOUT），文件描述符号为：1，默认输出到显示终端；
3. 标准错误输出（STDERR）,文件描述符号为：2，默认输出到显示终端；

### 2 什么是重定向？如何重定向？

#### （1）什么是重定向？

IO 重定向是为了改变默认输入、输出的位置，如默认情况下标准输出（STDOUT），标准错误输出（STDERR）都是输出到显示终端，如对标准输出、标准错误输出改变其默认输出位置，可重定向输出到指定的文件中（实际工作中经常这么使用），要重定向就要配合一些语法符号。

#### （2）如何重定向？

- **Linux Shell 使用 " > " 和 ">>" 进行对文件描述符进行重定向**
- ">" # 使用本次输出内容替换原有文件的内容；
- ">>" 把本次输出追加到原文件的后面；
- **常见的一些输出重定向（标准输出和标准错误输出）表示**
- 【>】标准输出覆盖重定向
- 【>>】标准输出追加重定向
- 【2>】标准错误输出覆盖重定向
- 【2>>】标准错误输出追加重定向
- 【&>】将标准输出和标准错误输出都同时覆盖重定向
- 【&>>】将标准输出和标准错误输出都同时追加重定向

# 实验

### sleep

**编写代码/user/sleep.c**

```c
#include "kernel/types.h"
#include "user/user.h"
int main(int argc, char *argv[])
{
    // 参数个数不为2则报错
    if (argc != 2)
    {
        fprintf(2, "usage: sleep\n");
        exit(1); // 表示异常退出
    }
    int seconds = atoi(argv[1]); // 获得需要sleep的秒数
    sleep(seconds);
    exit(0);
}
```

**测试: 进入qemu**

```ba
$ sleep
usage: sleep
$ sleep 10 10
usage: sleep
$ sleep 10
```

**系统测试:**

报错 :/usr/bin/env: ‘python’: No such file or directory

表示找不到python环境,但发现我们其实已经安装了python

解决方法：使用软链接:
```bash
sudo ln -s /usr/bin/python3 /usr/bin/python
```

![image-20230423220523841](https://typora-picgo-picbed.oss-cn-shenzhen.aliyuncs.com/img/202304232205001.png)

### pingpong

管道并非属于进程资源，而是和套接字一样，属于操作系统资源。因此fork产生子进程后其实还是只有一个管道,它复制了用于管道I/O的文件描述符
		管道pipe()，在fork()之后调用就相当于两个进程分别创建两个管道，管道属性不同的话不能通信；在fork()之前调用，就相当于子进程复制了管道属性，两个管道属性是相同的，所以能通信。

```c
#include "kernel/types.h"
#include "user/user.h"
int main(int argc, char *argv[])
{
    int fd[2], pid;
    char buf[2];
    // 创建管道
    if (pipe(fd) < 0)
    {
        exit(1);
    }

    // 创建子进程
    pid = fork();

    // 父进程
    if (pid > 0)
    {
        int id = getpid();
        // 写
        write(fd[1], "0", 1);
        close(fd[1]);
        // 等待子进程退出
        wait(0);

        // 读
        read(fd[0], buf, 1);
        close(fd[0]);

        fprintf(2, "%d: received ping\n", id);
    }
    else if (pid == 0)
    {
        int id = getpid();
        // 读
        read(fd[0], buf, 1);
        close(fd[0]);
        fprintf(2, "%d: received pong\n", id);

        // 写
        write(fd[1], "1", 1);
        close(fd[1]);
    }
    exit(0);
}
```

### primes

```c
#include "kernel/types.h"
#include "user/user.h"

void primes(int lp[2]){ //传入参数为左管道
    close(lp[1]); //关闭左管道写
    int num;
    if(read(lp[0],&num,4)==0){ //读取数据完毕，退出
        exit(0);
    }
    printf("prime %d\n",num);
    
    int p[2];
    pipe(p);
    int data;
    while(read(lp[0],&data,4)){
        if(data%num){
            write(p[1],&data,4);
        }
    }
    close(lp[0]); //关闭左管道读
    int pid = fork();

    if(pid==0){
        primes(p);
    }else{
        close(p[0]);
        close(p[1]);
        wait(0);
    }
    exit(0);

}


int main(char argc, char **argv)
{
    int p[2], pid;
    pipe(p);
    // 将数字送入管道
    for (int i = 2; i <= 35; i++){
        write(p[1], &i, 4);
    }

    pid = fork();
    
    if(pid==0){ //子进程进行下一步处理
        primes(p);
    }else{
        close(p[0]);
        close(p[1]);
        wait(0);
    }
    exit(0);
}
```

### find

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"

void 
find(char *path, char *file)
{
  char buf[512], *p;
  int fd;
  struct dirent de;
  struct stat st;

  if((fd = open(path, 0)) < 0){
    fprintf(2, "find: cannot open %s\n", path);
    return;
  }

  if(fstat(fd, &st) < 0){
    fprintf(2, "find: cannot stat %s\n", path);
    close(fd);
    return;
  }
  strcpy(buf, path);
  p = buf+strlen(buf);
  *p++ = '/';
  while(read(fd, &de, sizeof(de)) == sizeof(de)){
    if(de.inum == 0)
      continue;
    memmove(p, de.name, DIRSIZ);
    p[DIRSIZ] = 0;
    if(stat(buf, &st) < 0){
      printf("find: cannot stat %s\n", buf);
      continue;
    }
    switch (st.type) {
    case T_DIR:
      if(strcmp(de.name, ".") != 0 && strcmp(de.name, ".."))
        find(buf, file);
    case T_FILE:
      if (strcmp(de.name, file) == 0) {
				printf("%s\n", buf);
			}
    }
  }
  close(fd);
  return;
}

int
main(int argc, char *argv[])
{
  /* 未指定目录, 则在当前目录下查找 */
  if(argc == 2){
    find(".", argv[1]);
  }
  else if(argc == 3){
    find(argv[1], argv[2]);
  }
  else{
    fprintf(2, "find: find <dir> <file>\n");
    exit(1);
  }
  exit(0);
}
```

### xargs

xargs.c中的main函数所接收到的argv，其实仅仅是xargs 后面的一些参数，比如说：`find . b | xargs grep hello`，在这个命令里边，xargs的main中接收到的argv就是 xargs grep hello,其中标准输入为(find . b)

```c
#include "kernel/types.h"
#include "user/user.h"
#include "kernel/param.h"
#define MSG_SIZE 512
int main(int argc, char** argv)
{
    // 获取标准输入, 即"|"前的命令
    char buf[MSG_SIZE];
    // 获得实际的命令
    char* xargv[MAXARG];
    // 获得命令参数,从1开始是去除xargs
    for(int i = 1 ; i < argc; i ++){
        xargv[i-1] = argv[i];
    }
    xargv[argc] = 0;
    int i,len;
    while(1){
        i = 0;
        while(1){
            len = read(0,&buf[i],1);
            if( len==0 || buf[i]=='\n') break;
            i ++;
        }
        if(i==0) break;// 说明读完了
        buf[i] = 0; //以0表示结尾
        xargv[argc-1] = buf; // 拼接命令
        if(fork()==0){ //子进程运行
            exec(xargv[0],xargv);
            exit(0);
        }else{
            // 等待子进程结束
            wait(0);
        }
    }
    exit(0);
}
```

