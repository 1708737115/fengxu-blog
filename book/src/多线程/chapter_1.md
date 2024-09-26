# 初窥多线程(二) 基于C语言实现的多线程编写

## 前言
在上一篇文章中我们介绍了在计算机底层视角下的虚拟内存和操作系统在用户层所进行的各个分层，在这篇文章我们就要开始尝试书写多线程代码了,其实在c++11后c++就提供供了线程类给我们使用,c++线程类其实主要是对c操作多线程的函数进行了封装，本质上其实是一致的，所以在讲解我们cpp的多线程编写之前，我觉得先来了解一下C语言是如何实现多线程的编写的，这样可以让我们更好的去理解cpp线程类的工作原理，话不多说，发车发车！

## 线程的创建
在之前我们讲解Linux下的进程控制时说过我们在创建进程时进程都会有自己的进程编号`pid`,而线程和它们一样，每一个线程都有唯一的线程编号,它的类型为`pthread_t`,它是一个无符号长整形数,我们可以调用下面这个函数来获取当前线程的线程编号:
```cpp
pthread_t ptread_self(void);  //返回当前线程的线程编号
```
如果我们希望在一个进程中创建子线程,就要调用线程创建函数,但是和进程不同,我们必须要给每一个线程指定一个线程处理函数，否则线程将无法正常工作,处理函数的定义如下:
```cpp
int pthread_create(pthread_t *thread, const pthread_attr_t *attr,
                   void *(*start_routine) (void *), void *arg);
```
- 参数:
	- thread: 传出参数，是无符号长整形数，线程创建成功, 会将线程ID写入到这个指针指向的内存中
	- attr: 线程的属性, 一般情况下使用默认属性即可, 写NULL
	- start_routine: 函数指针，创建出的子线程的处理动作，也就是该函数在子线程中执行。
	- arg: 作为实参传递到 start_routine 指针指向的函数内部
- 返回值：线程创建成功返回0，创建失败返回对应的错误号

下面我们来看一个线程创建的实例:
```cpp
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<string.h>
#include <pthread.h>


// 线程的处理函数
void* work(void* arg)
{
    printf("子线程id=%ld\n",pthread_self());
    for(int i=0;i<9;i++)
    {
        printf("child id=%d\n",i);
    }
    return NULL;
}

int main(int argc,char* argv[])
{
    pthread_t tid;
    pthread_create(&tid,NULL,work,NULL);
    printf("主线程id=%ld\n",pthread_self());
    for(int i=0;i<3;i++)
    {
        printf("main id=%d\n",i);
    }
    sleep(10);
    return 0;
}
```
注意：
我们在Linux下编译该代码要导入线程库。编译命令如下:
```makefile
all: demo1

demo1: Create_Thread.c

	gcc -pthread -o demo1 Create_Thread.c 

clean:
	rm -f demo1
```
运行结果如下：
```text
root@iZuf6ckztbjhtavfplgp0dZ:~/mylib/cppdemo/Linux系统编程/多线程/threads(c)# ./demo1
主线程id=139930484275008
main id=0
main id=1
main id=2
子线程id=139930484270848
child id=0
child id=1
child id=2
child id=3
child id=4
child id=5
child id=6
child id=7
child id=8
```

如果我们去除`sleep`函数的使用我们会发现结果是这样的:
```text
主线程id=140150867830592
main id=0
main id=1
main id=2
```
这是因为虚拟地址生存周期是和主线程保持一致的，与子线程无关，主线程提前结束，导致哪怕子线程还没有开始运行，程序也自动停止运行了，这里的`sleep`函数所起到的作用主要就是线程同步。

## 线程的退出
在我们编写多线程代码时，如果我们希望让线程退出，但是不希望因此导致虚拟地址空间的释放(主线程突出时会释放)，这时候我们可以调用线程退出函数，这样线程的退出就不会影响其他线程的正常使用了，函数定义如下:
```cpp
void pthread_exit(void* retval)
```

-` retval`：线程退出的时候携带的数据，当前子线程的主线程会得到该数据。如果不需要使用，指定为NULL

下面我们来看一个简单的示例:
```cpp
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <pthread.h>

void* work(void* arg)
{
    sleep(5);
    printf("子线程运行中,子线程id:%ld\n",pthread_self());\
    for(int i=1;i<9;i++)
    {
        printf("id=:%d\n",i);
        if (i==6)
        {
            pthread_exit(NULL);
        }
    }
    return NULL;
}

int main(int argc,int *argv[])
{
    pthread_t tid;
    pthread_create(&tid,NULL,work,NULL);

    printf("主线程运行中,主线程id:%ld\n",pthread_self());
    for(int i=0;i<3;i++)
    {
        printf("id=:%d\n",i);
    }
    pthread_exit(NULL);//主线程退出
    return 0;
}
```
输出结果为:
```text
主线程运行中,主线程id:140161600366400
id=:0
id=:1
id=:2
子线程运行中,子线程id:140161600362240
id=:1
id=:2
id=:3
id=:4
id=:5
id=:6
```
我们可以看到虽然主线程提前推出了，但是却并没有影响到子线程的运行。

## 线程回收
### 线程回收函数
进程和线程一样，子线程退出的时候它的内核资源是由主线程来回收,线程回收的函数是`pthread_join()`,该函数是阻塞函数，当有子线程正在运行,调用该函数会阻塞,直到所有子线程都退出后才能进行子线程资源的回收，一次只能回收一个子线程的资源，如果我们有多个子线程资源需要回收，需要借助循环来完成,函数原型如下:
```cpp
int pthread_join(pthread_t pid,void** retval);
```
参数说明：

- pid:要被回收的线程编号
- retval:二级指针,指向一级指针,是一个传出参数 ,一级指针里面储存的是`pthread_exit()`函数传出的数据，如果不需要该数据可以设为`NULL`
- 返回值：线程回收成功返回0，回收失败返回错误号。

### 回收子线程数据的实现方式
在子线程退出的时候可以通过`pthread_join`函数来将数据传出,我们在回收子线程的同时也可以接收子线程的数据这样的实现方法有很多种，下面我们来看下面几种:
**备注**：
导致实现方法多样性的原因：	通过上面的介绍，我们知道子线程在被回收的时候会将数据写入到一块内存中,然后采纳数传出该内存的地址而非是存储数据本身，而传出的参数类型是`void*`,这个万能指针可以指向任意一块内存，也导致了我们可以通过不同的形式来接收子线程数据。
1. **使用子线程栈**
在我们接收子线程数据的时候可以通过子线程栈来回收子线程数据，示例代码如下:

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <pthread.h>

typedef struct Person
{
    int id;
    char* name;
    int age;
}Person;

void* work(void* arg)
{
    printf("子线程id:%ld",pthread_self());
    for(int i=0;i<9;i++)
    {
        printf("id=%d\n",i);
        if (i==6)
        {
            Person p;
            p.id=1;
            p.name="张三";
            p.age=20;
            pthread_exit((void*)&p);
        }
    }
    return NULL;
}

int main(int argc,char* argv[])
{
    pthread_t tid;
    if (pthread_create(&tid,NULL,work,NULL)!=0)
    {
        printf("线程创建失败");
        return -1;
    }
    printf("主线程id:%ld\n",pthread_self());
    void *ptr;
    pthread_join(tid,&ptr);
    struct Person* p=(struct Person*)ptr;
    printf("id=%d\n",p->id);
    printf("name=%s\n",p->name);
    printf("age=%d\n",p->age);
    printf("子线程数据成功接收\n");
    return 0;
}
```
我们编译运行后结果如下：
```cpp
主线程id:139934351951680
子线程id:139934351947520id=0
id=1
id=2
id=3
id=4
id=5
id=6
id=0
name=
age=22476544
子线程数据成功接收
```
我们可以发现在主线程中并没有子线程的数据，这是因为当子线程退出时，子线程所占据的栈区就会被回收，进而导致了子线程想要传递的数据被释放掉了，所以说我们一般不会采取子线程栈来接收数据，而是使用其他方式来接收数据。
2. **使用全局变量**
 
```cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <pthread.h>

typedef struct Person
{
    int id;
    char* name;
    int age;
}Person;

Person p;

void* work(void* arg)
{
    printf("子线程id:%ld",pthread_self());
    for(int i=0;i<9;i++)
    {
        printf("id=%d\n",i);
        if (i==6)
        {
            p.id=1;
            p.name="张三";
            p.age=20;
            pthread_exit((void*)&p);
        }
    }
    return NULL;
}

int main(int argc,char* argv[])
{
    pthread_t tid;
    if (pthread_create(&tid,NULL,work,NULL)!=0)
    {
        printf("线程创建失败");
        return -1;
    }
    printf("主线程id:%ld\n",pthread_self());
    void *ptr;
    pthread_join(tid,&ptr);
    struct Person* p=(struct Person*)ptr;
    printf("id=%d\n",p->id);
    printf("name=%s\n",p->name);
    printf("age=%d\n",p->age);
    printf("子线程数据成功接收\n");
    return 0;
}
```
输出结果为:
```text
主线程id:140702000142144
子线程id:140702000137984id=0
id=1
id=2
id=3
id=4
id=5
id=6
id=1
name=张三
age=20
子线程数据成功接收
```
3. **使用主线程栈**
虽然线程之间有自己的栈空间，但是它们彼此之间也可以互相访问，而一般主线程都是最后退出的，所以我们可以尝试把子线程返回的数据保存到了主线程的栈区内存中。

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <pthread.h>

typedef struct Person
{
    int id;
    char* name;
    int age;
}Person;


void* work(void* arg)
{
    Person *p=(Person*)arg;
    printf("子线程id:%ld",pthread_self());
    for(int i=0;i<9;i++)
    {
        printf("id=%d\n",i);
        if (i==6)
        {
            p->id=1;
            p->name="张三";
            p->age=20;
            pthread_exit((void*)&p);
        }
    }
    return NULL;
}

int main(int argc,char* argv[])
{
    Person p;
    pthread_t tid;
    if (pthread_create(&tid,NULL,work,&p)!=0)
    {
        printf("线程创建失败");
        return -1;
    }
    printf("主线程id:%ld\n",pthread_self());
    void *ptr;
    pthread_join(tid,&ptr);
    printf("id=%d\n",p.id);
    printf("name=%s\n",p.name);
    printf("age=%d\n",p.age);
    printf("子线程数据成功接收\n");
    return 0;
}
```

## 线程分离
在一些情况下，程序中的主线程会拥有自己的业务处理流程，如果让主线程负责子线程的资源回收，那么调用`pthread_join`函数在子线程全部结束前主线程会一直阻塞，这时候我们可使用线程分离函数来将该线程剥离出来，调用该函数后子线程会与主线程分离，但是这样`pthread_join`就接收不到子线程资源了，线程分离函数定义如下:
```cpp
int pthread_detach(pthread_t id);
```
示例如下:
```cpp
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <pthread.h>

// 子线程的处理代码
void* working(void* arg)
{
    printf("我是子线程, 线程ID: %ld\n", pthread_self());
    for(int i=0; i<9; ++i)
    {
        printf("child == i: = %d\n", i);
    }
    return NULL;
}

int main()
{
    //创建一个子线程
    pthread_t tid;
    pthread_create(&tid, NULL, working, NULL);

    printf("子线程创建成功, 线程ID: %ld\n", tid);
    // 2. 子线程不会执行下边的代码, 主线程执行
    printf("我是主线程, 线程ID: %ld\n", pthread_self());
    for(int i=0; i<3; ++i)
    {
        printf("i = %d\n", i);
    }

    // 设置子线程和主线程分离
    pthread_detach(tid);

    // 让主线程自己退出即可
    pthread_exit(NULL);
    
    return 0;
}
```

## 一些其他的线程函数

### 线程取消
线程取消指的是我们可以在一个线程中调用它来取消另一个线程，函数定义如下:
```cpp
int pthread_cancel(pthread_t pid);
```
代码示例如下：
```cpp
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <pthread.h>

// 子线程的处理代码
void* work(void* arg)
{
    printf("我是子线程, 线程ID: %ld\n", pthread_self());
    for(int i=0; i<9; ++i)
    {
        sleep(1);
        printf("child == i: = %d\n", i);
    }
    return NULL;
}

int main()
{
    //创建一个子线程
    pthread_t tid;
    pthread_create(&tid, NULL, work, NULL);

    printf("子线程创建成功, 线程ID: %ld\n", tid);
    // 2. 子线程不会执行下边的代码, 主线程执行
    printf("我是主线程, 线程ID: %ld\n", pthread_self());
    for(int i=0; i<3; ++i)
    {
        sleep(1);
        printf("i = %d\n", i);
    }

    // 设置子线程和主线程分离
    pthread_cancel(tid);

    // 让主线程自己退出即可
    pthread_exit(NULL);
    
    return 0;
}
```
输出如下:
```text
我是主线程, 线程ID: 139763539765056
我是子线程, 线程ID: 139763539760896
i = 0
child == i: = 0
i = 1
child == i: = 1
i = 2
child == i: = 2
```
注意：线程的取消分两步：

- 主线程基于线程取消函数发送请求
- 当子线程再次进行系统调用时，线程会被取消(没有这一步，线程就还存在)

## 结语
关于C语言库中关于线程的函数介绍到此就告一段落了，下一篇文章我们就要开始介绍cpp中一些关于线程的知识了，下篇见!
 
