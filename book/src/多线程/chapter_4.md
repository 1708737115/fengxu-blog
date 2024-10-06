# 条件变量与信号量

## 条件变量

### 什么是条件变量

条件变量是线程间同步的一种机制，它允许一个或多个线程在某些条件满足时被唤醒，从而继续执行。条件变量通常与互斥锁一起使用，以确保线程在访问共享资源时不会发生竞争条件,常见的条件变量主要有以下几种:

- condition-variable:提供与 std::unique_lock关联的条件变量
- condition_variable_any:提供与任何锁类型关联的条件变量

两者的主要区别在于，condition_variable_any可以凭借`unique_lock`来与任何类型的锁一起使用，而condition_variable只能与`std::mutex`一起使用。

### 条件变量的基本操作

条件变量的操作主要包括两个:

- wait:等待条件变量，直到条件满足
- notify_one/notify_all:通知等待条件变量的线程

关于这两个操作的常见api主要有下面几种:

- wait:等待条件变量，直到条件满足
- wait_for:等待条件变量，直到条件满足或超时
- wait_until:等待条件变量，直到条件满足或指定的时间点
- notify_one:通知一个等待条件变量的线程
- notify_all:通知所有等待条件变量的线程

### 基于条件变量的生产者消费者模型

下面我们来看一下如何使用上面所说的条件变量来实现一个生产者消费者模型(考虑到两个条件变量的用法基本一致，我们只以condition_variable为例):

```cpp
//test.h
#include <iostream>
#include <condition_variable>
#include <thread>
#include <chrono>
#include <mutex>
#include <list>
#include <functional>

using namespace std;
#define MAX_SIZE 100 // 任务队列的最大长度

template <typename T>
class SyncQueue
{
private:
    mutex m_mutex;                // 互斥锁
    condition_variable non_empty; // 判断当前生产者消费者模型中的任务队列是否为空
    condition_variable non_full;  // 判断当前生产者消费者模型中的任务队列是否为满
    list<T> m_queue;              // 任务队列
    int m_maxsize;                // 任务队列的最大长度
public:
    SyncQueue() : m_maxsize(MAX_SIZE) {} // 构造函数
    void put(T t)                        // 生产函数
    {
        unique_lock<mutex> lock(m_mutex); // 这里通过unique_lock来获取互斥锁，保证线程安全
        non_full.wait(lock, [this]()
                      { return m_queue.size() != m_maxsize; }); // 等待任务队列不满
        m_queue.push_back(t);
        cout << t << " 被生产" << endl; 
        non_empty.notify_one(); // 唤醒一个消费者线程
    }
    T take() // 消费函数
    {
        unique_lock<mutex> lock(m_mutex); // 这里通过unique_lock来获取互斥锁，保证线程安全
        non_empty.wait(lock, [this]()
                       { return m_queue.size() > 0; }); // 等待任务队列不为空
        T t = m_queue.front();
        m_queue.pop_front();
        non_full.notify_one(); // 唤醒一个生产者线程
         cout << t << " 被消费" << endl; 
        return t;
    }
    bool empty()
    {
        unique_lock<mutex> locker(m_mutex);
        return m_queue.empty();
    }

    bool full()
    {
        unique_lock<mutex> locker(m_mutex);
        return m_queue.size() == m_maxsize;
    }

    int size()
    {
        unique_lock<mutex> locker(m_mutex);
        return m_queue.size();
    }
    ~SyncQueue() {} // 析构函数
};

//test1.cpp
#include "./include/test.h"
int main()
{
    SyncQueue<int> q;
    auto produce=std::bind(&SyncQueue<int>::put,&q,std::placeholders::_1);
    auto consume=std::bind(&SyncQueue<int>::take,&q);
    thread t1[3],t2[3];
    for(int i=0;i<3;i++)
    {
        t1[i]=thread(produce,i+100);
        t2[i]=thread(consume);
    }
    for(int i=0;i<3;i++)
    {
        t1[i].join();
        t2[i].join();
    }
    return 0;
}
```

同时附上cmake文件(仅供参考):

```shell
# 设置打印编译信息
set(CMAKE_VERBOSE_MAKEFILE ON)

# 设置cmake编译器最低版本
cmake_minimum_required (VERSION 3.10)
 
project (learn_thread)

#头文件搜素路径
include_directories(${PROJECT_SOURCE_DIR}/include)

# 添加可执行文件
add_executable(demo1 test1.cpp)

# 添加链接库
link_libraries(demo1 PRIVATE Threads::Threads)

# 添加编译选项
add_compile_options(-c -g)
```

## 信号量

在多进程编程中我们就对信号量做了一定的介绍,在cpp20中，并发库对信号量做了封装实现,让我们对信号量有更方便的操作。

### 信号量的种类

cpp11中信号量分为两种,一种为计数信号量,一种为二值信号量。

- 计数信号量:计数信号量可以用于控制多个线程对共享资源的访问,当计数信号量大于0时,线程可以访问共享资源,当计数信号量等于0时,线程需要等待信号量释放。
- 二值信号量:二值信号量只能用于控制两个线程对共享资源的访问,当二值信号量等于1时,线程可以访问共享资源,当二值信号量等于0时,线程需要等待信号量释放。

其实本质上来说,二值信号量实现的是一个类似于互斥量的功能,但是二值信号量起到的更多的是一个信号的作用，他是通知线程之间的事件,比如线程是应该执行还是阻塞,而互斥量起到的更多的是一个锁的作用,他是保护共享资源不被多个线程同时访问。

而计数信号量则可以用于控制多个线程对共享资源的访问,当计数信号量大于0时,线程可以访问共享资源,当计数信号量等于0时,线程需要等待信号量释放。

### 信号量的api

虽然信号量的之类有二值和计数两种,但是他们的api都是一样的,只是初始化的参数不同,首先我们来看一下它们共有的api；

- release():释放信号量,将信号量的计数加1。
- acquire():获取信号量,将信号量的计数减1,如果计数小于0,则线程需要等待信号量释放。
- try_acquire():尝试获取信号量,将信号量的计数减1,如果计数小于0,则返回false,否则返回true。
- try_acquire_for():尝试获取信号量,将信号量的计数减1,如果计数小于0,则等待一段时间,如果等待时间到了,则返回false,否则返回true。
- try_acquire_until():尝试获取信号量,将信号量的计数减1,如果计数小于0,则等待直到指定的时间,如果等待时间到了,则返回false,否则返回true。

### 信号量的初始化

信号量的初始化有两种方式,一种是通过构造函数,一种是通过赋值操作符。

- 构造函数:构造函数需要传入一个整数,表示信号量的初始计数。

### 信号量的使用

首先是`binary_semaphore`,我们将演示如何使用`binary_semaphore`来控制两个线程对共享资源的访问。

```cpp
#include <iostream>
#include <thread>
#include <semaphore>
#include <chrono>
#include <mutex>
#include <random>

std::binary_semaphore binary_lock(1); // 初始状态为有信号
int count=0; // 共享变量

void add()
{
    for(int i=0;i<100000;i++)
    {
            binary_lock.acquire(); // 获取信号量，如果信号量为0，则阻塞当前线程(信号量的P操作)
        count++;
        binary_lock.release(); // 释放信号量，将信号量加1(信号量的V操作);
    }
}

int main()
{
    std::thread t1(add);
    std::thread t2(add);
    t1.join();
    t2.join();
    std::cout<<"count="<<count<<std::endl;
    return 0;
}
```

然后是`count_semaphore`,我们将演示如何用它配合`std::mutex`来控制对共享资源的访问,这里我们以生产者消费者模型为例:

```cpp
#include <iostream>
#include <queue>
#include <thread>
#include <chrono>
#include <mutex>
#include <condition_variable>
#include <random>

const int BUFFER_SIZE = 10;
std::queue<int> buffer;
std::mutex buffer_mutex;
std::condition_variable cond_var;
bool stop_threads = false;

// 生产者和消费者的逻辑保持不变，但是需要监听stop_threads标志。

void producer(int id) {
    int count = 0;
    std::random_device rd;
    std::mt19937 gen(rd());
    std::uniform_int_distribution<> dis(1000, 5000);

    while (!stop_threads) {
        std::unique_lock<std::mutex> lk(buffer_mutex);
        cond_var.wait(lk, [] { return buffer.size() < BUFFER_SIZE; }); // 等待缓冲区非满
        buffer.push(count);
        std::cout << "Produced: " << count << std::endl;
        count++;
        lk.unlock();
        cond_var.notify_one(); // 通知消费者
        std::this_thread::sleep_for(std::chrono::milliseconds(dis(gen)));
    }
}

void consumer(int id) {
    //定义随机数生成器
    std::random_device rd;
    std::mt19937 gen(rd());
    std::uniform_int_distribution<> dis(1000, 5000);

    while (!stop_threads) {
        std::unique_lock<std::mutex> lk(buffer_mutex);
        cond_var.wait(lk, [] { return !buffer.empty(); }); // 等待缓冲区非空
        int value = buffer.front();
        buffer.pop();
        std::cout << "Consumed: " << value << std::endl;
        lk.unlock();
        cond_var.notify_one(); // 通知生产者
        std::this_thread::sleep_for(std::chrono::milliseconds(dis(gen)));  // 模拟消费时间
    }
}

int main() {
    std::thread prod(producer, 0);
    std::thread cons(consumer, 0);

    prod.join();
    cons.join();

    // 设置停止标志并唤醒所有等待中的线程
    stop_threads = true;
    cond_var.notify_all();

    return 0;
}
```

**备注:**

- 由于上面的有关信号量的信号量封装类是cpp20的产物，所以我们编译的时候药品注意编译器版本，编译命令可以参考:

```shell
g++ -std=c++20 -o test test2.cpp
```

- 在信号量的初始化函数中其实STL标准库中只有

```cpp
std：：counting_semaphore<LeastMaxValue>：：counting_semaphore
```

在源码中关于`binary_semaphore`的初始化函数是这样的:

```cpp
using binary_semaphore = std::counting_semaphore<1>;
```

所以两种信号量本质上用法没有什么太大区别,只不过`binary_semaphore`是`count_semphore`的一个特例,我们在使用时

```cpp
std::binary_semaphore binary_lock(1);
```

也不过是初始化二值信号量的状态。
