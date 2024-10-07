# 基于多线程实现的并发服务器

```cpp
//server
#include <iostream>
#include <cstring>
#include <thread>
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

using namespace std;

struct SockInfo {
    int fd;      // 通信句柄
    pthread_t pid; // 线程id
    sockaddr_in addr; // 客户端地址
};

SockInfo sock_infos[128];
sockaddr_in server_addr;

int listenfd;

bool InitServer(unsigned int port, unsigned int backlog) {  // backlog为单个线程的最大连接数
    listenfd = socket(AF_INET, SOCK_STREAM, 0);  // 初始化服务端的监听
    if (listenfd < 0) {
        perror("listensocket init failed");
        return false;
    }
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(port);
    server_addr.sin_addr.s_addr = htonl(INADDR_ANY); // 监听所有网卡
    int ret = bind(listenfd, (struct sockaddr*)&server_addr, sizeof(server_addr)); // 绑定
    if (ret < 0) {
        perror("bind failed");
        return false;
    }
    ret = listen(listenfd, backlog);
    if (ret < 0) {
        perror("listen failed");
        return false;
    }
    return true;
}

void* worker(void* arg) {
    SockInfo* pinfo = (SockInfo*)arg;
    cout<<"child thread, connfd: " << pinfo->fd << endl;
    while (true) {
        char buff[1024];
        memset(buff, 0, sizeof(buff));
        int ret = recv(pinfo->fd, buff, sizeof(buff), 0);
        if (ret == 0) {
            cout << "client disconnected" << endl;
            pinfo->fd = -1;
            break;
        } else if (ret == -1) {
            perror("recv failed");
            break;
        }
        cout << "receive msg: " << buff << endl;
        ret=send(pinfo->fd,buff,strlen(buff),0); 
        if(ret==-1)
        {
            perror("send failed");
            return 0;
        }
        cout<<"send msg: "<<buff<<endl;
    }
    close(pinfo->fd);
    pthread_exit(NULL);
}

int main(int argc, char* argv[]) {
    if (argc < 2) {
        cerr << "Usage: " << argv[0] << " <port>" << endl;
        return EXIT_FAILURE;
    }
    unsigned int port = atoi(argv[1]);
    if (!InitServer(port, 5)) {
        exit(EXIT_FAILURE);
    }

    int max = sizeof(sock_infos) / sizeof(sock_infos[0]);
    for (int i = 0; i < max; i++) {
        memset(&sock_infos[i], 0, sizeof(sock_infos[i]));
        sock_infos[i].fd = -1;
        sock_infos[i].pid = -1;
    }
    
    SockInfo* pinfo;
    while (true) {
        // 创建子线程
        int found = 0;
        for (int i = 0; i < max; i++) {
            if (sock_infos[i].fd == -1) {
                pinfo = &sock_infos[i];
                found = 1;
                break;
            }
        }

        if (!found) {  // 如果没有找到空闲位置
            // 拒绝新的连接请求
            int connfd = accept(listenfd, (struct sockaddr*)&pinfo->addr, (socklen_t*)&(pinfo->addr));
            if (connfd != -1) {
                char error_msg[] = "已达到连接数的最大值\n";
                send(connfd, error_msg, sizeof(error_msg) - 1, 0);  // 发送错误信息
                close(connfd);  // 关闭连接
                continue;
            }
        }

        int connfd = accept(listenfd, (struct sockaddr*)&pinfo->addr, (socklen_t*)&(pinfo->addr));
        if (connfd == -1) {
            perror("accept failed");
            continue;
        }
        pinfo->fd = connfd;
        cout << "parent thread, connfd: " << connfd << endl;

        // 创建线程处理客户端连接
        pthread_create(&pinfo->pid, NULL, worker, pinfo);
        pthread_detach(pinfo->pid);  // 设置线程为分离状态
    }

    close(listenfd);
    return 0;
}
```

````cpp
// client
#include <iostream>
#include <cstring>
#include <cstdio>
#include <unistd.h>
#include <netdb.h>
#include <sys/socket.h>
#include <sys/types.h>


using namespace std;

int main(int argc,char *argv[])
{
    if(argc!=3)
    {
        cout<<"Examples: ./demo1 服务端 端口号"<<endl;
        return -1;
    }

    //创建socket套接字
    int socketid=socket(AF_INET,SOCK_STREAM,0);
    if(socketid==-1)
    {
        perror("socket error");
        return -1;
    }

    //绑定服务端
    struct hostent* h;  //存放服务端IP的数据结构
    if((h=gethostbyname(argv[1]))==0)
    {
        perror("gethostbyname error");
        close(socketid);
        return -1;
    }
    struct sockaddr_in serveraddr;
    memset(&serveraddr,0,sizeof(serveraddr));
    serveraddr.sin_family=AF_INET;
    serveraddr.sin_port=htons(atoi(argv[2]));
    if(connect(socketid,(struct sockaddr*)&serveraddr,sizeof(serveraddr))==-1)
    {
        perror("connect error");
        close(socketid);
        return -1;
    }

    //通讯
    int iret;
    char buffer[1024];
    memset(buffer,0,sizeof(buffer));
    for(int i=1;i<10;i++)
    {
        //发送报文
        sprintf(buffer,"这是第%d份报文",i);
        if((iret=send(socketid,buffer,strlen(buffer),0))==-1)
        {
            perror("send error");
            close(socketid);
            break;
        }
        sleep(10);
        cout<<"发送成功"<<endl;

        //接收回应报文
        memset(buffer,0,sizeof(buffer));
        if((iret=recv(socketid,buffer,sizeof(buffer),0))==-1)
        {
            perror("recv error");
            close(socketid);
            break;
        }
        cout<<"接收成功"<<endl;
    }

    //断开连接，释放资源
    close(socketid);
    cout<<"连接已断开"<<endl;
    return 0;
}
```
