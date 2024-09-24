# 初窥多线程(三) cpp中的线程类

## 前言

在cpp11之前其实一直也没有对并发编程有什么语言级别上的支持,而在cpp11中加入了线程以及提供了对线程的封装类，一方面降低了并发编程的难度,另一方面也加强了了多线程程序的可移植性。

## 线程类(std::thread)

### 前言

在cpp11中为我们提供了`std::thread`作为线程类,基于这个类我们可以快速的创建一个新的线程，接下来我们来看一下它的一些常见api:

### 线程类的构造函数

在我们进行多线程程序的编写时,常见的构造函数主要有以下几种:

```cpp
thread() noexpect;  //普通的构造函数
thread(thread&& other); //移动构造函数
template<class Function,Args... args>
explict thread(Fuction &&f,Args&& ... args);  //①
thread(const thread&) =delete; //禁止拷贝构造函数
```

上面就是主要会使用的多线程构造函数,这里我们主要讲解一下第三个构造函数，其他三个比较好懂，而第三个涉及的cpp11新特性比较多，这里我们着重讲一下:

- 首先是`template<class Function,Args... args>`
 	- `class Function`表示的是你想要在线程中执行的函数类型或者是可调用对象的类型，比如普通函数、lambda表达式、函数对象等.
 	- `Args... args`是一个变长模板参数包，表示Function所接受的参数类型序列。...表示可以有任意数量的参数，每个Args代表一个参数类型

- 然后是`explict thread(Fuction &&f,Args&& ... args)`
 
 	- `explict` ：`explict`关键字用于防止隐式类型转换，确保这个构造函数只能被显式调用。这意味着你不能在不需要明确指定类型的情况下意外地创建一个std::thread对象，例如通过自动类型推导或者一些隐式转换操作。
 	- `thread(Fuction &&f,Args&& ... args)`:这里函数形参使用右值引用使得左值引用与右值引用都可以绑定在上面，实现了参数的完美转发机制，详细的请参考：
[cpp随笔——浅谈右值引用，移动语义与完美转发](https://blog.csdn.net/qq_73924465/article/details/139886619?spm=1001.2014.3001.5501)

### 线程类的公有成员函数

- **get_id()**
应用程序启动之后默认只有一个线程，这个线程一般称之为主线程或父线程，通过线程类创建出的线程一般称之为子线程，每个被创建出的线程实例都对应一个线程ID，这个ID是唯一的，可以通过这个ID来区分和识别各个已经存在的线程实例，这个获取线程ID的函数叫做get_id()，示例代码如下：

```cpp
#include <iostream>
#include <thread>
#include <chrono>
using namespace std;

void func(int num, string str)
{
    for (int i = 0; i < 10; ++i)
    {
        cout << "子线程: i = " << i << "num: "
            << num << ", str: " << str << endl;
    }
}

void func1()
{
    for (int i = 0; i < 10; ++i)
    {
        cout << "子线程: i = " << i << endl;
    }
}

int main()
{
    cout << "主线程的线程ID: " << this_thread::get_id() << endl;
    thread t(func, 520, "i love you");
    thread t1(func1);
    cout << "线程t 的线程ID: " << t.get_id() << endl;
    cout << "线程t1的线程ID: " << t1.get_id() << endl;
}
```

这里我们用`thread()函数来创建`类一个线程，`func`是线程的工作函数,然后在`thread`函数中传递工作函数所需的参数。

上述代码存在一个小问题,主线程会在子线程结束前退出，而主线程退出会导致子线程提前退出导致无法得到我们所需的效果。

- **join()**
这个join的使用基本与c语言中的`pthread_join`函数功能相像，我们可以基于这个来修复一下上面代码的bug：

```cpp
#include <iostream>
#include <thread>
#include <chrono>
using namespace std;

void func(int num, string str)
{
    for (int i = 0; i < 10; ++i)
    {
        cout << "子线程: i = " << i << "num: "
            << num << ", str: " << str << endl;
    }
}

void func1()
{
    for (int i = 0; i < 10; ++i)
    {
        cout << "子线程: i = " << i << endl;
    }
}

int main()
{
    cout << "主线程的线程ID: " << this_thread::get_id() << endl;
    thread t(func, 520, "i love you");
    thread t1(func1);
    cout << "线程t 的线程ID: " << t.get_id() << endl;
    cout << "线程t1的线程ID: " << t1.get_id() << endl;
    t.join();
    t1.join();
    return 0;
}
```

这样就可以得到我们想要的结果了。

- **detach()**
std::thread 类的 detach() 成员函数用于将线程与创建它的线程（即主线程）分离。调用 detach() 后，被分离的子线程将成为一个独立的执行单元，它有自己的生命周期，不再受主线程的控制。

```cpp
#include <iostream>
#include <thread>
#include <chrono>
using namespace std;

void func(int num)
{
    for (int i = 0; i < 10; ++i)
    {
        cout << "子线程: i = " << i << "num: "
            << num  << endl;
    }
}

void func1()
{
    for (int i = 0; i < 10; ++i)
    {
        cout << "i = " << i << endl;
    }
}

int main()
{
    cout << "主线程的线程ID: " << this_thread::get_id() << endl;
    thread t(func, 520);
    thread t1(func1);
    cout << "线程t 的线程ID: " << t.get_id() << endl;
    cout << "线程t1的线程ID: " << t1.get_id() << endl;
    t.detach();
    t1.join();
    return 0;
}
```

**注意**:

 1. 线程在被`detach`后不能再进行`join`操作
 2. 线程对象在销毁前必须选择是`detach`或`join`,否则就会出现异常。

- **joinable()**
`joinable()`函数用于判断主线程和子线程是否处理关联（连接）状态，一般情况下，二者之间的关系处于关联状态，该函数返回一个布尔类型：
`返回值为true`：主线程和子线程之间有关联（连接）关系
`返回值为false`：主线程和子线程之间没有关联（连接）关系

- **operator=**
线程的资源是不可复制的,所以我们无法通过重载赋值符获得两个一模一样的线程对象，我们来看一下它的声明：

```cpp
thread& opreator=(thread&& t)noexcept;
thread& opreator=(thread& t) =delete;
```

我们可以看到它只进行资源所有权的转移而不进行对对象资源的复制。

### 静态函数

thread线程类还提供了一个静态方法，用于获取当前计算机的CPU核心数，根据这个结果在程序中创建出数量相等的线程，每个线程独自占有一个CPU核心，这些线程就不用分时复用CPU时间片，此时程序的并发效率是最高的。

```cpp
int thread::hardware_concurrency();
```

示例代码:

```cpp
#include <iostream>
#include <thread>
#include <chrono>
using namespace std;

int main()
{
    int num = thread::hardware_concurrency();
    cout << num;
    return 0;
}
```
