# 基于socket套接字的网络编程

## 客户端类的编写

### 客户端通讯的过程

客户端连接的过程其实很好理解，主要就是以下几步:

- 创建客户端socket
- 基于客户端socket和服务端ip以及服务端开放的通讯端口与服务端建立连接
- 读取/发送数据
- 关闭socket，断开连接
而我们的对客户端类的编写，也是基于上面的几步过程来展开的。

### 客户端的私有成员

在上面我们提到了客户端连接服务端所需的一些信息，例如客户端socket，服务端ip以及服务端开放的通讯端口(云服务器开放通讯端口需要设置安全组)，所以我们可以这样定义客户端类:

```cpp
private:
    int m_socket; // 客户端的socket
    unsigned int server_port;// 服务端的端口
    string server_ip; //服务端的ip
```

### 客户端的公共函数

我们上面说过客户端连接服务端以及相关工作的大致流程，所以在定义客户端类的函数时，大概是以下类型的函数:

```cpp
public:
        ctcpclient(){m_socket=-1;}
        bool Connect(const unsigned int port,const string& ip);  //客户端连接服务端
        bool Read(string& buff,const int itimeout=0);  //接收文本数据
        bool Read(void* buff,const int bufflen,const int itimeout=0); //接收二进制数据
        bool Write(const string& buff); //发送文本数据
        bool Write(const void* buff,const int bufflen); //发送二进制数据
        void Close(); //关闭连接
        ~ctcpclient(){Close();}
```

这里的构造函数与析构函数无需多言，接下来我们主要对相关的工作函数进行探究。

### Connect(客户端连接)函数

在讲解Connect函数之前我们先来看一下它的具体执行的逻辑:

```cpp
  bool ctcpclient::Connect(const unsigned int port, const string &ip)
    {
        if (m_socket != -1)
        {
            Close();
        }
        // 忽略SIGPIPE信号，防止程序异常退出。
        // 如果send到一个disconnected socket上，内核就会发出SIGPIPE信号。这个信号
        // 的缺省处理方法是终止进程，大多数时候这都不是我们期望的。我们重新定义这
        // 个信号的处理方法，大多数情况是直接屏蔽它。
        signal(SIGPIPE, SIG_IGN);

        server_port = port;
        server_ip = ip;
        m_socket = socket(AF_INET, SOCK_STREAM, 0);
        if (m_socket < 0)
        {
            return false;
        }
        struct sockaddr_in server_addr;
        struct hostent *h;
        memset(&server_addr, 0, sizeof(server_addr));
        if ((h = gethostbyname(ip.c_str())) == NULL)
        {
            Close();
            return false;
        }
        server_addr.sin_family = AF_INET;//指定通讯协议
        server_addr.sin_port = htons(server_port); //指定通讯端口
        memset(h, 0, sizeof(h));
        memcpy(h->h_addr, &server_ip[0], server_ip.length());  //指定通讯IP地址
        if (connect(m_socket, (struct sockaddr *)&server_addr, sizeof(server_addr)) != 0)
        {
            return false;
        }
        return true;
    }
```

上述主要经过了一下几个步骤：

- 检查socket，查看当前客户端是否处于未连接状态
- 设置相关信号的处理方式，防止异常情况的出现
- 初始化客户端socket
- 定义`server_addr`与`struct hostent *h`结构体配置相关信息
- 与客户端建立连接

### Read(接收)函数

Read函数在这里所起到的作用主要是接收数据的作用,接下来我们将从接收数据的不同作为开始来探究其中的细节。

#### 接收文本数据

在对相关函数的执行逻辑与细节进行讲解之前，我们先来看一下相关的函数签名与函数实现：

```cpp
 bool Read(string& buff,const int itimeout=0);  //接收文本数据
 bool tcpread(const int sockfd,string &buffer,const int itimeout=0); // 读取文本数据
 bool readn(const int sockfd, char *buffer, const size_t n);
```

```cpp
  bool ctcpclient::Read(string &buff, const int itimeout)
    {
        if (m_socket < 0)
        {
            return false;
        }
        return (tcpread(m_socket, buff, itimeout));
    }


    bool tcpread(const int sock, string &buff, const int itimeout)
    {
        if (sock < 0)
        {
            return false;
        }
        if (itimeout > 0)
        {
            struct pollfd fds;
            fds.fd = sock;
            fds.events = POLLIN;
            int ret = poll(&fds, 1, itimeout * 1000);
            if (ret < 0)
            {
                return false;
            }
            if (ret == 0)
            {
                return false;
            }
        }
        if (itimeout < -1)
        {
            struct pollfd fds;
            fds.fd = sock;
            fds.events = POLLIN;
            int ret = poll(&fds, 1, 0);
            if (ret < 0)
            {
                return false;
            }
            if (ret == 0)
            {
                return false;
            }
        }
        int bufflen = 0;
        if (readn(sock, (char *)&bufflen, 4) == false) // 读取报文长度
        {
            return false;
        }
        buff.resize(bufflen);
        if (readn(sock, &buff[0], bufflen) == false) // 读取报文内容
        {
            return false;
        }
        return true;
    }

 bool readn(const int sockfd, char *buffer, const size_t n)
    {
        int nleft = n; // 剩余需要读取的字节数。
        int idx = 0;   // 已成功读取的字节数。
        int nread;     // 每次调用recv()函数读到的字节数。

        while (nleft > 0)
        {
            if ((nread = recv(sockfd, buffer + idx, nleft, 0)) <= 0)
                return false;

            idx = idx + nread;
            nleft = nleft - nread;
        }

        return true;
    }
```

我们可以看到上面有关于数据接收的函数一共有三个,这里主要是客户端与服务端接收/发送数据的方式基本一致，所以我们选择对相关函数进行封装避免多次书写重复函数使代码编的臃肿，下面来给大家解释主要函数的作用:

- `tcpread`
我们知道端到端的通讯其实不是每次都是立即进行的,所以接收数据的一方有时候要等待发送数据的一方将数据发送过来，而这里我们基于poll实现了一个超时机制，让我们可以手动设置接收数据方是否等待以及等待的最大时长
- `readn`
这个主要是实现对数据的读写，相对于直接调用`recv`函数，每次从socket读取指定数量的字节，即使recv函数不能一次读取所有字节。通过在循环中跟踪剩余需要读取的字节数，可以确保读取完整的数据，进而避免因为recv函数每次读取的字节数不固定而导致的数据读取不完整或错误。

#### 二进制数据

二进制数数接收与文本数据的接收又有所不同，我们来看一下它的函数签名与具体逻辑:

- 函数签名

```cpp
bool Read(void* buff,const int bufflen,const int itimeout=0); //接收二进制数据
bool tcpread(const int sockfd, void *buffer, const int ibuflen, const int itimeout = 0);//接收二进制数据
bool readn(const int sockfd, char *buffer, const size_t n);
```

- 函数逻辑

```cpp
    bool ctcpclient::Read(void *buff, const int bufflen, const int itimeout)
    {
        if (m_socket < 0)
        {
            return false;
        }
        return (tcpread(m_socket, buff, bufflen, itimeout));
    }


 bool tcpread(int sock, void *buff, const int bufflen, const int itimeout)
    {
        if (sock < 0)
        {
            return false;
        }
        if (itimeout > 0)
        {
            struct pollfd fds;
            fds.fd = sock;
            fds.events = POLLIN;
            int ret = poll(&fds, 1, itimeout * 1000);
            if (ret <= 0)
            {
                return false;
            }
        }
        if (itimeout < -1)
        {
            struct pollfd fds;
            fds.fd = sock;
            fds.events = POLLIN;
            int ret = poll(&fds, 1, 0);
            if (ret <= 0)
            {
                return false;
            }
        }
        if (readn(sock, (char *)buff, bufflen) == false) // 读取报文内容
        {
            return false;
        }
        return true;
    }

 bool readn(const int sockfd, char *buffer, const size_t n)
    {
        int nleft = n; // 剩余需要读取的字节数。
        int idx = 0;   // 已成功读取的字节数。
        int nread;     // 每次调用recv()函数读到的字节数。

        while (nleft > 0)
        {
            if ((nread = recv(sockfd, buffer + idx, nleft, 0)) <= 0)
                return false;

            idx = idx + nread;
            nleft = nleft - nread;
        }

        return true;
    }
```

我们可以发现二进制数据的接收与文本数据相比有所不同，相对于文本数据，二进制数据减少了一个接收数据长度的过程，这是因为我们在接收/二进制数据时，二进制数据通常会包含自身的大小信息。在通信双方约定好数据格式之后，发送方会在发送数据时先将数据的大小信息编码到数据中，接收方在接收数据时可以直接根据数据的大小信息来确定整个报文的大小，从而正确地解析和处理数据。

### Write函数

write函数的细节与read函数类似，这里不做赘述，直接看函数签名与逻辑了:

- 函数签名

```cpp
 bool Write(const string& buff); //发送文本数据
 bool Write(const void* buff,const int bufflen); //发送二进制数据
 bool tcpwrite(const int sockfd, const void *buffer, const int ibuflen);//发送二进制数据
 bool tcpwrite(const int sockfd, const string &buffer);  //发送文本数据
 bool readn(const int sockfd, char *buffer, const size_t n);
```

- 函数逻辑

```cpp
 bool ctcpclient::Write(const string &buff)
    {
        if (m_socket < 0)
        {
            return false;
        }
        return (tcpwrite(m_socket, buff));
    }

    bool ctcpclient::Write(const void *buff, const int bufflen)
    {
        if (m_socket < 0)
        {
            return false;
        }
        return (tcpwrite(m_socket, (char *)buff, bufflen));
    }

 bool tcpwrite(const int sock, const string &buff)
    {
        if (sock < 0)
        {
            return false;
        }
        int bufflen = buff.length();
        if (writen(sock, (char *)&bufflen, 4) == false) // 发送报文长度
        {
            return false;
        }
        if (writen(sock, &buff[0], bufflen) == false) // 发送报文内容
        {
            return false;
        }
        return true;
    }

    bool tcpwrite(int sock, const void *buff, const int bufflen)
    {
        if (sock < 0)
        {
            return false;
        }
        if (writen(sock, (char *)buff, bufflen) == false) // 发送报文内容
        {
            return false;
        }
        return true;
    }

  bool writen(const int sockfd, const char *buffer, const size_t n)
    {
        int nleft = n; // 剩余需要写入的字节数。
        int idx = 0;   // 已成功写入的字节数。
        int nwritten;  // 每次调用send()函数写入的字节数。

        while (nleft > 0)
        {
            if ((nwritten = send(sockfd, buffer + idx, nleft, 0)) <= 0)
                return false;

            nleft = nleft - nwritten;
            idx = idx + nwritten;
        }

        return true;
    }
```

### Close函数

Close函数主要用来关闭已经打开的socket

```cpp
void ctcpclient::Close()
{
   if (m_socket > 0)
   {
      close(m_socket);
      m_socket = -1;
   }
}
```

## 服务端类的编写

### 服务端类的工作流程

- 初始化监听socket，指定端口与ip,将socket设置为监听状态
- 从等待连接的队列中选取一个客户端进行连接
- 发送/接收数据
- 关闭socket,断开连接

### 服务端类的成员

```cpp
class ctcpserver
    {
    private:
        int m_listensock;//服务端的监听socket
        int m_connsock; //已连接的客户端socket
        int sockaddr_len;//客户端地址的长度
        struct sockaddr_in server_addr;//服务端地址
        struct sockaddr_in client_addr;//客户端地址
    public:
        ctcpserver(){m_listensock=-1;m_connsock=-1;}
        bool Initserver(const unsigned int port,const int backlog=5);//初始化服务端
        bool Accept(); //从已连接队列中获取一个客户端连接
        bool Read(string& buff,const int itimeout=0); //接收文本数据
        bool Read(void* buff,const int bufflen,const int itimeout=0); //接收二进制数据
        bool Write(const string& buff); //发送文本数据
        bool Write(const void* buff,const int bufflen); //发送二进制数据
        char* getclientip(); //获取客户端的ip
        void Closelisten(); //关闭监听socket
        void Closeconn(); //关闭已连接的客户端socket
        ~ctcpserver(){Closeconn();Closelisten();}
    };
```

### Initserver函数

在讲解前我们来看一下函数的具体逻辑:

```cpp
bool ctcpserver::Initserver(const unsigned int port, const int backlog) // backlog:等待连接队列的最大长度
    {
        if (m_listensock != -1)
        {
            Closelisten();
        }

        signal(SIGPIPE, SIG_IGN);

        // 打开SO_REUSEADDR选项，当服务端连接处于TIME_WAIT状态时可以再次启动服务器，
        // 否则bind()可能会不成功，报：Address already in use。
        int opt = 1;
        setsockopt(m_listensock, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

        m_listensock = socket(AF_INET, SOCK_STREAM, 0);
        if (m_listensock < 0)
        {
            return false;
        }
        memset(&server_addr, 0, sizeof(server_addr));
        server_addr.sin_family = AF_INET;

        m_listensock = socket(AF_INET, SOCK_STREAM, 0);
        if (m_listensock < 0)
        {
            return false;
        }
        struct sockaddr_in server_addr;
        memset(&server_addr, 0, sizeof(server_addr));
        server_addr.sin_family = AF_INET;
        server_addr.sin_addr.s_addr = htonl(INADDR_ANY);
        server_addr.sin_port = htons(port);
        if (bind(m_listensock, (struct sockaddr *)&server_addr, sizeof(server_addr)) != 0)
        {
            Closelisten();
            return false;
        }
        if (listen(m_listensock, backlog) != 0)
        {
            Closelisten();
            return false;
        }
        return true;
    }
```

我们梳理一下它这个的工作流程；

- 检查服务端是否已经被初始化
- 设置相关信号的处理方式，防止异常情况的出现
- 初始化服务端的监听socket
- 设置相关参数,并指定其为用于通信的ip与端口(bind)
- 将socket设置为监听状态

### Accept函数

一个时间段可能会有多个客户端连接服务端，这时候就形成了一个等待队列，服务单会在这个等待队列里面利用`accept`函数选取一个客户端进行连接：

```cpp
  bool ctcpserver::Accept()
    {
        if (m_listensock < 0)
        {
            return false;
        }

        sockaddr_len = sizeof(struct sockaddr_in);
        if ((m_connsock = accept(m_listensock, (sockaddr *)&client_addr, (socklen_t *)&sockaddr_len)) < 0)
        {
            return false;
        }
        return true;
    }
```

### Read与Write函数

服务端接收与发送数据与客户端基本功一致，这里就不做赘述，基本的这一点什么都已经提出来了，我们直接看代码:

```cpp
bool ctcpserver::Read(string &buff, const int itimeout)
    {
        if (m_listensock < 0)
        {
            return false;
        }
        return (tcpread(m_connsock, buff, itimeout));
    }

    bool ctcpserver::Read(void *buff, const int bufflen, const int itimeout)
    {
        if (m_listensock < 0)
        {
            return false;
        }
        return (tcpread(m_connsock, buff, bufflen, itimeout));
    }

    bool ctcpserver::Write(const string &buff)
    {
        if (m_listensock < 0)
        {
            return false;
        }
        return (tcpwrite(m_connsock, buff));
    }

    bool ctcpserver::Write(const void *buff, const int bufflen)
    {
        if (m_listensock < 0)
        {
            return false;
        }
        return (tcpwrite(m_connsock, (char *)buff, bufflen));
    }
```

### getclientip()函数

该函数主要的作用是获取连接的客户端的ip:

```cpp
  char *ctcpserver::getclientip()
    {
        return inet_ntoa(client_addr.sin_addr);
    }
```

### Close函数

```cpp
 void ctcpserver::Closelisten()
    {
        if (m_listensock > 0)
        {
            close(m_listensock);
            m_listensock = -1;
        }
    }

    void ctcpserver::Closeconn()
    {
        if (m_connsock > 0)
        {
            close(m_connsock);
            m_connsock = -1;
        }
    }
```

## 结语

Cpp不同于其他语言，像Go,Python等语言对上述的细节其实已经封装好了，但是cpp则是需要我们去一点点的实现,为了避免重复的书写代码，我们可以将它们封装成类来供我们去使用，以上就是这篇文章的全部内容了,大家下篇见！
