# 多进程下的生产消费者模型

## 什么是生产消费者模型

生产者-消费者模式是一种经典的并发编程模式，用于解决生产者与消费者之间进行数据交换与同步问题，在多线程/进程环境下，生产者负责生成数据并将其放入共享的数据缓冲区，而消费者负责从数据缓冲区中取出数据并进行处理，生产者和消费者通过共享的数据缓冲区进行通信，而我们这样做的原因是在实际生产中生产者与消费者的速度往往是不同的，而我们需要保证进程/线程之间的同步与协作

## 相关概念

- 生产者：负责生成数据并将其放入共享的数据缓冲区中
- 消费者:负责从共享的数据缓冲区中取出数据并进行处理
- 数据缓冲区：用于生产者和消费者之间进行数据交换的共享空间，通常是一个 队列或者缓冲区。

## 示例demo

```cpp
//生产者
#include "../public.h"

struct stgril
{
    int no;
    char name[20];
};

using ee=stgril;

int main()
{
    stgril s1,s2,s3;
    s1.no=1;
    s2.no=2;
    s3.no=3;
    strcpy(s1.name,"s1");
    strcpy(s2.name,"s2");
    strcpy(s3.name,"s3");
    //创建共享内存
    int shmid=shmget(0x5550,sizeof(squeue<stgril,5>),0666|IPC_CREAT);
    if(shmid<0)
    {
        perror("shmget failed");
        return -1;
    }

    //将共享内存连接到当前进程的地址空间
    squeue<stgril,5>* q=(squeue<stgril,5>*)shmat(shmid,0,0);
    if(q==(void*)-1)
    {
        perror("shmat failed");
        return -1;
    }
    q->init();
    csemp mutex;
    csemp cond;
    mutex.init(0x5001);
    cond.init(0x5002,0,0);
    mutex.wait();  //加锁
    q->push(s1);
    q->push(s2);
    q->push(s3);
    mutex.post();   //解锁
    cond.post(3);  //设置信号量,表示生产者生成3个数据

    shmdt(q);
    return 0;
}
```

```cpp
//消费者
#include "../public.h"
#include <unistd.h>

struct stgril
{
    int no;
    char name[20];
};

using st = stgril;

int main()
{
    // 获取共享内存
    int shmid = shmget(0x5550, sizeof(squeue<stgril, 5>), 0666 | IPC_CREAT);
    if (shmid < 0)
    {
        perror("shmget");
        return -1;
    }

    // 将共享内存连接到当前进程的地址空间
    squeue<stgril, 5> *p = (squeue<stgril, 5> *)shmat(shmid, 0, 0);
    if (p == (void *)-1)
    {
        perror("shmat");
        return -1;
    }

    p->init();
    csemp mutex;
    csemp cond;
    mutex.init(0x5001);
    cond.init(0x5002, 0, 0);
    while (true)
    {
        mutex.wait();
        while (p->empty())
        {
            mutex.post();
            cond.wait();
            mutex.wait();
        }
        st a = p->front();
        p->pop();
        mutex.post();
        cout << "no=" << a.no << " name=" << a.name << endl;
        sleep(5);
    }
    // 断开共享内存
    shmdt(p);
}
```
