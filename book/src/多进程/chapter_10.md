# 进程通信方式——管道

## 什么是管道

管道是进程间通信的一种方式，它的本质其实是内核中的一块内存(或者说是内核缓冲区),这块区域的数据存储在一个环形队列,不过由于管道使用的是内核里面 的内存，所以我们无法对管道里面的数据进行直接操作，只能在管道的两端读/写数据。

## 管道的特点

- 管道数据是基于队列来维护的
- 管道对应的内核缓冲区大小的固定，默认4K
- 管道分为两部分:读端和写端(队列的两端),数据从写端进入管道，从端流出管道
- 管道的数据只能读一次,可以理解数据是在管道里面流动,由一端流向另一端.
- 对管道的操作(读,写)默认是阻塞的
  - 读管道：管道中没有数据，读操作被阻塞，当管道中有数据之后阻塞才能解除。
  - 写管道：管道被写满了，写数据的操作被阻塞，当管道变为不满的状态，写阻塞解除。
  - Linux的文件IO函数:

  ```cpp
  ssize_t read(int fd,void *buf,size_t count);
  ssize_t write(int fd,const void *buf,size_t count);
  ```

## 匿名管道

### 匿名管道的创建

匿名管道是管道的一种,匿名主要是指管道没有名字,但是本质是不变,它有管道的所有特性,匿名管道只能实现有血缘关系的进程间通信(父子进程,兄弟进程,爷孙进程,叔侄进程)

```cpp
#include <unistd.h>
int pipe(int pipefd[2]);
```

- 参数：传出参数，需要传递一个整形数组的地址，数组大小为 2，也就是说最终会传出两个元素
  - pipefd[0]: 对应管道读端的文件描述符，通过它可以将数据从管道中读出
  - pipefd[1]: 对应管道写端的文件描述符，通过它可以将数据写入到管道中
- 返回值：成功返回 0，失败返回 -1

### 进程间的通信

在讲解进程间通过管道的提醒具体内容之前，我们先来看一个简单的样例:

```cpp
#include <iostream>
#include <cstring>
#include <unistd.h>
#include <fcntl.h>
#include <errno.h>
#include <sys/wait.h>
#include <sys/types.h>

using namespace std;

int main()
{
    int fd[2] ;//定义两个文件描述符
    int ret=pipe(fd);  //创建管道
    if(ret==-1)
    {
        perror("pipe error");
    }
    int pid=fork();  //创建子进程
    if(pid==0)
    {
        close(fd[0]);
        /*这里子进程关闭读端,保留写端，这里我们希望将原来的在终端输出的内容改为输入到管道给父进程读取*/
        dup2(fd[1],STDOUT_FILENO);//将输出位置重定向到管道上
        execlp("ps","ps","aux",NULL);
        perror("execlp error");
    }
    else if(pid>0)  //执行父进程操作
    {
        close(fd[1]); //关闭写端
        char buff[1024];
        //读管道 
        //管道无数据——> read函数阻塞
        //管道有数据——> read停止阻塞(如果正处于阻塞状态),读取数据(若管道已满，写操作阻塞);
        while(true)
        {
            sleep(5);
            memset(buff,0,sizeof(buff));
            int len=read(fd[0],buff,sizeof(buff));
            if(len==0)  cout<<"pipe is empty"<<endl;
            cout<<"buff:"<<buff<<" "<<"len:"<<len<<endl;
        }
        wait(NULL); //等待子进程结束回收资源
    }
    return 0;
}
```

## 备注

- 子进程中执行shell命令相对于启动一个磁盘程序,因此需求使用execl()/execlp()函数
  - execlp("ps","ps","aux",NULL)
- 子进程中执行完shell命令直接就可以终端输出结果,这些消息传递父进程？
  - 进程数据传递需要管道，子进程需要将数据写入管道，父进程需要从管道中读取数据
  - 这里子进程想将终端输出结果写入管道，因此需要将标准输出重定向到管道
  - 这里父进程要等待子进程执行完shell命令释放子进程资源进而避免僵尸进程，因此需要使用wait()函数

## 有名管道

有名管道和匿名管道类似，但是有名管道可以用于无亲缘关系的进程间通信，之所以叫有名管道，是因为它在磁盘上有一个文件，类型为p，它的大小为0，因为它的内容还是存储在
内存缓存区，但是我们打开它可以获取文件蛮舒服，进而基于它们我们可以实现无血缘关系进程之间的通讯，同时由于有名管道式一个文件实体，所以我们创建文件时渝新欧下面两种:

- 基于shell命令创建

```shell
mkfifo filename
```

- 基于C语言创建

```c
#include<sys/types.h>
#include<sys/stat.h>
int mkfifo(const char *pathname, mode_t mode);
```

- 参数
  - pathname:文件名
  - mode:权限
- 返回值
  - 成功:0
  - 失败:-1

## 基于有名管道实现进程间通信

- 写进程

```cpp
// 有名管道的写端
#include <string>
#include <iostream>
#include <fcntl.h>
#include <unistd.h>
#include <cstdio>
#include <sys/stat.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <cerrno>
#include <cstdlib>
#include <string.h>

using namespace std;

int main()
{
    int wfd=open("./testfifo",O_WRONLY);
    if(wfd==-1)
    {
        perror("open");
        return -1;
    }
    cout<<"open success"<<endl;
    int i=0;
    while(i<100)
    {
        char str[1024];
        memset(str,0,sizeof(str));
        sprintf(str,"hello world %d\n",i);
        int wn=write(wfd,"hello world",strlen("hello world"));
        if(wn<0)
        {
            cerr<<"错误码:"<<errno<<",错误原因："<<strerror(errno)<<endl;
            break;
        }
        if(wn>0)
        {
            cout<<"write "<<wn<<" bytes"<<endl;
        }
        i++;
        sleep(1);
    }
    close(wfd);
    return 0;
}
```

- 读进程

```cpp
#include <iostream>
#include <fcntl.h>
#include <unistd.h>
#include <sys/stat.h>
#include <cstring>

using namespace std;

int main()
{
    int rfd = open("./testfifo", O_RDONLY);
    if (rfd == -1)
    {
        perror("open");
        return -1;
    }

    char buf[1024];

    while (true)
    {
        memset(buf, 0, sizeof(buf));
        int len = read(rfd, buf, sizeof(buf) - 1); // 减去1以防止覆盖结束符
        if (len >0)   cout<<"receive: "<<buf<<endl;
    }

    close(rfd); // 关闭文件描述符
    return 0;
}
```

## 管道描述符的修改

```cpp
// 管道操作眼需要创建对应两个文件描述符, 分别是管道的读端和写端，但是创建完管道后也可以将它们对应的属性进行修改(比如将阻塞改为非阻塞)，步骤如下:

// 1. 获取读端的文件描述符的flag属性
int flag = fcntl(fd[0], F_GETFL);
// 2. 添加非阻塞属性到 flag中
flag |= O_NONBLOCK;
// 3. 将新的flag属性设置给读端的文件描述符
fcntl(fd[0], F_SETFL, flag);
// 4. 非阻塞读管道
char buf[4096];
read(fd[0], buf, sizeof(buf));
```

**备注**:虽然可以修改但是一般不建议这么做，这里涉及的仅供参考。
