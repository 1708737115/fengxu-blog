# IO多路复用 —— epoll

## 前言

在之前我们就已经介绍过了select和poll,在作为io多路复用的最后一个的epoll,我们来总结一下它们之间的区别:

## select

#### 实现原理

select 通过一个文件描述符集合（fd_set）来工作，该集合可以包含需要监控的文件描述符。调用者会指定一个超时时间，如果在这个时间内没有任何描述符准备好，则函数返回。select 可以同时监听读、写和异常三种类型的事件。

#### 优点

select 函数调用和实现比较简单，同时它支持跨平台

#### 缺点

1. select是基于位图这一数据结构来存储与遍历文件描述符相关的信息，由于位图数据结构的限制，select最多能同时监听1024个文件描述符，如果超过1024个文件描述符,无法实现大规模的并发处理。

2. 每次调用select，需要拷贝位图，而且select属于用户态，网络通信属于内核态，需要拷贝两次，会影响select的性能。

3. select的每次监听都需要遍历一整个位图，随着需要监听的socket增加，性能会大大下降。

## poll

#### 实现原理

poll 通过一个链表来存储需要监控的文件描述符，当文件描述符就绪时，链表中的节点会被移动到就绪链表中，当链表为空时，poll会阻塞。poll与select类似，也是通过一个文件描述符集合来工作，但是poll所使用的数据结构是一个结构体数组，它的结构如下:

```cpp
struct pollfd
{
    int fd; //存储的socket
    short events; // socket触发的事件
    short revents; // 返回的事件
}
```

#### 优点

 poll 函数调用和实现简单，同时它支持跨平台，同时相对于select，poll没有了1024的限制，可以实现对更多文件描述符的监听。

#### 缺点

 poll监视的连接数没有1024的限制，但是随着socket的增多，poll的效率会降低，无法处理超大规模并发。

## epoll

#### epoll的原理

epoll 全名是 eventpoll,是Linux内核2.5.44 版本之后才出现的一个事件通知机制。它属于IO多路复用技术的一种形式,IO多路复用技术指的是一个操作里同时监听多个输入输出源,在其中一个或多个输入输出源可用的时候返回,然后对其进行读写操作。上面的select和poll都具有一个通病,那就是都有性能瓶颈,而epoll可以承受百万级别的连接，属于select和poll模式的升级版。

#### epoll的优点

- select和poll监听都是线性检测的，而epoll是基于红黑树来管理文件描述符,对于事件的发生，它不像select和poll那样需要遍历整个文件描述符集合，而是通过回调函数来通知，所以epoll的效率更高.

- select和poll在工作中需要对集合进行判断来看哪些文件描述符已经就绪，而epoll则不需要，epoll通过回调函数来通知，所以epoll的效率更高。

当我们需要监听大量的文件描述符时，epoll的效率会更高，所以epoll是当前最常用的IO多路复用技术。

## epoll的操作函数

在Linux内核中，主要给我们提供了一下三个函数来操作epoll:

```cpp
#include <sys/epoll.h>
//创建一个epoll实例,通过红黑树来管理文件描述符
int epoll_create(int size);
//管理红黑树中的文件描述符(添加,修改,删除)
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
//等待文件描述符就绪
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

#### epoll_create

epoll_create函数用于创建一个epoll实例，参数size表示要监听的文件描述符的数量，但是这个参数在Linux 2.6.8之后已经没有意义了，因为epoll的红黑树可以动态扩展，所以大于0即可。

```cpp
int epfd = epoll_create(0);
```

 返回值:

- 成功:返回epoll实例的文件描述符
- 失败:返回-1,并设置errno

#### epoll_ctl

epoll_ctl函数用于管理红黑树中的文件描述符，参数epfd表示epoll实例的文件描述符，参数op表示要进行的操作，参数fd表示要监听的文件描述符，参数event表示要监听的事件。

在epoll中有以下和事件相关的数据结构

```cpp
typedef union epoll_data {
  void        *ptr;
 int          fd; // 通常情况下使用这个成员, 和epoll_ctl的第三个参数相同即可
 uint32_t     u32;
 uint64_t     u64;
} epoll_data_t;

struct epoll_event {
 uint32_t     events;      /* Epoll events */
 epoll_data_t data;        /* User data variable */
};

```

而在epool_ctl函数中主要要注意下面几个参数；

```cpp
int epoll_ctl(int epfd,int op,int fd,struct epoll_event *event);
```

- `epfd`:epoll实例的文件描述符
- `op`:要进行的操作
  - `EPOLL_CTL_ADD`:添加文件描述符
  - `EPOLL_CTL_MOD`:修改文件描述符
  - `EPOLL_CTL_DEL`:删除文件描述符
- `fd`:要监听的文件描述符
- `event`:epoll事件,用来修饰fd,指定检测什么事件
  - `events`:事件类型
    - `EPOLLIN`:读事件
    - `EPOLLOUT`: 写事件
    - `EPOLLERR`: 错误事件
-
  - `data`:用户数据,通常情况下使用fd即可

#### epoll_wait

epoll_wait函数用于等待事件的发生，参数epfd表示epoll实例的文件描述符，参数events表示要监听的事件，参数maxevents表示最多监听多少个事件，参数timeout表示等待时间。

```cpp
int epoll_wait(int epfd,struct epoll_event *events,int maxevents,int timeout);
```

- `epfd`:epoll实例的文件描述符
- `events`:要监听的事件
- `maxevents`:最多监听多少个事件
- `timeout`:等待时间
  - `0`：函数不阻塞，不管epoll实例中有没有就绪的文件描述符，函数被调用后都直接返回
  - `大于0`：如果epoll实例中没有已就绪的文件描述符，函数阻塞对应的毫秒数再返回
  - `-1`：函数一直阻塞，直到epoll实例中有已就绪的文件描述符之后才解除阻塞
  
## epoll的使用

这里为实现了一个简单的基于epoll实现的服务端与客户端通讯，大家可以自己测试一下:

```cpp

//server.cpp
#include <iostream>
#include <unistd.h>
#include <ctype.h>
#include <stdlib.h>
#include <sys/stat.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <sys/epoll.h>
#include <string.h>

int main(int argc,char* argv[])
{
    if(argc!=3)
    {
        std::cout<<"命令行参数数量不对"<<std::endl;
        std::cout<<"./demo port timeout"<<std::endl;
        exit(-1);
    }
    // 创建套接字
    int lfd=socket(AF_INET,SOCK_STREAM,0);
    if(lfd<0)
    {
        perror("create socket");
        return -1;
    }

    //绑定
    struct sockaddr_in server_addr;
    memset(&server_addr,0,sizeof(server_addr));
    server_addr.sin_port=htons(atoi(argv[1]));
    server_addr.sin_family=AF_INET;
    server_addr.sin_addr.s_addr=htonl(INADDR_ANY);

    //设置端口复用
    int opt=1;
    setsockopt(lfd,SOL_SOCKET,SO_REUSEADDR,&opt,sizeof(opt));

    //绑定端口
    int ret=bind(lfd,(struct sockaddr*)&server_addr,sizeof(server_addr));
    if(ret<0)
    {
        perror("bind");
        exit(-1);
    }

    //监听
    ret=listen(lfd,10);
    if(ret<0)
    {
        perror("listen");
    }

    //创建epoll实例
    int epfd=epoll_create(100);
    if(epfd<0)
    {
        perror("epfd");
        exit(-1);
    }

    struct epoll_event ev;
    ev.data.fd=lfd;
    ev.events=EPOLLIN;
    ret=epoll_ctl(epfd,EPOLL_CTL_ADD,lfd,&ev);
    if(ret<0)
    {
        perror("epoll_ctl");
    }

    struct epoll_event evs[1024];
    int size=(sizeof(evs)/sizeof(struct epoll_event));

    while(1)
    {
        int num=epoll_wait(epfd,evs,size,atoi(argv[2]));
        for(int i=0;i<num;i++)
        {
            int fd=evs[i].data.fd;
            if(fd==lfd)  //如果是监听的socket
            {
                int cfd=accept(fd,0,0);  //接收客户端的连接
                ev.events=EPOLLIN;
                ev.data.fd=cfd;
                int ret=epoll_ctl(epfd,EPOLL_CTL_ADD,cfd,&ev);
                if(ret<0)
                {
                    perror("epoll_ctl");
                    exit(-1);
                }
            }
            else  //不是监听的说明要接收客户端的消息
            {
                char buffer[1024];
                memset(buffer,0,sizeof(buffer));
                int len=recv(fd,buffer,sizeof(buffer)-1,0);
                std::cout<<len<<std::endl;
                if(len==0)  //客户端已经断开连接
                {
                    int ret=epoll_ctl(epfd,EPOLL_CTL_DEL,fd,NULL);
                    if(ret<0)
                    {
                        perror("epoll_ctl_del");
                        exit(-1);
                    }
                    close(fd);
                }
                else if(len>0)
                {
                    std::cout<<333<<std::endl;
                    std::cout<<"client:"<<buffer<<std::endl;
                    char* recebuf="ok";
                    send(fd,recebuf,sizeof(recebuf),0);
                }
                else
                {
                    perror("recv");
                    exit(-1);
                }
            }
        }

    }
    return 0;
}
```

```cpp
//client.cpp
#include <iostream>
#include <unistd.h>
#include <ctype.h>
#include <stdlib.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <string.h>

int main(int argc, char* argv[])
{
    if (argc != 3) {
        std::cout << "命令行参数数量不对" << std::endl;
        std::cout << "./client server_ip port" << std::endl;
        return -1;
    }

    // 创建套接字
    int sockfd = socket(AF_INET, SOCK_STREAM, 0);
    if (sockfd < 0) {
        perror("socket");
        return -1;
    }

    // 设置服务器地址信息
    struct sockaddr_in server_addr;
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(atoi(argv[2]));
    if (inet_pton(AF_INET, argv[1], &server_addr.sin_addr) <= 0) {
        perror("inet_pton");
        close(sockfd);
        return -1;
    }

    // 连接到服务器
    if (connect(sockfd, (struct sockaddr*)&server_addr, sizeof(server_addr)) < 0) {
        perror("connect");
        close(sockfd);
        return -1;
    }

    // 发送数据给服务器
    const char* sendbuf = "Hello from client!";
    ssize_t send_len = send(sockfd, sendbuf, strlen(sendbuf), 0);
    if (send_len < 0) {
        perror("send");
        close(sockfd);
        return -1;
    }
    std::cout << "Sent: " << sendbuf << std::endl;

    // 接收服务器的数据
    char buffer[1024];
    memset(buffer, 0, sizeof(buffer));
    ssize_t recv_len = recv(sockfd, buffer, sizeof(buffer)-1, 0);
    if (recv_len > 0) {
        std::cout << "Received from server: " << buffer << std::endl;
    } else if (recv_len == 0) {
        std::cout << "Server closed the connection." << std::endl;
    } else {
        perror("recv");
    }

    // 关闭套接字
    close(sockfd);

    return 0;
}
```

```shell
# makefile
all: client server

server: server.cpp
 g++ -g -o server server.cpp

client: client.cpp
 g++ -g -o client client.cpp
```

## epoll的工作模式

epoll有两种工作模式：LT（Level Triggered，水平触发）和ET（Edge Triggered，边缘触发）。

### LT模式

LT又叫水平模式,是epoll的默认工作模式。在这种模式下，内核会不断地通知你文件描述符是否就绪，即使你已经读取了数据。也就是说，即使你读取了数据，文件描述符仍然被认为是就绪的，内核会继续通知你。这种模式适用于需要不断地检查文件描述符是否就绪的场景。

水平模式主要有以下特点:

- 这是epoll的默认工作模式。
- 在LT模式下，当一个文件描述符准备好进行读写操作时，epoll会通知应用程序。
- 如果应用程序没有立即处理该事件，或者在处理过程中没有完全读取或写入所有数据，那么只要文件描述符仍然处于就绪状态，epoll将继续通知应用程序。

其实对于大多数应用来说，LT模式足够使用，并且它的行为与传统的poll和select相似

## ET模式

ET又叫边缘模式，是epoll的高级模式。在这种模式下，内核只会通知你文件描述符从非就绪状态变为就绪状态一次，即使你读取了数据，文件描述符仍然被认为是非就绪的，内核不会继续通知你。这种模式适用于需要高效处理大量并发连接的场景。

边缘模式主要有以下特点:

- ET模式是一种低延迟、高性能的工作模式。
- 在ET模式下，epoll只会在状态发生变化时通知应用程序一次。例如，如果一个文件描述符从非就绪变为就绪，epoll将通知应用程序；但是，如果应用程序未能在第一次通知后立即处理完所有可用的数据，那么即使该文件描述符仍然是就绪状态，epoll也不会再次发送通知，直到该文件描述符的状态再次发生变化（即从就绪变回非就绪，再由非就绪变成就绪）。

因此，在ET模式下，应用程序必须确保每次收到通知时都尽可能多地读取或写入数据，以避免错过后续的通知。这通常意味着在循环中尽可能多地尝试读写操作，直到遇到EAGAIN或EWOULDBLOCK错误为止，这表明当前没有更多可读写的就绪数据。

**PS**:

1. LT模式会不断通知应用程序，即使应用程序已经开始读取数据,这样可以保证应用程序能够及时处理数据，但是 频繁的通知会带来性能的损耗，而ET模式只会在状态发生变化时通知应用程序一次，因此应用程序需要确保每次收到通知时都尽可能多地读取或写入数据，以避免错过后续的通知。所以ET模式要求应用程序在每次收到通知时都尽可能多地读取或写入数据，否则可能会错过后续的通知。因此，ET模式通常需要更复杂的编程逻辑，并且对应用程序的设计和实现有更高的要求。

2.我们在使用ET模式要设置EPOLLET标志如下:

```cpp
if(fd==lfd)  //如果是监听的socket
{
    int cfd=accept(fd,0,0);  //接收客户端的连接
    ev.events=EPOLLIN|EPOLLET;  //设置边沿触发
    ev.data.fd=cfd;
    int ret=epoll_ctl(epfd,EPOLL_CTL_ADD,cfd,&ev);
    if(ret<0)
    {
        perror("epoll_ctl");
        exit(-1);
    }
}
```

3.LT模式下支持阻塞和非阻塞，而ET模式下只支持非阻塞。
