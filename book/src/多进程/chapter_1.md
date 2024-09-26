# 进程终止

## 进程的终止

 在`main()`函数中，return的返回值就是终止状态，如果没有`return`语句或者调用`exit()`,那么该进程终止状态为0

我们可以通过

~~~shell
echo $?
~~~

来查看线程终止的状态

正常终止进程函数有三个：

~~~cpp
void exit(int status);
void _exit(int status);
void _Exit(int status);
~~~

`status`也是进程终止的状态

**注意**：进程如果被异常终止，终止状态也为非0

## 资源释放

`return`表示函数返回，会调用局部对象的析构函数，`main()`函数中的`return`还会调用全局对象的析构函数

`exit()`表示进程终止，它不会调用局部对象的析构函数，只会调用全局变量的析构函数

**注意：**`exit()`会执行清理工作再退出，但是`_EXIT()`和`——exit（）`不会执行清理工作

## 进程的终止函数

进程可以利用`atexit`函数来登记终止函数(最多32个)，这些函数将由`exit()`自动调用。

**注意：**运行登记函数的顺序与登记函数顺序相反

示例代码：

~~~cpp
#include <iostream>
#include <stdlib.h>

using namespace std;

void fuc1()
{
    cout<<"调用了fuc1()"<<endl;
}

void fuc2()
{
    cout<<"调用了fuc2()"<<endl;
}

int main()
{
    atexit(fuc1);
    atexit(fuc2);
    exit(0);
}
~~~

输出：

![image-20231221164902241](C:\Users\fengxu\AppData\Roaming\Typora\typora-user-images\image-20231221164902241.png)
