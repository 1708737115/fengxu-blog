# linux的信号

## 信号的概念

在Linux中，信号是一种用于进程间通信和处理异步事件的机制，用于进程之间相互传递消息和通知进程发生了事件，但是，它不能给进程传递任何数据。

信号产生的原因有很多种，在`shell`中，我们可以使用`kill`和`killall`来发送信号

~~~shell
kill -信号的类型  进程编号
killall -信号的类型 进程名
~~~

## 信号的类型

常见信号类型：

- `SIGINT`:终止进程（键盘快捷键 `ctrl+c`）
- `SIGKILL`: 采用kill -9 进程编号，强制杀死程序

......

## 信号的处理

进程对信号的处理方法一般有三种：

- 对该信号进行默认处理，一般是终止该进程
- 设置中断的处理函数，受到该型号的函数进行处理
- 忽略该信号，不做如何处理

`signal()`函数可以设置程序对信号的处理方式

函数的声明：

~~~cpp
sighandler_t signal(int signum,sighandler_t handler)
~~~

**注释**：

参数`signum`表示信号的编号

参数`handler`表示信号的处理方式，有三种情况：

1. `SIG_DFL`:恢复参数`signum`所指信号的处理方法为默认值
2. 一个自定义的处理信号的函数，信号的编号为这个自定义函数的参数
3. `SIG_IGN`:忽略`signum`所指的信号

示例代码：

~~~cpp
#include <iostream>
#include <unistd.h>
#include <signal.h>

using namespace std;

void func(int signum)
{
    cout<<"收到了信号"<<signum<<endl;
    signal(1,SIG_DFL);//将函数的处理方式由自定义函数改为了默认方式处理
}

int main(int argc,char *argv[],char *envp[])
{
    signal(1,func);
    signal(15,func);
    signal(2,SIG_IGN);//忽略信号2
    while(1)
    {
        cout<<argc<<endl;
        sleep(1);
    }
    return 0;
}
~~~

## 信号的作用

​ 服务程序运行在后台，如果想终止它，一般不会直接杀死它，以防止出现意外。

​ 我们·一般会选择向进程去发送一个信号，当程序收到这个信号的时候能够调用函数，并通过函数中英语善后的代码，原计划的退出。

​ 我们也可以向其发送`0`的信号来确保程序是否存活

示例代码：

~~~cpp
#include <iostream>
#include <signal.h>

using namespace std;

void Exit(int signum)
{
    cout<<"收到了"<<signum<<"信号"<<endl;
    cout<<"开始释放资源并退出"<<endl;
    //释放资源的代码
    cout<<"退出程序"<<endl;
    exit(0);
}
int main()
{
    for(int i=1;i<=64;i++) 
    {
        signal(i,SIG_IGN);
    }
    signal(2,Exit);
    signal(15,Exit);
    while(1)
    {
        cout<<"fengxu\n";
    }
}
~~~
