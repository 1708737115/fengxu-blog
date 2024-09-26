# 进程的心跳

## 什么是进程的心跳

在我们日常后台服务程序运行中,一般是调度模块，进程心跳以及进程监控共同工作，进而实现实现服务的稳定运行,在前面我们介绍过如何去实现一个简单的调度模块,而今天我们所要介绍的就是如何实现进程的心跳，首先什么是进程的心跳呢？`进程的心跳机制是一种监控进程健康状况的方法，通常用于服务端应用或者需要长期运行的后台守护进程（daemon`,我们在后台服务程序在运行中由于有很多功能需要运行，所以一般是以多进程的方式来同时运行多个后台服务程序,那么问题来了，当有多个后台服务程序在运行时我们如何来确定它们的运行状态呢？这就是进程心跳的功能了，进程心跳，顾名思义，就是我们用来判断进程是否正常运行的一种手段，我们首先定义一个心跳信息结构体,如下:

```cpp
struct stprocinfo  //存储进程心跳信息的结构体
{
    int pid;  //进程编号
    char name[50]={0}; //进程名称
    int timeout;  //超时时间
    time_t atime; //最后一次心跳时间
    stprocinfo() =default;  //默认构造函数 
    stprocinfo(int pid,char* name,int timeout,time_t atime):pid(pid),timeout(timeout),atime(atime)
    {
        strcpy(this->name,name);
    }
};
```

结构体中存储了进程的相关信息，同时我们会通过`atime`和`timeout`来判断结构体是否是正常运行，下面我们来看一下具体的实现过程。

## 进程心跳的初步实现

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/1df63db6d71e1b092b2dc5f72d43c4d2.png)
上面是一个进程心跳的初步实现，主要是以下几步:

 1. 创建共享内存

 ```cpp
  //创建共享内存
    shmid=shmget((key_t)0X5550,sizeof(struct stprocinfo)*1000,IPC_CREAT|0666);
    if (shmid==-1)
    {
        perror("创建共享内存失败");
        return -1;
    }
    cout<<"shmid="<<shmid<<endl;
 ```

 2. 将共享内存连接到当前进程的地址空间

 ```cpp
     //将共享内存连接到当前进程的地址空间
    m_shm=(stprocinfo*)shmat(shmid,nullptr,0);
    if(m_shm==(void*)-1)
    {
        perror("共享内存连接失败");
        return -1;
    }
    cout<<"shmat ok"<<endl;
 ```

 3. 遍历共享内存，寻找空闲位置

 ```cpp
 
    //遍历当前共享内存的占用情况,主要用来调试
    for(int i=0;i<1000;i++)
    {
        if(m_shm[i].pid!=0)
        {
            cout<<"pid="<<m_shm[i].pid<<" "
            <<"name="<<m_shm[i].name<<" "
            <<"timeout="<<m_shm[i].timeout<<" "
            <<"atime="<<m_shm[i].atime<<endl;
        }
    }

    stprocinfo info(getpid(),"server1",30,time(0));

    //在共享内存中查找是否有空闲的地址
    for(int i=0;i<1000;i++)
    {
        if(m_shm[i].pid==0)
        {
            m_pos=i;
            cout<<"找到空闲地址,m_pos="<<m_pos<<endl;
            break;
        }
    }

    if(m_pos==-1)
    {
        cout<<"共享内存已满"<<endl;
        Exit(-1);
    }

    memcp
 ```

 4. 程序运行并且更新进程心跳

 ```cpp
     while(1)
    {
        //写入程序的运行逻辑
        sleep(10);  //模拟程序运行
        m_shm[m_pos].atime=time(0);  //更新当前进程的心跳时间
    }
 ```

完整代码如下:

```cpp
#include "../../public/_public.h"

using namespace idc;   //自己封装的命名空间

struct stprocinfo  //存储进程心跳信息的结构体
{
    int pid;  //进程编号
    char name[50]={0}; //进程名称
    int timeout;  //超时时间
    time_t atime; //最后一次心跳时间
    stprocinfo() =default;  //默认构造函数 
    stprocinfo(int pid,char* name,int timeout,time_t atime):pid(pid),timeout(timeout),atime(atime)
    {
        strcpy(this->name,name);
    }
};

int shmid=-1; //共享内存的ID
stprocinfo* m_shm=nullptr;//指向共享内存的地址
int m_pos=-1; //记录当前进程在共享内存的位置
void Exit(int sig); //信号处理函数

int main()
{
    //信号处理
    signal(SIGINT,Exit);
    signal(SIGTERM,Exit);

    //创建共享内存
    shmid=shmget((key_t)0X5550,sizeof(struct stprocinfo)*1000,IPC_CREAT|0666);
    if (shmid==-1)
    {
        perror("创建共享内存失败");
        return -1;
    }
    cout<<"shmid="<<shmid<<endl;

    //将共享内存连接到当前进程的地址空间
    m_shm=(stprocinfo*)shmat(shmid,nullptr,0);
    if(m_shm==(void*)-1)
    {
        perror("共享内存连接失败");
        return -1;
    }
    cout<<"shmat ok"<<endl;

    //遍历当前共享内存的占用情况,主要用来调试
    for(int i=0;i<1000;i++)
    {
        if(m_shm[i].pid!=0)
        {
            cout<<"pid="<<m_shm[i].pid<<" "
            <<"name="<<m_shm[i].name<<" "
            <<"timeout="<<m_shm[i].timeout<<" "
            <<"atime="<<m_shm[i].atime<<endl;
        }
    }

    stprocinfo info(getpid(),"server1",30,time(0));

    //在共享内存中查找是否有空闲的地址
    for(int i=0;i<1000;i++)
    {
        if(m_shm[i].pid==0)
        {
            m_pos=i;
            cout<<"找到空闲地址,m_pos="<<m_pos<<endl;
            break;
        }
    }

    if(m_pos==-1)
    {
        cout<<"共享内存已满"<<endl;
        Exit(-1);
    }

    memcpy(&m_shm[m_pos],&info,sizeof(struct stprocinfo));  //将当前进程的心跳信息存入共享内存

    while(1)
    {
        //写入程序的运行逻辑
        sleep(10);  //模拟程序运行
        m_shm[m_pos].atime=time(0);  //更新当前进程的心跳时间
    }
    return 0;
}

void Exit(int sig)
{
    cout<<"收到信号:"<<sig<<endl;
    //清除共享内存中的心跳信息结构体
    if(m_pos!=-1)
    {
        memset(m_shm+m_pos,0,sizeof(struct stprocinfo));
        m_pos=-1;
    }
    //将共享内存分离出来
    if(m_shm!=nullptr)
    {
        shmdt(m_shm);
    }
    exit(0);
}
```

编译代码如下：

```cpp
# 开发框架头文件路径
PUBINCL = -I/root/mylib/project/public

# 开发框架cpp文件名，这里直接包含进来，没有采用链接库，是为了方便调试。
PUBLICPP = /root/mylib/project/public/_public.cpp
# 编译参数
CFLAGS= -g

all: demo1

demo1:demo1.cpp
 g++ $(CFLAGS) -o demo1 demo1.cpp $(PUBLICPP)

clean:
 rm -f demo1

```

**注明**
以上仅供参考，大家根据自己的需求来进行修改。

测试并且运行，这里我们开启三个命令行来测试：
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/4329b9151f228954baa27e097f9b2cb3.png)
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/935b8e9addb71a46e3368b5e83415754.png)
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/7aaff84c409fc81c8fe05f498e5ec1f3.png)
我们看到基本实现了我们想的功能，将带有进程心跳的信息储存到了结构体中，后面我们会介绍我们如何基于守护进程来对进程的心跳来实现对进程状态的监控，不过这是后话了。

# 进程心跳代码的优化

上面我们已经实现了一个简单的进程心跳代码，但是现在有几个实际情况需要我们来考虑,分别是：

- 这里我们关于退出信号的处理是无法处理`kill -9`这种异常退出的，这个时候我们的`Exit`函数是无法正常发挥功能的,同时也会导致该进程的心跳信息残留在共享内存中，所以我们要对代码进行优化:

```cpp
    //在共享内存中查找是否有空闲的地址

    for(int i=0;i<1000;i++)  //如果有与该进程pid相同，说明有进程未清理干净
    {
        if(m_shm[i].pid==info.pid)
        {
            cout<<"找到旧进程"<<endl;
            m_pos=i;
            break;
        }
    }
    
    if(m_pos==-1)
    {
        for(int i=0;i<1000;i++)
        {
            if(m_shm[i].pid==0)
            {
                m_pos=i;
                cout<<"找到空闲地址,m_pos="<<m_pos<<endl;
                break;
            }
        }
    }

    if(m_pos==-1)
    {
        cout<<"共享内存已满"<<endl;
        Exit(-1);
    }

```

- 这里我们要求的使用场景是多个进程在同时运行,那么问题来了，如果同时多个程序运行同时向共享内存的一块地址写怎么办?这里我们可以基于进程同步来解决这个问题,而向实现进程同步，就需要信号量了，这里我使用的是已经封装好的信号量，代码如下:

```cpp
    class csemp
    {
    private:
        union semun // 用于操控共享内存的联合体
        {
            int value;
            struct semid_ds *buf;
            unsigned short *arry;
        };
        int m_semid; // 信号量id

        /*如果将m_semflg设为SEM_UNOD,操作系统将跟踪进程对信号量的修改,在全部修改过信号量的进程终止后将信号量设置为初始值
        m_semflag=1时用于互斥锁，m_semflag=0时用于生产消费者模型*/
        short m_semflg;                           // 信号量值
        csemp(const csemp &) = delete;            // 禁用拷贝构造函数
        csemp &operator=(const csemp &) = delete; // 禁用赋值运算符

    public:
        csemp() : m_semid(-1) {}
        /*如果信号量已存在，就获取信号量
        如果信号量不存在，就创建信号量并将其初始化为value
        互斥锁时，value=1,semflag=SEM_UNOD
        生产消费者模型时，value=0，semflag=0*/
        bool init(key_t key, unsigned short value = 1, short semflg = SEM_UNDO);
        bool wait(short value = -1); // P操作
        bool post(short value = 1);  // V操作
        int getvalue();
        bool destroy();
        ~csemp();
    };


        bool csemp::init(key_t key, unsigned short value, short semflg)
    {
        if (m_semid != -1) // 信号量已经初始化了
        {
            return false;
        }
        m_semflg = semflg;
        if ((m_semid = semget(key, 1, 0666)) == -1) // 尝试获取信号量
        {
            if (errno == ENOENT) // 未找到信号量
            {
                if ((m_semid = semget(key, 1, IPC_CREAT | 0666 | IPC_EXCL)) == -1) // 创建信号量
                {
                    if (errno == EEXIST) // 信号量已存在
                    {
                        if ((m_semid = semget(key, 1, 0666)) == -1)
                        {
                            perror("init semget 1");
                            return false;
                        }
                        return true;
                    }
                    else
                    {
                        perror("init semget 2");
                        return false;
                    }
                }
                union semun b;
                b.value = value;
                if (semctl(m_semid, 0, SETVAL, b) == -1) // 设置信号量初值
                {
                    perror("init semctl");
                    return false;
                }
            }
            else
            {
                perror("init semget 3");
                return false;
            }
        }
        return true;
    }

    bool csemp::wait(short value)
    {
        if (m_semid == -1)
            return false;
        struct sembuf s;
        s.sem_num = 0;
        s.sem_op = value;
        s.sem_flg = m_semflg;
        if (semop(m_semid, &s, 1) == -1)
        {
            perror("wait semop");
            return false;
        }
        return true;
    }

    bool csemp::post(short value)
    {
        if (m_semid == -1)
            return false;
        struct sembuf s;
        s.sem_num = 0;
        s.sem_op = value;
        s.sem_flg = m_semflg;
        if (semop(m_semid, &s, 1) == -1)
        {
            perror("post semop");
            return false;
        }
        return true;
    }

    int csemp::getvalue()
    {
        return semctl(m_semid, 0, GETVAL);
    }

    bool csemp::destroy()
    {
        if (m_semid == -1)
            return false;
        if (semctl(m_semid, 0, IPC_RMID) == -1)
        {
            perror("destroy semctl");
            return false;
        }
        return true;
    }

    csemp::~csemp()
    {
    }
```

实现代码如下：

```cpp
 csemp shmlock;
    if(shmlock.init(0x5550)==-1)
    {
        cout<<"shmlock init error"<<endl;
        Exit(-1);
    }

    shmlock.wait(); //加锁

    //在共享内存中查找是否有空闲的地址

    for(int i=0;i<1000;i++)
    {
        if(m_shm[i].pid==info.pid)
        {
            cout<<"找到旧进程"<<endl;
            m_pos=i;
            break;
        }
    }

    if(m_pos==-1)
    {
        for(int i=0;i<1000;i++)
        {
            if(m_shm[i].pid==0)
            {
                m_pos=i;
                cout<<"找到空闲地址,m_pos="<<m_pos<<endl;
                break;
            }
        }
    }

    if(m_pos==-1)
    {
        cout<<"共享内存已满"<<endl;
        shmlock.post(); //解锁
        Exit(-1);
    }

    memcpy(&m_shm[m_pos],&info,sizeof(struct stprocinfo));  //将当前进程的心跳信息存入共享内存
    shmlock.post(); //解锁
```

最后我们就实现一个基本的可以实现多进程监控的进程心跳程序了，在最后我们可以将它封装成一个简单的类,供我们以后使用：

```cpp
    // 进程心跳有关的类

    struct st_procinfo // 存储进程心跳信息的结构体
    {
        int pid;                // 进程编号
        char name[50] = {0};    // 进程名称
        int timeout;            // 超时时间
        time_t atime;           // 最后一次心跳时间
        st_procinfo() = default; // 默认构造函数
        st_procinfo(int pid, char *name, int timeout, time_t atime) : pid(pid), timeout(timeout), atime(atime)
        {
            strcpy(this->name, name);
        }
    };

    #define MAXNUM 1000;   // 最大进程数
    #define SHMKEYP 0x5095 // 共享内存的key。
    #define SEMKEYP 0x5095 // 信号量的key。

    class cpactive // 实现进程心跳的类
    {
    private:
        int shmid;
        int m_pos;
        st_procinfo *m_shm;

    public:
        cpactive();
        bool init(int timeout,string pname="",clogfile *logfile=nullptr);  //这里传指针视为了选择是否使用日志打印
        bool update();  //更新心跳时间
        ~cpactive();
    };


  cpactive::cpactive()
 {
     m_shmid=0;
     m_pos=-1;
     m_shm=0;
 }

 // 把当前进程的信息加入共享内存进程组中。
 bool cpactive::addpinfo(const int timeout,const string &pname,clogfile *logfile)
 {
    if (m_pos!=-1) return true;

    // 创建/获取共享内存，键值为SHMKEYP，大小为MAXNUMP个st_procinfo结构体的大小。
    if ( (m_shmid = shmget((key_t)SHMKEYP, MAXNUMP*sizeof(struct st_procinfo), 0666|IPC_CREAT)) == -1)
    { 
        if (logfile!=nullptr) logfile->write("创建/获取共享内存(%x)失败。\n",SHMKEYP); 
        else printf("创建/获取共享内存(%x)失败。\n",SHMKEYP);

        return false; 
    }

    // 将共享内存连接到当前进程的地址空间。
    m_shm=(struct st_procinfo *)shmat(m_shmid, 0, 0);
  
    /*
    struct st_procinfo stprocinfo;    // 当前进程心跳信息的结构体。
    memset(&stprocinfo,0,sizeof(stprocinfo));
    stprocinfo.pid=getpid();            // 当前进程号。
    stprocinfo.timeout=timeout;         // 超时时间。
    stprocinfo.atime=time(0);           // 当前时间。
    strncpy(stprocinfo.pname,pname.c_str(),50); // 进程名。
    */
    st_procinfo stprocinfo(getpid(),pname.c_str(),timeout,time(0));    // 当前进程心跳信息的结构体。

    // 进程id是循环使用的，如果曾经有一个进程异常退出，没有清理自己的心跳信息，
    // 它的进程信息将残留在共享内存中，不巧的是，如果当前进程重用了它的id，
    // 守护进程检查到残留进程的信息时，会向进程id发送退出信号，将误杀当前进程。
    // 所以，如果共享内存中已存在当前进程编号，一定是其它进程残留的信息，当前进程应该重用这个位置。
    for (int ii=0;ii<MAXNUMP;ii++)
    {
        if ( (m_shm+ii)->pid==stprocinfo.pid ) { m_pos=ii; break; }
    }

    csemp semp;                       // 用于给共享内存加锁的信号量id。

    if (semp.init(SEMKEYP) == false)  // 初始化信号量。
    {
        if (logfile!=nullptr) logfile->write("创建/获取信号量(%x)失败。\n",SEMKEYP); 
        else printf("创建/获取信号量(%x)失败。\n",SEMKEYP);

        return false;
    }

    semp.wait();  // 给共享内存上锁。

    // 如果m_pos==-1，表示共享内存的进程组中不存在当前进程编号，那就找一个空位置。
    if (m_pos==-1)
    {
        for (int ii=0;ii<MAXNUMP;ii++)
            if ( (m_shm+ii)->pid==0 ) { m_pos=ii; break; }
    }

    // 如果m_pos==-1，表示没找到空位置，说明共享内存的空间已用完。
    if (m_pos==-1) 
    { 
        if (logfile!=0) logfile->write("共享内存空间已用完。\n");
        else printf("共享内存空间已用完。\n");

        semp.post();  // 解锁。

        return false; 
    }

    // 把当前进程的心跳信息存入共享内存的进程组中。
    memcpy(m_shm+m_pos,&stprocinfo,sizeof(struct st_procinfo)); 

    semp.post();   // 解锁。

    return true;
 }

 // 更新共享内存进程组中当前进程的心跳时间。
 bool cpactive::uptatime()
 {
    if (m_pos==-1) return false;

    (m_shm+m_pos)->atime=time(0);

    return true;
 }

 cpactive::~cpactive()
 {
    // 把当前进程从共享内存的进程组中移去。
    if (m_pos!=-1) memset(m_shm+m_pos,0,sizeof(struct st_procinfo));

    // 把共享内存从当前进程中分离。
    if (m_shm!=0) shmdt(m_shm);
 }
```

至此一个简单的进程心跳类就封装号了,后面我会介绍如何基于守护进程，进程心跳和调度模块来实现对进程的监控，大家下篇见!
