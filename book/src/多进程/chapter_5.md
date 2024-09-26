# 信号量

## 循环队列

### 前言

在我们使用共享内存的时候，我们一般要注意以下几点：

1. 共享内存无法自动拓展，我们只能使用c++内置的数据类型
2. 共享内存不能采用STL日期，也不能使用移动语义

而在我们日常对共享内存进行操作的时候，主要还是以队列为主

### 代码示例

```cpp
//public.h
#ifndef _PUBLIC_HH
#define _PUBLIC_HH

#include<iostream>
#include<cstring>
#include<cstdio>
#include<sys/shm.h>
#include<sys/ipc.h>
#include<unistd.h>


template<typename TT,int MaxLength>
class squeue
{
    private:
    bool M_inited;//判断队列是否已经被初始化
    TT M_data[MaxLength];//存储数据的数组
    int M_head;//队头指针
    int M_tail;//队尾指针
    int length;//队列中实际元素的数量
    squeue(const squeue &) =delete;//禁止拷贝构造函数
    squeue& operator=(const squeue &) =delete;//禁止直接赋值
    public:
    squeue() {init();}  //构造函数
    void init()
    {
        if(M_inited!=true)
        {
            M_inited=true;
            M_head=0;
            M_tail=MaxLength-1;
            length=0;
            sizeof(M_data,0,sizeof(M_data));
        }
    }

    bool push(TT& ee)
    {
        if(length==MaxLength)
        {
            std::cout<<"the queue is full"<<std::endl;
            return false;
        }
        M_tail=(M_tail+1)%MaxLength;
        M_data[M_tail]=ee;
        length++;
        return true;
    }

    int getSize()
    {
        return length;
    } 

    bool isFull()
    {
        return length==MaxLength;
    }

    TT& front()
    {
        return M_data[M_head];
    }

    bool pop()
    {
        if(length==0)
        {
            std::cout<<"the queue is empty"<<std::endl;
            return false;
        }
        M_head=(M_head+1)%MaxLength;
        length--;
        return true;
    } 

    ~squeue() {}  //析构函数
};

#endif // __PUBLIC_HH

//共享内存下的循环队列.cpp
#include "public.h"

using namespace std;

struct gril
{
    int no;
    char name[20];
};

int main()
{
    squeue<gril,5> q;
    //初始化共享内存
    int shmid=shmget(0x5550,sizeof(q),0666|IPC_CREAT);
    if(shmid==-1)
    {
        cout<<"shmget(0x5550) failed"<<endl;
        return -1;
    }
    cout<<"shmid="<<shmid<<endl;
    //将共享内存连接到当前进程的地址空间
    squeue<gril,5>* q_ptr=(squeue<gril,5>*)shmat(shmid,0,0);
    if(q_ptr==(void*)-1)
    {
        cout<<"shmat failed"<<endl;
        return -1;
    }

    //对共享内存进行读写
    q.init();
    gril a[5];
    for(int i=0;i<5;i++)
    {
        a[i].no=i;
        strcpy(a[i].name,"girl");
        q.push(a[i]);
    }
    for(int i=0;i<3;i++)
    {
        q.pop();
    }
    cout<<"队列的长度为"<<q.getSize()<<endl;

    //断开连接
    shmdt(q_ptr);

    return 0;

}
```

## 信号量

### 信号量的基本概念

- 信号量本质上是一个是一个非负数的计数器，用于给共享资源创建一个标志，来表示该共享资源的被占用情况

### 信号量的两种操作

- P操作：将信号量的值减一，如果信号量的值为0，则会阻塞等待，直到信号量的值再次大于0
- V操作：将信号量得值加一，任何时候都不会阻塞

### 信号量的应用场景

- 如果约定的信号量的取值只是0和1(0-资源不可用;1-资源可用)，我们可以实现互斥锁
- 如果约定信号量的取值表示可用资源的数量，可以实现生产/消费者模型

### 常用函数

- **semop函数**

  `semop` 函数是 Linux 系统调用，用于对指定的信号量进行操作。它通常由 `sem_t` 类型的结构体作为参数，用于对信号量的操作，如信号量的初始化、等待和通知等。

  `semop` 函数的定义在 `unistd.h` 头文件中，通常在 C 语言中使用。在 Linux 系统中，信号量用于同步和通信，可以用来实现线程之间的同步。信号量可以有多个，每个信号量代表一个资源，线程可以通过等待和通知来访问这些资源。`semop` 函数就是线程用于对信号量进行操作的重要接口。

  ```CPP
  int semop(const struct sem_t *set, unsigned int nsems, int cmd);
  ```

  参数 `set` 是一个 `sem_t` 类型的结构体数组，用于存放信号量操作的参数。`nsems` 是一个无符号整数，表示 `set` 数组中信号量的数量。`cmd` 是一个整数，表示要执行的信号量操作。这个值可以是以下几种：

  - `SEM_WAIT`：等待信号量可用。
  - `SEM_POST`：释放信号量到一个就绪的线程。
  - `SEM_INIT`：初始化信号量。
  - `SEM_DESTROY`：销毁信号量。

  `semop` 函数返回一个整数，表示信号量操作的成功与否。如果操作成功，返回 `0`；否则返回 `-1`。

### 示例代码

```cpp
//public.h

#include <iostream>
#include <cstring>
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/sem.h>

using namespace std;

//信号量
class csemp
{
private:
    union semun    //用于操控共享内存的联合体
    {
       int value;
       struct semid_ds *buf;
       unsigned short *arry;
    };
    int m_semid; //信号量id


    /*如果将m_semflg设为SEM_UNOD,操作系统将跟踪进程对信号量的修改,在全部修改过信号量的进程终止后将信号量设置为初始值
    m_semflag=1时用于互斥锁，m_semflag=0时用于生产消费者模型*/
    short m_semflg; //信号量值
    csemp(const csemp&) =delete;  //禁用拷贝构造函数
    csemp& operator=(const csemp&) =delete; //禁用赋值运算符

public:
    csemp():m_semid(-1){}
    /*如果信号量已存在，就获取信号量
    如果信号量不存在，就创建信号量并将其初始化为value
    互斥锁时，value=1,semflag=SEM_UNOD
    生产消费者模型时，value=0，semflag=0*/
    bool init(key_t key,unsigned short value=1,short semflg=SEM_UNDO);
    bool wait(short value=-1);  //P操作
    bool post(short value=1);   //V操作
    int getvalue();
    bool destroy();
    ~csemp();
};

//循环队列
template<typename T,int MaxLength>
class squeue
{
private:
    int m_inited=-1;
    int m_head;
    int m_tail;
    int m_length;
    T m_data[MaxLength];
    squeue(const squeue&)=delete; 
    squeue& operator=(const squeue&)=delete;
public:
    squeue(){init();}
    bool init()
    {
        if(m_inited!=-1) return false;
        m_inited=1;
        m_head=0;
        m_tail=0;
        m_length=0;
        memset(m_data,0,sizeof(m_data));
        return true;
    }
    bool push(const T& data)
    {
        if(full()) return false;
        m_data[m_tail]=data;
        m_tail=(m_tail+1)%MaxLength;
        m_length++;
        return true;
    }

    bool pop()
    {
        if(empty()) return false;
        m_data[m_head]=0;
        m_head=(m_head+1)%MaxLength;
        m_length--;
        return true;
    }

    bool empty()
    {
        return m_length==0;
    }

    bool full()
    {
        return m_length==MaxLength;
    }

    int size()
    {
        return m_length;
    }
    T& front()
    {
        return m_data[m_head];
    }

    void Print()
    {
        for(int i=m_head;i<m_tail;i++)
        {
            cout<<m_data[i]<<" ";
        }
    }

    ~squeue(){}
};

//public.cpp
#include "public.h"

bool csemp::init(key_t key,unsigned short value,short semflg)
{
    if(m_semid!=-1)  //信号量已经初始化了
    {
        return false;
    }
    m_semflg=semflg;
    if((m_semid=semget(key,1,0666))==-1)   //尝试获取信号量
    {
        if(errno==ENOENT) //未找到信号量
        {
            if((m_semid=semget(key,1,IPC_CREAT|0666|IPC_EXCL))==-1)  //创建信号量
            {
                if(errno==EEXIST)//信号量已存在
                {
                    if((m_semid=semget(key,1,0666))==-1)
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
            b.value=value;
            if(semctl(m_semid,0,SETVAL,b)==-1)  //设置信号量初值
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
    if(m_semid==-1)  return false;
    struct sembuf s;
    s.sem_num=0;
    s.sem_op=value;
    s.sem_flg=m_semflg;
    if(semop(m_semid,&s,1)==-1)
    {
        perror("wait semop");
        return false;
    }
    return true;
}

bool csemp::post(short value)
{
    if(m_semid==-1)  return false;
    struct sembuf s;
    s.sem_num=0;
    s.sem_op=value;
    s.sem_flg=m_semflg;
    if(semop(m_semid,&s,1)==-1)
    {
        perror("post semop");
        return false;
    }
    return true;
}

int csemp::getvalue()
{
    return semctl(m_semid,0,GETVAL);
}

bool csemp::destroy()
{
    if(m_semid==-1)  return false;
    if(semctl(m_semid,0,IPC_RMID)==-1)
    {
        perror("destroy semctl");
        return false;
    }
    return true;
}

csemp::~csemp()
{

}


//Shared_Memory3.cpp
/*本程序演示用信号量给共享内存加锁。*/
#include "public.h"
#include <iostream>
#include <sys/ipc.h>
#include <sys/shm.h>
#include <sys/wait.h>
#include <unistd.h>
#include <semaphore.h>

using namespace std;

struct gril
{
    int no;
    char name[20];
};

int main(int argc,char* argv[])
{
    int shmid=shmget(0x5550,sizeof(gril),IPC_CREAT|0666);
    if(shmid==-1)
    {
        perror("shmget");
        return -1;
    }
    cout<<"shmid="<<shmid<<endl;
    //将共享内存绑定到当前进程的地址空间
    gril* p=(gril*)shmat(shmid,NULL,0);
    if(p==(void*)-1)
    {
        perror("shmat");
        return -1;
    }
    cout<<"shmat ok"<<endl;

    //加锁
    csemp mutex;
    mutex.init(0x5550);

    cout<<"正在加锁中"<<endl;
    mutex.wait();
    cout<<"加锁成功"<<endl;

    //操作共享内存
    p->no=atoi(argv[1]);
    strcpy(p->name,argv[2]);
    cout<<p->no<<" "<<p->name<<endl;
    sleep(20);

    //解锁
    mutex.post();
    cout<<"解锁成功"<<endl;

    //解除共享内存的绑定
    shmdt(p);

    //删除共享内存
    if(shmctl(shmid,IPC_RMID,NULL)==-1)
    {
        perror("shmctl");
        return -1;
    }
    return 0;
}
```

使用的编译命令：

```makefile
all: demo1

demo1:Shared_Memory3.cpp public.h public.cpp
 g++ -std=c++11 -o demo1  Shared_Memory3.cpp public.cpp
clean:
 rm -f demo1
```
