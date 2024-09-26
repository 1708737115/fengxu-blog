# 多进程下的服务端

```cpp

// server.h
#include <iostream>

class Server
{
private:
    int m_listenfd; // 监听套接字
    int m_clientfd; // 客户端套接字
    unsigned m_server_port; //服务端端口号
    std::string m_client_ip; // 客户端IP地址
public:
    Server();
    bool InitServer(const unsigned short& port);
    bool Accept();
    bool Send(const std::string& buff);
    bool Recv(std::string& buff,int max_len);
    bool closeClientfd();
    bool closeListenfd();
    const std::string& getClientip();
    ~Server();
};

void FatherExit(int sig);  // 父进程退出信号处理函数
void ChildExit(int sig);   // 子进程退出信号处理函数
```

```cpp
//server.cpp
#include "server.h"
#include <cstring>
#include <cstdlib>
#include <unistd.h>
#include <signal.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <sys/wait.h>
#include <sys/types.h>
#include <sys/socket.h>

using namespace std;


Server::Server(): m_clientfd(-1),m_listenfd(-1){}

bool Server::InitServer(const unsigned short& port)
{
    if((m_listenfd=socket(AF_INET,SOCK_STREAM,0))==-1)
    {
        return false;
    }
    struct sockaddr_in server_addr;
    memset(&server_addr,0,sizeof(server_addr));
    server_addr.sin_family=AF_INET;
    server_addr.sin_port=htons(port);
    server_addr.sin_addr.s_addr=htonl(INADDR_ANY);
    if(bind(m_listenfd,(struct sockaddr*)&server_addr,sizeof(server_addr))==-1)
    {
        close(m_listenfd);
        return false;
    }
    if(listen(m_listenfd,5)==-1)
    {
        close(m_listenfd);
        return false;
    }
    return true;
}

bool Server::Accept()
{
    struct sockaddr_in client_addr;
    socklen_t client_addr_len=sizeof(client_addr);
    if((m_clientfd=accept(m_listenfd,(struct sockaddr*)&client_addr,&client_addr_len))==-1)
    {
        return false;
    }
    m_client_ip=inet_ntoa(client_addr.sin_addr);
    return true;
}
bool Server::Send(const string& buff)
{
    if(m_clientfd==-1)
    {
        return false;
    }
    if(send(m_clientfd,buff.c_str(),buff.size(),0)==-1)
    {
        return false;
    }
    return true;
}

bool Server::Recv(string& buff,int maxlen)
{
    buff.clear();
    buff.resize(maxlen);
    int readn=recv(m_clientfd,&buff[0],buff.size(),0);
    if(readn<=0)
    {
        buff.clear();
        return false;
    }
    buff.resize(readn);
    return true;
}

bool Server::closeClientfd()
{
    if(m_clientfd==-1) return false;
    close(m_clientfd);
    return true;
}

bool Server::closeListenfd()
{
    if(m_listenfd==-1) return false;
    close(m_listenfd);
    return true;
}

const string& Server::getClientip()
{
    return m_client_ip;
}

Server::~Server()
{
    closeClientfd();
    closeListenfd();
}

Server server;


int main(int argc,char* argv[])
{
    if(argc!=2)
    {
       cout<<"Example: ./demo port"<<endl;
       return -1; 
    }

    //先忽略掉所有的信号，后面再对它们进行专门的处理
    for(int i=1;i<=64;i++)
    {
        signal(i,SIG_IGN);
    }
    signal(2,FatherExit);    //SIGINT   Ctrl+c
    signal(15,FatherExit);   //SIGTERM   kill
    if(!server.InitServer(atoi(argv[1])))
    {
        perror("InitServer");
        return -1;
    }
    while(true)
    {
        if(!server.Accept())
        {
            perror("Accept");
            return -1;
        }
        cout<<"Accept from "<<server.getClientip()<<endl;
        int pid=fork();
        if(pid==-1)
        {
            perror("fork");  //系统资源不足
            return -1;
        }
        else if(pid>0)
        {
            server.closeClientfd();   //父进程只需要处理客户端的连接请求，无需与客户端进行通信，因此关闭客户端套接字
            continue;  //让父进程返回继续接收客户端的连接申请
        }
        server.closeListenfd();  //子进程不需要监听客户端的连接请求，因此关闭监听套接字
        signal(15,ChildExit);
        signal(2,SIG_IGN);      //重新设置子进程退出信号
        cout<<"客户端已连接"<<server.getClientip()<<endl;
        string buff;
        while (true)
        {
            // 接收对端的报文，如果对端没有发送报文，recv()函数将阻塞等待。
            if (server.Recv(buff,1024)==false)
            {
                perror("recv()"); break;
            }
            cout << "接收：" << buff<< endl;
 
            buff="ok";  
            if (server.Send(buff)==false)  // 向对端发送报文。
            {
                 perror("send"); break;
            }
            cout << "发送：" << buff << endl;
        }
    }
}

void FatherExit(int sig)
{
    //暂时忽略其他信号,避免对操作造成干扰
   signal(SIGINT,SIG_IGN);
   signal(SIGTERM,SIG_IGN);

   cout<<"父进程正在退出"<<endl;

   server.closeListenfd();  //关闭监听套接字

   kill(0,SIGTERM);   //向所有子进程发送SIGTERM信号

   exit(0);
}

void ChildExit(int sig)
{
     //暂时忽略其他信号,避免对操作造成干扰
    signal(SIGINT,SIG_IGN);
    signal(SIGTERM,SIG_IGN);
    cout<<"子进程"<<getpid()<<"正在退出,sig="<<sig<<endl;
    server.closeClientfd();  //关闭客户端套接字
    server.closeClientfd();  //关闭客户端套接字
    exit(0);
}
```

```makefile
all: demo1

demo1:
 g++ -o demo1 server.cpp

clean:
 rm -f demo1
```
