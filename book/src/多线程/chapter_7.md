## 什么是池化技术

池化技术是一种资源管理策略，它通过重复利用已存在的资源来减少资源的消耗，从而提高系统的性能和效率。在计算机编程中，池化技术通常用于管理线程、连接、数据库连接等资源。

我们会将可能使用的资源预先创建好，并且将它们创建在一个池中，当需要使用这些资源时，直接从池中获取，使用完毕后再将它们归还到池中，而不是每次都创建和销毁资源。

池化技术的引用场景十分广泛,例如线程池、数据库连接池、对象池等，今天我们主要要探讨的就是线程池

## 什么是线程池

线程池是一种典型的池化技术的应用，在我们日常使用多线程来处理任务时，如果每次都创建和销毁线程,频繁的创建与销毁线程会出现大量不必要的资源消耗，降低系统的性能。而在线程池中我们可以预先创建一定数量的线程，当需要执行任务时，直接从线程池中获取线程来执行任务，任务执行完毕后，线程并不会被销毁，而是继续保留在线程池中，等待下一次任务的执行,通过这种线程复用的方式，可以大大减少线程的创建和销毁，从而提高系统的性能和效率。

## 线程池的优点

- 避免频繁创建与销毁线程：线程池预先创建并维护一定数量的工作线程，避免了频繁创建和销毁线程带来的系统开销，特别是在处理大量短生命周期任务时，效果尤为显著。

- 负载均衡与缓存局部性：线程池可以根据任务负载动态调整线程工作状态，避免过度竞争和闲置。同时，线程在执行任务过程中可以充分利用CPU缓存，提高执行效率。

- 控制并发级别：通过限制线程池大小和任务队列容量，可以有效控制系统的并发级别，防止因过度并发导致的资源争抢和性能下降。

- 简化编程模型
  - 线程池提供了一种简化的编程模型，开发者无需关心线程的创建、管理和销毁，只需将任务提交给线程池即可。这大大简化了多线程编程的复杂性，提高了开发效率。

  - 线程池还提供了一些高级功能，如任务优先级、任务超时、任务取消等，这些功能可以帮助开发者更好地管理任务执行过程，提高系统的可靠性和稳定性。

## 线程池的组成部分

该线程池类的主要有以下组成部分

- 线程池类
线程池类主要负责管理线程池的状态并根据线程池的状态淘汰的管理工作线程的状态并且实现任务的异步提交
  - 工作线程类
  - 有关线程池状态的枚举类
  - 任务队列`task_queue`
  - 用来存放工作线程的列表`worker_list`
  - 控制线程池状态/工作线程列表/以及任务队列的互斥锁
  - 用于唤醒/阻塞线程的条件变量
  - 基于线程异步实现的任务提交函数
  相关定义如下:

  ```cpp
    class ThreadPool
    {
    private:
        class worker_thread; // 工作线程类
        enum class status_t : std::int8_t
        {
            TERMINATED = -1,
            TERMINATING = 0,
            RUNNING = 1,
            PAUSED = 2,
            SHUTDOWN = 3
        }; // 线程池的状态: 已终止:-1,正在终止:0,正在运行:1,已暂停:2,等待线程池中任务完成,但是不接收新任务:3
        std::atomic<status_t> status;                    // 线程池的状态
        std::atomic<std::size_t> max_task_count;         // 线程池中任务的最大数量
        std::shared_mutex status_mutex;                  // 线程池状态互斥锁
        std::shared_mutex task_queue_mutex;              // 任务队列的互斥锁
        std::shared_mutex worker_lists_mutex;            // 工作线程列表的互斥锁
        std::condition_variable_any task_queue_cv;       // 任务队列的条件变量
        std::condition_variable_any task_queue_cv_full;  // 任务队列满的条件变量
        std::condition_variable_any task_queue_cv_empty; // 任务队列空的条件变量
        std::queue<std::function<void()>> task_queue;    // 任务队列,其中存储待执行的任务
        std::list<worker_thread> worker_lists;           // 工作线程列表
        // 考虑到为了确保线程池的唯一性和安全性,禁止使用拷贝赋值与移动赋值
        ThreadPool(ThreadPool &) = delete;
        ThreadPool &operator=(ThreadPool &) = delete;
        ThreadPool(ThreadPool &&) = delete;
        ThreadPool &operator=(ThreadPool &&) = delete;
        // 在取得对状态变量的访问权后,调用下列函数来改变线程池的状态
        void pause_with_status_lock();                            // 暂停线程池
        void resume_with_status_lock();                           // 恢复线程池
        void shutdown_with_status_lock();                         // 立刻关闭线程池
        void shutdown_with_wait_status_lock(bool wait_for_tasks); // 等待任务执行完毕关闭线程池
        void terminate_with_status_lock();                        // 终止线程池
        void wait_with_status_lock();                             // 等待所有任务执行完毕
    public:
        ThreadPool(std::size_t max_task_count,std::size_t inital_thread_count=0); // 构造函数
        ~ThreadPool();                                                               // 析构函数
        template <typename Func, typename... Args>
        auto submit(Func &&f, Args &&...args) -> std::future<decltype(f(args...))>; // 提交任务,实现对线程任务的异步提交
        void pause();                                                                  // 暂停线程池
        void resume();                                                                 // 恢复线程池
        void shutdown();                                                               // 立刻关闭线程池
        void shutdown_wait();                                                          // 等待任务执行完毕关闭线程池
        void terminate();                                                              // 终止线程池
        void wait();                                                                   // 等待所有任务执行完毕
        void add_thread(std::size_t count);                                            // 增加线程
        void remove_thread(std::size_t count);                                         // 删除线程
        void set_max_task_count(std::size_t count);
        std::size_t get_task_count();   // 获取任务数量
        std::size_t get_thread_count(); // 获取线程数量
    };

    template <typename Func, typename... Args>
    auto ThreadPool::submit(Func &&f, Args &&...args) -> std::future<decltype(f(args...))>
    {
        std::shared_lock<std::shared_mutex> status_lock(status_mutex);
        switch (status.load())
        {
            case status_t::TERMINATED:
            throw std::runtime_error("ThreadPool is terminated");
            case status_t::TERMINATING:
            throw std::runtime_error("ThreadPool is terminating");
            case status_t::PAUSED:
            throw std::runtime_error("ThreadPool is paused");
            case status_t::SHUTDOWN:
            throw std::runtime_error("ThreadPool is shutdown");
            case status_t::RUNNING:
            break;
        }
        if(max_task_count > 0&&max_task_count.load()==get_task_count())
            throw std::runtime_error("ThreadPool is full");
        using return_type=decltype(f(args...));
        auto task=std::make_shared<std::packaged_task<return_type()>>(std::bind(std::forward<Func>(f), std::forward<Args>(args)...));
        std::unique_lock<std::shared_mutex> lock2(task_queue_mutex);
        task_queue.emplace([task](){ (*task)(); }); 
        lock2.unlock();
        status_lock.unlock();
        task_queue_cv.notify_one();  //唤醒一个线程来执行当前任务
        std::future<return_type> res=task->get_future();
        return res;
    }
  ```

- 工作线程类
  工作线程类主要负责从任务队列中取出任务并执行，同时处理根据当前自身线程的状态变化动态调整线程池的状态。
  
  - 有关工作线程状态的枚举类
  - 线程状态的原子变量
  - 控制线程阻塞/唤醒的condition_variable
  相关定义如下：

  ```cpp
    class ThreadPool::worker_thread
    {
        private:
            enum class status_t:int8_t{
                TERMINATED=-1,
                TERMINATING=0,
                PAUSE=1,
                RUNNING=2,
                BLOCKED=3
            };   // 1- 线程已终止 0- 线程正在终止 1- 线程已暂停 2- 线程正在运行 3- 线程已阻塞,等待任务中
            ThreadPool *pool; //指向线程池
            std::atomic<status_t> status; //线程状态
            std::shared_mutex status_mutex; //线程状态互斥锁
            std::binary_semaphore sem; //信号量,要来控制线程的阻塞和唤醒
            std::thread thread; //工作线程
            //禁用拷贝构造与移动构造以及相关复赋值
            worker_thread(const worker_thread &) = delete;
            worker_thread(worker_thread &&) = delete;
            worker_thread &operator=(const worker_thread &) = delete;
            worker_thread &operator=(worker_thread &&) = delete;

            void resume_with_status_lock();
            status_t terminate_with_status_lock();
            void pause_with_status_lock();
            
        public:
            worker_thread(ThreadPool *pool);
            ~worker_thread();
            void pause();
            void resume();
            status_t terminate();
    };
  ```

## 线程池的工作机理剖析

对于该项目的线程池而言,它主要存在下面三个机制：

1. 基于状态机的状态转换
2. 任务的异步执行机制
3. 工作线程的任务执行

- 基于状态机的状态转换

在线程池中主要有两种状态转换,首先是线程池的状态转换,这里我们通过枚举定义了线程池的不同状态,让线程池在这些状态中不断切换,枚举类的定义如下:

```cpp
enum class status_t : std::int8_t
        {
            TERMINATED = -1,
            TERMINATING = 0,
            RUNNING = 1,
            PAUSED = 2,
            SHUTDOWN = 3
        }; 
// 线程池的状态: 已终止:-1,正在终止:0,正在运行:1,已暂停:2,等待线程池中任务完成,但是不接收新任务:3

std::atomic<status_t> status;//线程池的状态,这里使用原子变量来保证操作的原子性，实现了线程安全
```

而线程池的状态机模型如下:
![Alt text](../../../../mythreadPoll/%E7%8A%B6%E6%80%81%E6%9C%BA%E6%A8%A1%E5%9E%8B.png)

具体的状态转化代码可以参考:

```cpp
    void ThreadPool::pause_with_status_lock()
    {
        switch (status.load())
        {
        case status_t::TERMINATED:  // 线程池已经终止
        case status_t::TERMINATING: // 线程池正在终止
        case status_t::PAUSED:      // 线程池已经暂停
        case status_t::SHUTDOWN:    // 线程池已经关闭
            return;
        case status_t::RUNNING:
            status.store(status_t::PAUSED);
            break;
        default:
            throw std::runtime_error("unknown status");
        }
        std::unique_lock<std::shared_mutex> lock(worker_lists_mutex);
        for (auto &worker : worker_lists)
        {
            worker.pause();
        }
    }
```

工作线程的状态转换就简单了,只有四种状态:

```cpp
 enum class status_t:int8_t{
                TERMINATED=-1,
                TERMINATING=0,
                PAUSE=1,
                RUNNING=2,
                BLOCKED=3
            };   // 1- 线程已终止 0- 线程正在终止 1- 线程已暂停 2- 线程正在运行 3- 线程已阻塞,等待任务中
```

它的状态转换基本上和线程池息息相关,可以参考线程池的状态转换，这里不做过多说明。

- 任务的异步执行

在讲线程的异步执行之前,先来了解一下什么是线程的异步执行:

```text
在计算机科学中，“异步”通常指的是一个操作在发起之后，不需要等待其完成就可以继续执行后续的操作。异步编程模型允许程序在等待某个耗时操作完成的同时继续执行其他任务，从而提高整体效率和响应速度。
```

在c++11中为了实现异步执行,引入了std::async,它可以在一个单独的线程中执行一个函数,并返回一个std::future对象,通过这个对象可以获取函数的返回值。这里不做赘述，后面我们单独出一篇文章来探讨这个问题，这里我们只需要知道它的作用就可以了。这里我们来看一下线程池的任务提交函数`submit`:

```cpp
    template <typename Func, typename... Args>
    auto ThreadPool::submit(Func &&f, Args &&...args) -> std::future<decltype(f(args...))>
    {
        std::shared_lock<std::shared_mutex> status_lock(status_mutex);
        switch (status.load())
        {
            case status_t::TERMINATED:
            throw std::runtime_error("ThreadPool is terminated");
            case status_t::TERMINATING:
            throw std::runtime_error("ThreadPool is terminating");
            case status_t::PAUSED:
            throw std::runtime_error("ThreadPool is paused");
            case status_t::SHUTDOWN:
            throw std::runtime_error("ThreadPool is shutdown");
            case status_t::RUNNING:
            break;
        }
        if(max_task_count > 0&&max_task_count.load()==get_task_count())
            throw std::runtime_error("ThreadPool is full");
        using return_type=decltype(f(args...));
        auto task=std::make_shared<std::packaged_task<return_type()>>(std::bind(std::forward<Func>(f), std::forward<Args>(args)...));
        std::unique_lock<std::shared_mutex> lock2(task_queue_mutex);
        task_queue.emplace([task](){ (*task)(); }); 
        lock2.unlock();
        status_lock.unlock();
        task_queue_cv.notify_one();  //唤醒一个线程来执行当前任务
        std::future<return_type> res=task->get_future();
        return res;
    }
```

它的实现思路如下:

1. 首先查看当前线程池的状态，如果不是RUNNING状态,抛出异常
2. 查看当前的任务队列是否已满，如果已满，则抛出异常
3. 将任务转换为std::packaged_task对象，并将其包装为std::function对象，以便在线程池中执行，这里使用了std::forward将参数传递给任务函数实现完美转发,通过std::function对象将任务函数包装std::function<void()>类型,以便在线程池中通过在工作线程中可以用统一的格式（直接用 () 进行调用）对任何形式的任务进行调用执行。
4. 将std::packaged_task对象添加到任务队列中，并返回一个std::future对象,该对象可以用于获取任务函数的返回值

- 工作线程的任务执行

在讲解工作线程的任务执行之前,我们先来看一下它的具体实现：

```cpp
ThreadPool::worker_thread::worker_thread(ThreadPool *pool):pool(pool),status(status_t::RUNNING),sem(0),thread(
    [this](){
        while (true)
        {
            // 实现线程状态的判断，决定是否由该线程执行任务
            std::unique_lock<std::shared_mutex> unique_lock_status(this->status_mutex);
            while (true)
            {
                if (!unique_lock_status.owns_lock()) // 当锁被释放时，重新获取锁
                {
                    unique_lock_status.lock();
                }
                bool break_flag = false; // 当线程为运行态的时候，跳出循环
                switch (this->status.load())
                {
                case status_t::TERMINATING:
                    this->status.store(status_t::TERMINATED);
                case status_t::TERMINATED:
                    return;
                case status_t::RUNNING:
                    break_flag = true;
                    break;
                case status_t::PAUSE: // PAUSE状态下需要其他的线程来唤醒该线程需要解锁避免出现死锁
                    unique_lock_status.unlock();
                    this->sem.acquire(); // 阻塞当前线程
                    break;
                case status_t::BLOCKED: // 不支持Blocked
                default:
                    unique_lock_status.unlock();
                    throw std::runtime_error("invalid status");
                }
                if (break_flag)
                {
                    unique_lock_status.unlock();
                    break;
                }
            }

            // 判断队列是否为空，如果为空则阻塞当前线程
            std::unique_lock<std::shared_mutex> unique_lock_task(this->pool->task_queue_mutex);
            while (this->pool->task_queue.empty())
            {
                while (true)
                {
                    if (!unique_lock_status.owns_lock())
                    {
                        unique_lock_status.lock();
                    }
                    bool break_flag = false;
                    switch (this->status.load())
                    {
                    case status_t::TERMINATING:
                        status.store(status_t::TERMINATED);
                    case status_t::TERMINATED:
                        return;
                    case status_t::PAUSE:
                        unique_lock_task.unlock();
                        unique_lock_status.unlock();
                        this->sem.acquire(); // 阻塞线程
                        break;
                    case status_t::RUNNING:
                        this->status.store(status_t::BLOCKED);
                    case status_t::BLOCKED:
                        break_flag = true;
                        break;
                    default:
                        unique_lock_status.unlock();
                        unique_lock_task.unlock();
                        throw std::runtime_error("invalid status");
                    }
                    if (break_flag) // 若为阻塞状态等待唤醒
                    {
                        unique_lock_status.unlock();
                        break;
                    }
                }
                this->pool->task_queue_cv.wait(unique_lock_task);
                while (true)
                {
                    if (!unique_lock_status.owns_lock())
                    {
                        unique_lock_status.lock();
                    }
                    bool break_flag = false;
                    switch (this->status.load())
                    {
                    case status_t::TERMINATING:
                        status.store(status_t::TERMINATED);
                    case status_t::TERMINATED:
                        return;
                    case status_t::PAUSE:
                        unique_lock_task.unlock();
                        unique_lock_status.unlock();
                        this->sem.acquire(); // 阻塞线程
                        break;
                    case status_t::BLOCKED:
                        this->status.store(status_t::RUNNING);
                    case status_t::RUNNING:
                        break_flag = true;
                        break;
                    default:
                        unique_lock_status.unlock();
                        throw std::runtime_error("invalid status");
                    }
                    if (break_flag) // 若为阻塞状态等待唤醒
                    {
                        unique_lock_status.unlock();
                        break;
                    }
                }
            }
            // 尝试取出任务并执行
            try
            {
                auto task = this->pool->task_queue.front();
                this->pool->task_queue.pop();
                if (this->pool->task_queue.empty())
                {
                    this->pool->task_queue_cv_empty.notify_all();
                }
                unique_lock_task.unlock();
                task();
            }
            catch (const std::exception &e)
            {
                std::cerr<<e.what()<<std::endl;
            }
        }
    }){}

```

工作线程工作的进行是定义在线程构造函数中,即线程开始工作后,会一直执行这个函数,直到线程被销毁,它的实现逻辑是:

- 确定工作线程状态,决定是否由该线程执行任务
- 查看工作队列是否为空,不为空则取出一个任务
- 执行任务
- 根据线程池状态变更，如接收到暂停、恢复、终止等指令，工作线程调整自身状态并执行相应操作

## 拓展:ini文件的读写

考虑到线程池后续想要对其进行拓展,例如加入epoll进行网络通信，所以我实现了一个简单的ini解析器，他主要有两部分:

- Value类:实现了`int,double,long,string`等基础类型与Value对象之间的互相转换,代码如下：

```cpp
class Value  // 封装一个Value对象，支持string、int、double、bool类型，可以实现多个类型转换为Value对象，并且可以转换为多个类型
    {
    private:
        std::string m_value; 
    public:
        //支持各个类型的构造函数
        Value();
        Value(const std::string& value);
        Value(const int value);
        Value(const double value);
        Value(const bool value);
        Value(const char* value);
        ~Value();

        //赋值操作,支持任意类型
        Value& operator=(const std::string& value);
        Value& operator=(const int value);
        Value& operator=(const double value);
        Value& operator=(const bool value);
        Value& operator=(const char* value);

        //对于值的判断
        bool operator==(const Value& other);
        bool operator!=(const Value& other);

        //这里实现可以将Value对象转换为任意类型
        operator int();
        operator double();
        operator bool();
        operator std::string();
    };

```

- IniFile:基于嵌套map来存储ini文件的各个分区与分区的键值对，实现如下:

```cpp
    typedef std::map<std::string, Value> Section;

    class IniFile
    {
        private:
            std::string m_filename;
            std::map<std::string, Section> m_sections;
            std::string trim(std::string s);  //去除字符串两边的空格 
        public:
            IniFile();
            IniFile(const std::string& filename);
            ~IniFile();

            bool load(const std::string& filename);  //加载ini文件
            bool save(const std::string& filename);  //保存ini文件
            void show(); //显示ini文件内容
            void clear(); //清空ini文件内容
            
            Value& get(const std::string& section,const std::string& key); //获取指定分区指定键的值
            void set(const std::string& section,const std::string& key,const Value& value); //设置指定分区指定键的值
            bool remove(const std::string& section,const std::string& key); //删除指定分区指定键的值
            
            bool has(const std::string& section,const std::string& key); //判断键值对是否存在
            bool has(const std::string& section);// 判断分区是否存在
            
            Section& operator[](const std::string& section); // 重载[]操作符,用来访问分区中的键值
            std::string str(); //将字符串转换为ini文件格式
    };
```

篇幅有限，就不粘详细代码了，大家也可以尝试自己实现以下，毕竟cpp就是一个不断遭轮子的过程。

## 结语

线程池的实现是一个复杂的过程，需要考虑多线程并发、任务调度、线程管理等多个方面。本文通过一个简单的线程池实现，介绍了线程池的基本原理和实现方法。希望本文对您有所帮助。

- [博客地址](https://blog.csdn.net/qq_73924465/article/details/143217134?spm=1001.2014.3001.5501)
- [教程地址]()
- [项目地址](https://github.com/1708737115/mythreadPoll)
