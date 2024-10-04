# 初窥多线程(四) 线程同步与锁

## 什么是线程同步

线程同步是指多个线程在执行过程中，由于共享资源，导致数据不一致的问题。线程同步是为了解决多个线程同时访问共享资源时，由于线程切换导致的数据不一致问题,大家可能不是很理解为什么要线程同步，我们举个简单的例子:

```cpp
#include <iostream>
#include <thread>

using namespace std;

int counter=0;

void* ADD_COUNT()
{
    for(int i=0;i<100000;i++)
    {
        counter++;
    }
}

int main()
{
    thread a(ADD_COUNT);
    thread b(ADD_COUNT);
    a.join();
    b.join();
    cout<<"counter="<<counter<<endl;
}
```

上面代码我这里的运行结果是161335,而不是200000,这就是线程同步的问题,这里我们的counter是共享资源,而a,b两个线程同时访问counter,导致数据不一致,讲的通俗一点，在某个时刻，a和b线程都读到了Count=n,这是它们都会执行count=n+1,然后a线程执行完了，b线程执行完了，最终的结果是n+1,而不是n+2,导致最后的结果小于200000，而为了解决这个问题，我们就需要线程同步，让a,b两个线程不能同时访问共享资源，这样就可以才能保证数据的一致性。

## 如何实现线程同步

在讲解如何进行线程同步之前,我们要进行线程同步主要是为了防止多个线程同时访问共享资源而造成混乱,所以无论我们采用什么样的手段目的只有一个:保证共享资源在同一时刻只有一个线程访问.

那什么是共享资源呢?所谓的共享资源就是多个线程共同访问的变量，这些变量通常为全局数据区变量或者堆区变量，这些变量对应的共享资源也被称之为临界资源。我们在进行线程同步时所需要做的就是找到这些临界资源，然后保证同一时刻只有一个线程访问这些临界资源.

常见实现线程同步的方法主要有下面几种:

- 锁
- 条件变量
- 信号量

而今天我们介绍的就是——锁.

## 锁

### 锁的概念

锁，在我们日常生活中是用来保护我们的私有财产保证其不被他人侵犯，而在计算机中，锁也用来保护资源的，它用来保护共享资源不被多个线程同时访问，保证共享资源在同一时刻只有一个线程访问，从而保证数据的一致性。

### 互斥锁

#### 互斥锁的原理

互斥锁是一种用于多线程编程中，防止多个线程同时访问共享资源的同步机制。互斥锁可以确保在任何时刻只有一个线程可以访问共享资源，从而避免了数据竞争和一致性问题。它的原理很简单:首先一个共享资源开始是没有锁的，当一个线程想要访问这个共享资源的时候，它会先尝试看这个资源有没有锁,如果没有就可以正常对这个共享资源进行操作，如果有锁，那么这个线程就只能等待，直到锁被释放为止。

#### 互斥锁的种类

cpp11给我们提供了四种互斥锁的类型:

```cpp
std::mutex mtx; //最简单的互斥锁,不能递归使用
std::recursive_mutex r_mtx; //递归互斥锁
std::timed_mutex t_mtx; //定时互斥锁,可以设置等待时间
std::recursive_timed_mutex rt_mtx; //定时递归互斥锁
```

我们接下来依次来看看这四种锁的使用方法:

- `mutex`

```cpp
#include <iostream>
#include <mutex>
#include <chrono>
#include <thread>

using namespace std;

int counter=0;
mutex mtx; //互斥锁

void* ADD_COUNT()
{
    for(int i=0;i<100000;i++)
    {
        mtx.lock(); //加锁,若加锁失败线程会阻塞  除此之外我们还可以加try_lock()尝试加锁，如果加锁失败，则不会阻塞
        counter++;
        mtx.unlock(); //解锁,若解锁失败，则线程会阻塞
    //     this_thread::sleep_for(chrono::milliseconds(1)); //休眠1毫秒
    }
}

int main()
{
    thread a(ADD_COUNT);
    thread b(ADD_COUNT);
    a.join();
    b.join();
    cout<<"counter="<<counter<<endl;
}
```

上面就是一个简单的mutex的例子，我们创建了两个线程，然后让这两个线程去增加一个共享资源，但是这个共享资源被一个互斥锁保护着，所以这两个线程不能同时访问这个共享资源，只能一个一个的来，这样就保证了数据的一致性。而除此之外还有一些其他的函数api:

1. `try_lock`尝试加锁，如果加锁失败，则不会阻塞
2. `lock`加锁，如果加锁失败，则线程会阻塞
3. `unlock`解锁，如果解锁失败，则线程会阻塞

- timed_mutex
相对于`mutex`,`timed_mutex`多了一个等待时间，如果等待时间到了，还没有加锁成功，那么就会返回false，否则就会返回true,并且会自动解锁

```cpp
#include <iostream>
#include <mutex>
#include <chrono>
#include <thread>

using namespace std;

chrono::seconds timeout(1); // 超时时间
timed_mutex mtx;

void work()
{
    while(true)
    {
        if(mtx.try_lock_for(timeout))
        {
            cout<<"线程"<<this_thread::get_id()<<"获取锁"<<endl;
            this_thread::sleep_for(chrono::seconds(10));
            mtx.unlock();
            break;
        }
        else
        {
           cout<<"线程"<<this_thread::get_id()<<"正在尝试获取锁......."<<endl;
           this_thread::sleep_for(chrono::seconds(1));
        }
    }
}

int main()
{
    thread t1(work);
    thread t2(work);
    t1.join();
    t2.join();
    return 0;
}
```

还有一些其他的api:

1. `try_lock_until`尝试加锁，如果加锁失败，则不会阻塞，直到指定的时间点

- recursive_mutex
递归互斥锁std::recursive_mutex允许同一线程多次获得互斥锁，可以用来解决同一线程需要多次获取互斥量时死锁的问题，在下面的例子中使用独占非递归互斥量会发生死锁:

```cpp
#include <iostream>
#include <mutex>
#include <chrono>
#include <thread>

using namespace std;

class Calculate
{
    int m_i;
    mutex mtx;

public:
    Calculate():m_i(6){}
    void mul()
    {
        mtx.lock();
        m_i *= 2;
    }
    void div()
    {
        mtx.lock();
        m_i /= 2;
    }
    void test()
    {
        mul();
        div();
    }
};

int main()
{
    Calculate c;
    c.test();
    return 0;
}
```

当我们尝试获取多个锁的时候会出现死锁的情况(什么是死锁我们后面会介绍)，所以使用递归锁可以解决这个问题:

```cpp
#include <iostream>
#include <mutex>
#include <chrono>
#include <thread>

using namespace std;

class Calculate
{
    int m_i;
    recursive_mutex mtx;

public:
    Calculate():m_i(6){}
    void mul()
    {
        mtx.lock();
        m_i *= 2;
    }
    void div()
    {
        mtx.lock();
        m_i /= 2;
    }
    void test()
    {
        mul();
        div();
        show();
    }
    void show()
    {
        cout << m_i << endl;
    }
};

int main()
{
    Calculate c;
    c.test();
    return 0;
}
```

- resursive_timed_mutex
这个锁和recursive_mutex类似，但是它提供了超时机制，如果获取锁超时了，就会返回false，否则返回true,这个就不做过多介绍大家可以参考`mutex`和`timed_mutex`的用法。

### 读写锁

#### 读写锁的介绍

读写锁是一种特殊的互斥锁，我们可以将读写锁看做读锁与写锁，写锁允许多个线程同时读取共享资源，但是只允许一个线程写入共享资源。写锁在写入共享资源时，会阻塞其他线程的读取操作。这样可以保证数据的一致性，
同时避免了多个线程同时同时对共享资源进行不同操作导致的冲突，同时多个线程可以同时读取共享资源，提高了并发性能。

读写锁的实现原理是使用一个计数器来记录当前有多少个线程正在读取/写入共享资源，如果计数器为0，那么就可以写入/读取共享资源，否则就不能写入/读取共享资源。

#### 读写锁的用法

读写锁的常用函数主要有以下几个:

```cpp
- pthread_rwlock_t rwlock_init(void); // 初始化读写锁
- int pthread_rwlock_destroy(pthread_rwlock_t *rwlock); // 销毁读写锁
- int pthread_rwlock_rdlock(pthread_rwlock_t *rwlock); // 获取读锁
- int pthread_rwlock_wrlock(pthread_rwlock_t *rwlock); // 获取写锁
- int pthread_rwlock_unlock(pthread_rwlock_t *rwlock); // 释放锁
```

下面是一个简单的读写锁的例子:

```cpp
#include <iostream>
#include <thread>
#include <vector>
#include <chrono>
#include <random>
#include <pthread.h>

pthread_rwlock_t rwlock;
int count = 0;
std::default_random_engine generator;
std::uniform_int_distribution<int> distribution(0, 4999);

void read(int id) {
    pthread_rwlock_rdlock(&rwlock); // 获取读锁
    std::cout << "线程 " << id << " 拿到读锁" << std::endl;
    std::this_thread::sleep_for(std::chrono::milliseconds(distribution(generator)));
    std::cout << "线程 " << id << " 释放读锁" << std::endl;
    pthread_rwlock_unlock(&rwlock); // 释放读锁
}

void write(int id) {
    pthread_rwlock_wrlock(&rwlock); // 获取写锁
    std::cout << "线程 " << id << " 拿到写锁" << std::endl;
    int temp = count;
    std::this_thread::sleep_for(std::chrono::milliseconds(distribution(generator) % 1000));
    count = temp + 1;
    std::cout << "线程 " << id << " 释放写锁 " << count << std::endl;
    pthread_rwlock_unlock(&rwlock); // 释放写锁
}

void runThreads(int readCount, int writeCount) {
    std::vector<std::thread> threads;

    for (int i = 0; i < readCount; ++i) {
        threads.emplace_back(read, i);
    }

    for (int i = 0; i < writeCount; ++i) {
        threads.emplace_back(write, i + readCount);
    }

    for (auto& th : threads) {
        th.join();
    }
}

int main() {
    pthread_rwlock_t rwlock;
    pthread_rwlock_init(&rwlock, NULL);

    const int readCount = 7;
    const int writeCount = 3;
    runThreads(readCount, writeCount);
    std::cout << "最终结果: " << count << std::endl;

    pthread_rwlock_destroy(&rwlock);
    return 0;
}
```

### lock_guard以及unique_lock

#### lock_guard

互斥锁的获取和释放必须成对出现，而一旦出现线程在获取互斥锁而没有释放时，其它线程将永远等待！（使用 lock() 的情况下）为了解决此类情况，C++ 标准库提供了一个 RAII 式(什么是RAII我们后面会有所介绍)的方案，即模板类 lock_guard。
该类需要传入一个泛型参数，表示锁的类型，同时该类需要传入一个具体的锁，这样你仅需要初始化一个 lock_guard, 线程将自动获取锁，同时在线程结束时，临时变量被销毁，同时锁也会被释放。

```cpp
#include <iostream>
#include <mutex>
#include <thread>

using namespace std;

int a = 1;
mutex a_mutex;

void increase() {
    lock_guard<mutex> lg(a_mutex);
    this_thread::sleep_for(2000ms);
    ++a;
}

int main() {
    thread t1(increase);
    thread t2(increase);
    t1.join();
    t2.join();
    cout << a << endl; // 3
    return 0;
}
```

#### unique_lock

上面我们已经介绍了`lock_guard`,lock_guard 虽然方便，却仍有其不便之处，那就是锁的粒度太大了，在 lock_guard 中，上锁发现在初始化过程，而释放锁则需要在对象被销毁时，这样显然是不够灵活的。所以这里我们要介绍一个更灵活的模板类 unique_lock，它允许我们自定义锁的粒度，同时它还支持 RAII 的特性，即锁的获取和释放。

它具有下面这些特性：

1. 延迟锁定（deferred locking）
2. 尝试锁定 (attempts at locking)
3. 超时机制 (time-constrained)
4. 递归锁定 (recursive locking)

- 转移互斥锁的所有权 (transfer of lock ownership)
- 与条件变量一起使用 (use with condition variables)

不过`unique_lock`只是一个灵活的锁管理器，它并不能直接对互斥锁进行上锁和解锁操作，它需要与互斥锁配合使用,对锁进行包装,接下来我们来看一下他是怎么实现它的这些特性的:

- 延迟锁定

```cpp
void increase() {
     unique_lock<mutex> ul(a_mutex, defer_lock); // 延迟锁定,初始化时并不上锁，需要我们手动上锁
     this_thread::sleep_for(2000ms);
     ul.lock(); // 手动上锁
     ++a;
}

```

- 尝试锁定

```cpp
void increase() {
     unique_lock<mutex> ul(a_mutex, try_to_lock); // 尝试锁定,如果锁已经被其他线程占用，则返回false，否则返回true
     if(!ul.owns_lock()) {
         this_thread::sleep_for(2000ms);
         cout<<"failed to get mutex"
     }
     ++a;
}
```

- 超时机制

```cpp
void increase() {
     unique_lock<timed_mutex> ul(a_mutex, defer_lock); 
     if(!ul.try_lock_for(2s)) {
         cout<<"timeout";<<endl;
     }
     ++a;
}
```

- 递归锁定

```cpp
  void once() {
        unique_lock<recursive_mutex> ul(m);
        ++shared;
        cout << "once\n";
    }

    void twice() {
        unique_lock<recursive_mutex> ul(m);
        for (int i = 0; i < 2; i++) {
            cout << "twice: ";
            once();
        }
    }
```

- 移交所有权

```cpp
void func() {
    unique_lock ul(m);
    unique_lock ul2 = move(ul); // 移动构造，此时锁的所有权已经转移给ul2
    ul2.swap(ul);               // 交换所有权,结束后ul有锁，ul2不再拥有锁
    ul.release();               // 释放所有权，此时ul不再拥有锁
    ++a;
    m.unlock(); // unlock the m mutex manually
}
```

### shared_mutex

shared_mutex 是 C++17 引入的新的互斥锁类型，它允许多个线程同时拥有读权限，但只有一个线程可以拥有写权限。shared_mutex 提供了以下功能：

- lock()：获取独占锁，阻塞直到成功获取锁。
- try_lock()：尝试获取独占锁，如果失败则立即返回 false。
- unlock()：释放独占锁。
- lock_shared()：获取共享锁，阻塞直到成功获取锁。
- try_lock_shared()：尝试获取共享锁，如果失败则立即返回 false。
- unlock_shared()：释放共享锁。

这里我们就不得不说一下什么是独占锁和共享锁了

和我们日常生活的独占与共享一样,独占锁就是只能一个线程访问，而共享锁就是多个线程可以访问，当我们读取的时候，可以多个线程同时读取，但是写入的时候，只能有一个线程写入，其他线程需要等待写入完成才能继续写入。在读多写少的场景下，shared_mutex 可以提高并发性能。

示例:

```cpp
#include <iostream>
#include <thread>
#include <mutex>
#include <shared_mutex>

using namespace std;

int a = 1;
shared_mutex sm;

int read() {
    shared_lock<shared_mutex> sl(sm);
    cout << "read\n";
    this_thread::sleep_for(2s);
    return a;
}

int main() {
    thread t1(read);
    thread t2(read);
    thread t3(read);
    thread t4(read);
    t1.join();
    t2.join();
    t3.join();
    t4.join();
    return 0;
}
```

大家可以尝试运行一下代码，会发现2szuong 4 个线程同时读取了 a 的值，而不是是按照顺序读取的。

## 死锁问题

### 死锁的产生原因

死锁的原因在于两个线程同时获取多个锁且存在获取的锁被对方占用的时候，这个时候线程将永远处于阻塞状态，比如下面这样:

```cpp
#include <iostream>
#include <mutex>
#include <thread>

using namespace std;

int a = 1, b = 1;
mutex a_mutex, b_mutex;


void increase1() {
    lock_guard<mutex> lg1(a_mutex);
    this_thread::sleep_for(1000ms);
    ++a;
    lock_guard<mutex> lg2(b_mutex);
    ++b;
}
void increase2() {
    lock_guard<mutex> lg1(b_mutex);
    this_thread::sleep_for(1000ms);
    ++b;
    lock_guard<mutex> lg2(a_mutex);
    ++a;
}


int main() {
    thread t1(increase1);
    thread t2(increase2);
    t1.join();
    t2.join();
    cout << a << endl << b;
    return 0;
}
```

在上面的代码中t1在获取完a锁，t2在获取完b锁之后,t1需要获取b锁，t2需要获取a锁，但是a锁和b锁都被对方占用，导致两个线程都处于阻塞状态，程序将永远无法结束,而这也就是我们所说的死锁了。

### 预防死锁的方法

- 避免多次锁定, 多检查

- 对共享资源访问完毕之后, 一定要解锁，或者在加锁的使用 trylock

- 如果程序中有多把锁, 可以控制对锁的访问顺序(顺序访问共享资源，但在有些情况下是做不到的)，另外也可以在对其他互斥锁做加锁操作之前，先释放当前线程拥有的互斥锁。

- 项目程序中可以引入一些专门用于死锁检测的模块

## 参考文章与链接

[线程同步](https://subingwen.cn/linux/thread-sync/)

[cppreference](https://zh.cppreference.com/w/cpp/thread/resursive_timed_mutex)
