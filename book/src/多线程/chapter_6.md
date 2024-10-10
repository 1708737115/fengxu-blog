# RAII与智能指针

## 前言

在前面的章节中，我们介绍了多线程编程中的基本概念，包括线程的创建、同步、互斥等。在锁的介绍那篇文章中我们介绍了锁的包装器`unique_lock`与`lock_guard`,它们都是基于RAII（Resource Acquisition Is Initialization）原则的智能指针。RAII是一种编程范式，它使用对象的生命周期来管理资源的分配和释放。在多线程编程中，RAII可以有效地避免资源泄露和死锁等问题。而这篇文章我们将从RAII开始，探究c++11后内存管理的一些新特性.

## RAII

RAII（Resource Acquisition Is Initialization）是一种编程原则，它要求在创建对象时自动分配资源，并在对象销毁时自动释放资源。RAII通过将资源分配和释放封装在构造函数和析构函数中，实现了资源的自动管理，从而避免了手动管理资源带来的errors和bugs。RAII的核心思想是：将资源的分配和释放与对象的生命周期绑定在一起，当对象被创建时，资源被分配；当对象被销毁时，资源被释放。这种机制可以有效地避免资源泄露和死锁等问题。

在RAII一般是不会轻易使用浅拷贝的，因为浅拷贝会带来资源泄露的问题。例如，如果两个对象共享同一个资源，并且其中一个对象被销毁，那么另一个对象将无法访问该资源，从而导致资源泄露。因此，在RAII中，通常会使用深拷贝来避免资源泄露的问题。我们用下面的一个简单例子来说明一下:

```cpp
#include <iostream>

using namespace std;

class RAII
{
    private:
        int* ptr;
    public:
    RAII():ptr(new int(10)){}
    ~RAII(){delete ptr;}
};

int main()
{
    RAII r1;
    RAII r2(r1); // 拷贝构造函数
    return 0;
}
```

上面的代码中我们定义了一个简单的RAII类,并且示例化了两个对象r1和r2,但是当我们运行这段代码时,会出现一个错误,如果说在运行中我们不小心释放了r1的资源,由于r1与r2张建是浅拷贝的关系，它们使用的本质上是同一块内存,我们释放了r1的内存，r2的内存也就被释放了，这会导致程序崩溃。为了解决这个问题，我们可以下面的方法:

- 使用深拷贝来避免资源泄露的问题,但是频繁的复制会造成不必要的资源浪费。
- 使用引用计数来管理资源的共享。

而如何应用计数来管理资源的共享呢？这就需要我们引入智能指针了。

## 智能指针

智能指针是一种自动管理内存的机制，它通过引用计数来管理资源的共享。智能指针在对象被创建时自动分配资源，并在对象被销毁时自动释放资源。智能指针通过引用计数来跟踪对象的引用次数，当引用次数为0时，智能指针会自动释放资源。智能指针可以避免手动管理内存带来的errors和bugs，并且可以有效地避免资源泄露和死锁等问题。常见的智能指针主要有下面四种:

- `std::auto_ptr`：自动指针，在C++11中已弃用。
- `std::shared_ptr`：共享指针，允许多个智能指针共享同一个对象，当最后一个智能指针被销毁时，对象才会被销毁。
- `std::unique_ptr`：独占指针，独占一个对象，当智能指针被销毁时，对象才会被销毁。
- `std::weak_ptr`：弱指针，不拥有对象的所有权，可以用来解决循环引用的问题。

在对智能指针 讲解之前微妙先看几个智能指针中比较主要的api:

- reset:释放当前管理的资源，并将指针置为空。
- release:释放当前管理的资源，但是不将指针置为空。
- swap:交换两个智能指针管理的资源。
- get:返回当前管理的资源的指针。
- use_count:返回当前管理的资源的引用计数。
- expire:返回当前管理的资源是否已经被释放。

下面我们依次来看一下这几个智能指针的使用:

- **std::auto_ptr**:自动指针，在C++11中已弃用。
`auto_ptr`作为智能指针弊端是最明显的,它采用的所有权转移方法是隐式转移所有权，我们可以来看一下它的相关实现:

```cpp
 class auto_ptr{
     
     auto_ptr& operator=(auto_ptr tmp) noexcept {
         // copy and swap技术，这里不展开了
         // 注意当拷贝构造函数构造tmp时，会发生所有权的转移
         tmp.swap(*this);
         return *this;
     }
     
     // 拷贝构造,被复制时释放原来指针的所有权,交给复制方
     auto_ptr(auto_ptr &other) noexcept {
         ptr_ = other.release();
     }
     
     // 原来的指针释放所有权
     T *release() noexcept {
         T *ptr = ptr_;
         ptr_ = nullptr;
         return ptr;
     }
 };
```

在新的指针获取了旧指针资源后,旧指针就被置为空指针了，这时候我们再去访问旧指针就会造成未定义行为,所以`auto_ptr`在C++11中就被弃用了。

- **std::shared_ptr**:共享指针，允许多个智能指针共享同一个对象，当最后一个智能指针被销毁时，对象才会被销毁。

在介绍`shared_ptr`之前，我写一个简易版的`shared_ptr`，来帮助大家理解`shared_ptr`的实现原理:

```cpp
#include <iostream>
#include <memory>
#include <utility>
#include <atomic>

using namespace std;

template<typename T>
class Shared_ptr {
private:
    using element_type = T;
    element_type* ptr; ////智能指针管理的指针/资源
    atomic<int>* count;  //引用计数

public:
    Shared_ptr() : ptr(nullptr), count(nullptr) {} //默认构造函数
    explicit Shared_ptr(T* ptr) : ptr(ptr), count(new atomic<int>(1)) {} //避免智能指针被用于隐式类型转换或复制初始化

    Shared_ptr(const Shared_ptr& rhs) : ptr(rhs.ptr), count(rhs.count) {  ////拷贝构造函数
        if (*count) ++(*count);
    }

    Shared_ptr& operator=(const Shared_ptr<T>& rhs) {  //拷贝赋值运算符
        if (this != &rhs) {
            Shared_ptr tmp(rhs);
            swap(tmp);
        }
        return *this;
    }

    Shared_ptr(Shared_ptr&& rhs) noexcept : ptr(rhs.ptr), count(rhs.count) { //移动构造函数
        rhs.ptr = nullptr;
        rhs.count = nullptr;
    }

    Shared_ptr& operator=(Shared_ptr<T>&& rhs) noexcept {  //移动赋值运算符
        if (this != &rhs) {
            Shared_ptr tmp(std::move(rhs));
            swap(tmp);
        }
        return *this;
    }

    ~Shared_ptr() {  //析构函数
        if (ptr != nullptr) {
            --(*count);
            if (*count == 0) {
                delete ptr;
                delete count;
            }
        }
    }

    void swap(Shared_ptr& rhs) { //交换两个智能指针
        std::swap(ptr, rhs.ptr);
        std::swap(count, rhs.count);
    }

    void reset(T* ptr = nullptr) {  //将当前Shared_ptr 置为空，并释放其所管理的对象。
        Shared_ptr tmp(ptr);
        swap(tmp);
    }

    element_type* get() const noexcept {  //返回原始指针（即 T*）
        return ptr;
    }

    int use_count() const noexcept {  //返回当前有多少个Shared_ptr 对象指向同一个对象
        return *count;
    }

    bool unique() const noexcept { //返回当前Shared_ptr是否是唯一的
        return *count == 1;
    }

    element_type& operator*() const noexcept { //
        return *ptr;
    }

    element_type* operator->() const noexcept { //返回指针指向的对象
        return ptr;
    }

    explicit operator bool() const noexcept { //返回当前指针是否为空
        return ptr != nullptr;
    }
};

template<typename T, typename... Args>
Shared_ptr<T> make_Shared(Args&&... args) {
    T* ptr = new T(std::forward<Args>(args)...);
    atomic<int>* count = new atomic<int>(1);
    return Shared_ptr<T>(ptr);
}
```

我们不难看到在Shared_ptr中,主要有两个变量,一个是原始指针ptr,另一个是引用计数count。ptr指向实际管理的对象,而count则记录有多少个Shared_ptr指向同一个对象。当创建一个Shared_ptr对象时,引用计数会加1,当Shared_ptr对象被销毁时,引用计数会减1,当引用计数为0时,则说明没有Shared_ptr指向该对象,此时就会释放该对象。理解了这个，在结合一下上面的代码基本就理解了shared_ptr的原理了。

- **std::unique_ptr**:独占式智能指针,它不允许其他的智能指针共享其内部的指针,不允许通过复制构造函数或赋值操作符复制。它通过在析构函数中删除指针来实现自动释放内存的功能。

还是和原先一样我们来看一个手写的简单unique_ptr:

```cpp
#ifndef UNIQUE_PTR_H
#define UNIQUE_PTR_H

#include <iostream>
#include <memory>
#include <type_traits>

using namespace std;

template<typename T, typename Deleter = std::default_delete<T>>
class Unique_Ptr {
private:
    T* ptr;     //指向被管理的对象的指针
    Deleter deleter; //自定义删除器

public:
    Unique_Ptr() : ptr(nullptr), deleter() {} //默认构造函数

    explicit Unique_Ptr(T* ptr, Deleter d = Deleter()) : ptr(ptr), deleter(std::move(d)) {} //普通构造函数

    Unique_Ptr(const Unique_Ptr<T, Deleter>&) = delete; //禁止拷贝构造函数
    Unique_Ptr& operator=(const Unique_Ptr<T, Deleter>&) = delete; //禁止赋值操作符

    Unique_Ptr(Unique_Ptr<T, Deleter>&& other) : ptr(other.ptr), deleter(std::move(other.deleter)) { //移动构造函数
        other.ptr = nullptr;
    }

    Unique_Ptr& operator=(Unique_Ptr<T, Deleter>&& other) { //移动赋值操作符
        if (this != &other) {
            deleter(this->ptr);
            ptr = other.ptr;
            deleter = std::move(other.deleter);
            other.ptr = nullptr;
        }
        return *this;
    }

    void swap(Unique_Ptr<T, Deleter>& other) noexcept { //交换指针
        std::swap(ptr, other.ptr);
        std::swap(deleter, other.deleter);
    }

    T* get() const { //返回指向被管理对象的指针
        return ptr;
    }

    Deleter get_deleter() const { //返回删除器
        return deleter;
    }

    T* release() { //释放被管理对象的所有权
        T* temp = ptr;
        ptr = nullptr;
        return temp;
    }

    void reset(T* new_ptr = nullptr) { //释放被管理对象的所有权并指向新的对象
        if (ptr != nullptr) {
            deleter(ptr);
        }
        ptr = new_ptr;
    }

    T& operator*() const { //解引用操作符
        return *ptr;
    }

    T* operator->() const { //箭头操作符
        return ptr;
    }

    explicit operator bool() const noexcept { //显式转换为布尔值
        return ptr != nullptr;
    }

    ~Unique_Ptr() { //析构函数
        if (ptr != nullptr) {
            deleter(ptr);
        }
    }
};

#endif // UNIQUE_PTR_H
```

`unique_ptr`在cpp11中被提出，用来替换我们上面所说弊端很大的`auto_ptr`,我们可以看到在我们的实现中它并不支持拷贝构造函数和赋值操作符,这样我们就避免了`auto_ptr`中出现的隐藏bug,并且它还支持自定义删除器deleter,这样我们就可以在释放内存的时候执行一些额外的操作,比如释放资源等，而翻译删除器的方式也很简单:

```cpp
struct CustomDeleter 、

{
    void operator()(int* p) const 
    {
        std::cout << "Deleting with custom deleter" << std::endl;
        delete p;
    }
};

Unique_Ptr<int, CustomDeleter> uptr_custom(new int(30), CustomDeleter())
;
```

自定义删除器的方式不仅仅是`unique_ptr`，`shared_ptr`中也可以实现自定义删除器来用来关闭一些资源，比如文件，数据库连接等。

`unique_ptr`与`shared_ptr`最大的区别就在于`unique_ptr`拥有对象的所有权，而`shared_ptr`则共享对象的所有权，`unique_ptr`在释放对象的时候会直接释放内存，而`shared_ptr`则会通过引用计数来管理对象的生命周期，当引用计数为0的时候才会释放内存。

- **std::weak_ptr**

`std::weak_ptr`是一种不控制所指向对象生存期的智能指针，它指向一个`std::shared_ptr`管理的对象。`std::weak_ptr`不会改变引用计数，它的构造不会增加引用计数，析构也不会减少引用计数。`std::weak_ptr`可以用来解决`std::shared_ptr`循环引用的问题。

weak_ptr也有主要有下面几个api:

```cpp
std::weak_ptr<T> wptr; //创建一个空的weak_ptr
std::weak_ptr<T> wptr(sp); //使用shared_ptr创建一个weak_ptr
wptr.reset(); //将wptr置空
wptr.expired(); //检查所管理的对象是否已经释放，返回true表示已经释放
，返回false表示未释放
wptr.lock(); //如果weak_ptr管理的对象还存在，则返回一个shared_ptr，否则返回一个空的shared_ptr
```

这里我们主要介绍`lock`和`expire`函数以及`std::shared_ptr`和`std::weak_ptr`的循环引用问题。

首先是`lock`和`expire`函数，`lock`函数会返回一个`std::shared_ptr`，如果`weak_ptr`管理的对象还存在，则返回一个`std::shared_ptr`，否则返回一个空的`std::shared_ptr`，`expire`函数则会检查`weak_ptr`管理的对象是否还存在(就是我们查看指向的shared_ptr计数器的值是否大于0,大于就是存在，小于就是不存在)，如果存在则返回`false`，否则返回`true`。

然后就是`std::shared_ptr`和`std::weak_ptr`的循环引用问题，循环引用问题会导致内存泄漏，因为`std::shared_ptr`和`std::weak_ptr`都会增加引用计数，当引用计数为0的时候才会释放内存，如果两个`std::shared_ptr`互相引用对方，那么引用计数永远不会为0，就会导致内存泄漏。

我们来看下面的一个例子:

```cpp
#include <iostream>
#include <memory>

using namespace std;

class Parent;
class Child; 

typedef shared_ptr<Parent> parent_ptr;
typedef shared_ptr<Child> child_ptr; 

class Parent
{
public:
       ~Parent() { 
              cout << "~Parent()" << endl; 
       }
public:
       child_ptr children;
};

class Child
{
public:
       ~Child() { 
              cout << "~Child()" << endl; 
       }
public:
       parent_ptr parent;
};

int main()
{
  parent_ptr father(new Parent);
  child_ptr son(new Child);

  // 父子互相引用
  father->children = son;
  son->parent = father;

  cout << father.use_count() << endl;  // 引用计数为2
  cout << son.use_count() << endl;     // 引用计数为2

  return 0;
}
```

上面的代码中，`Parent`和`Child`互相引用对方，导致引用计数永远不会为0，从而导致了内存泄漏。我们可以使用`std::weak_ptr`来解决这个问题，`std::weak_ptr`不会增加引用计数，所以不会导致循环引用问题。

```cpp
#include <iostream>
#include <memory>

using namespace std;

class Parent;
class Child; 

typedef shared_ptr<Parent> parent_ptr;
typedef shared_ptr<Child> child_ptr; 

class Parent
{
public:
       ~Parent() { 
              cout << "~Parent()" << endl; 
       }
public:
       weak_ptr<Child> children;
};

class Child
{
public:
       ~Child() { 
              cout << "~Child()" << endl; 
       }
public:
       weak_ptr<Parent> parent;
};

int main()
{
  parent_ptr father(new Parent);
  child_ptr son(new Child);

  // 父子互相引用
  father->children = son;
  son->parent = father;

  cout << father.use_count() << endl;
  cout << son.use_count() << endl;     

  return 0;
}
```

在上面的代码中，我们使用了`std::weak_ptr`来代替`std::shared_ptr`，这样就不会导致循环引用问题。当`Parent`和`Child`对象被销毁时，它们的析构函数会被调用，输出`~Parent()`和`~Child()`，表示对象被正确地销毁了。
