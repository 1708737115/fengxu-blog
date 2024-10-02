# 进程通信——内存映射

## 什么是内存映射

内存映射是一种将文件内容映射到进程地址空间的技术，使得进程可以直接访问文件内容，而不需要通过系统调用进行读写操作。内存映射可以提高文件访问的效率，并且可以实现进程间的通信。

## 内存映射的原理

我们在进程中创建一个内存映射区，但由于这个内存映射区是创建在进程的地址空间中，所以外界的进程空间无法直接访问其他进程的内存映射区。但是，我们可以通过系统调用`mmap`将文件内容映射到内存映射区，这样进程就可以直接访问文件内容了，像下面这样:

![Alt text](image-7.png)
如上图那样,磁盘文件数据不仅可以完全加载到进程的内存映射区,还可以部分加载到进程的内存映射区，当进程A中的内存映射区数据被修改了，数据会被自动同步到磁盘文件，同时和磁盘文件建立映射关系的其他进程内存映射区中的数据也会和磁盘文件进行数据的实时同步,基于这样的同步机制我们可以实现进程间的通信。

## 内存映射的创建

内存映射的创建需要使用`mmap`函数，该函数的原型如下:

```c
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
```

参数说明:

- `addr`: 指定映射区的起始地址，如果为`NULL`，则由系统自动选择一个地址。
- `length`: 指定映射区的长度。
- `prot`: 指定映射区的保护方式，可以是以下值的组合:
  - `PROT_READ`: 可读。如果映射区被修改，则修改的内容会被同步到文件。
  - `PROT_WRITE`: 可写。如果映射区被修改，则修改的内容会被同步到文件。
  - `PROT_EXEC`: 可执行。
  - `PROT_NONE`: 无权限。
- `flags`: 指定映射区的标志，可以是以下值的组合:
  - `MAP_SHARED`: 共享映射区。多个进程可以共享同一个映射区。
  - `MAP_PRIVATE`: 私有映射区。每个进程都有自己的映射区，修改不会影响其他进程。
  - `MAP_ANONYMOUS`: 匿名映射区。不与文件关联，可以用于进程间的通信。
- `fd`: 指定要映射的文件的文件描述符。
- `offset`: 指定映射区的起始偏移量。

返回值:

- 成功返回映射区的起始地址。
- 失败返回`MAP_FAILED`。

由于参数较多,一般我们可以像下面这样创建:

- 第一个参数 addr 指定为 NULL 即可
- 第二个参数 length 必须要 > 0
- 第三个参数 prot，进程间通信需要对内存映射区有读写权限，因此需要指定为：PROT_READ | PROT_WRITE
- 第四个参数 flags，如果要进行进程间通信, 需要指定 MAP_SHARED
- 第五个参数 fd，打开的文件必须大于0，进程间通信需要文件操作权限和映射区操作权限相同
  - 内存映射区创建成功之后, 关闭这个文件描述符不会影响进程间通信
- 第六个参数 offset，不偏移指定为0，如果偏移必须是4k的整数倍

## 内存映射的释放

既然我们创建了内存映射区，那么在不需要的时候就需要释放它，释放内存映射区需要使用`munmap`函数，该函数的原型如下:

```c
int munmap(void *addr, size_t length);
```

参数说明:

- `addr`: 指定要释放的映射区的起始地址。
- `length`: 指定要释放的映射区的长度。

返回值:

- 成功返回0。
- 失败返回-1。

## 基于内存映射区实现进程间通信

### 内存映射区与管道通信的区别

- 管道通信是使用文件描述符进行通信，而内存映射区通信是直接使用内存地址进行通信。这导致了管道通信是阻塞的,而内存映射区通信是非阻塞的。

- 管道通信的数据需要经过内核缓冲区，而内存映射区通信的数据直接在用户空间和内核空间之间传递，不需要经过内核缓冲区。这导致了内存映射区通信的速度更快。

## 基于内存映射实现的进程通信

### 有血缘关系的进程通信

```cpp
#include <iostream>
#include <sys/types.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <cstring>
#include <unistd.h>
#include <wait.h>
#include <fcntl.h>

using namespace std;

int main()
{
    int fd = open("test.txt", O_RDWR | O_CREAT, 0666);
    if (fd == -1)
    {
        perror("open");
        return -1;
    }
    cout<<111<<endl;
    void* ptr = mmap(NULL, 4000, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    if (ptr == MAP_FAILED)
    {
        perror("mmap");
        return -1;
    }
    int pid = fork();
    if (pid > 0) // 父进程
    {
        const char* str = "test";
        cout<<222<<endl;
        // 确保字符串正确终止
        memcpy(ptr, str, strlen(str) + 1);
        // 等待一段时间，确保子进程有机会读取数据
        sleep(5); // 使用 sleep(1) 替代 usleep(1)，确保子进程有足够时间读取
    }
    else if (pid == 0) // 子进程
    {
        // 读取数据
        cout << "从内存映射区读取出来的数据: " << static_cast<char*>(ptr) << endl;
    }
    
    // 确保父进程等待子进程结束
    wait(NULL);
    
    // 解除映射并关闭文件
    munmap(ptr, 4000);
    close(fd);
    
    return 0;
}
```

**备注**: mmap不能去扩展一个内容为空的新文件，因为大小为0，所有本没有与之对应的合法的物理页，不能扩展。我们需要在新创建的空文件中先写入一些数据，否则会报错`bus error(总线错误)`

### 无血缘关系的进程通信

```cpp
//
// 写端
#include <iostream>
#include <sys/types.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <cstring>
#include <unistd.h>
#include <wait.h>
#include <fcntl.h>

using namespace std;

int main()
{
    int fd=open("test.txt",O_RDWR);
    if(fd==-1)
    {
        perror("open");
        return -1;
    }
    void* ptr=mmap(NULL,4096,PROT_READ|PROT_WRITE,MAP_SHARED,fd,0);
    if(ptr==MAP_FAILED)
    {
        perror("mmap");
        return -1;
    }
    const char* str="test";
    memcpy(ptr,str,strlen(str)+1);
    munmap(ptr,4096);
    return 0;
}
```

```cpp
//读端

#include <iostream>
#include <cstring>
#include <fcntl.h>  
#include <unistd.h>
#include <sys/mman.h>

using namespace std;

int main()
{
    int ret=open("test.txt",O_RDWR);
    if(ret==-1)
    {
        perror("open");
        return -1;
    }
    void* ptr=mmap(NULL,4096,PROT_READ|PROT_WRITE,MAP_SHARED,ret,0);
    if(ptr==MAP_FAILED)
    {
        perror("mmap");
        return -1;
    }
    cout<<"ptr:"<<(char*)ptr<<endl;
    munmap(ptr,4096);
    return 0;
}
```

## 基于内存映射实现文件拷贝

我们除了使用文件拷贝函数，也可以使用内存映射区实现文件拷贝，下面就是我们如何基于内存映射实现文件拷贝

```cpp
#include <iostream>
#include <cstring>
#include <cstdio>
#include <cstdlib>
#include <sys/mman.h>
#include <fcntl.h>
#include <unistd.h>

using namespace std;

int main()
{
    //打开待复制的文件
    int fd1=open("test.txt", O_RDONLY);
    if(fd1==-1)
    {
        perror("open");
        return 0;
    }
    //获取待复制文件的大小
    int size=lseek(fd1, 0, SEEK_END);
    //创建源文件的内存映射区
    void* ptr1=mmap(NULL, size, PROT_READ, MAP_SHARED, fd1, 0);
    if(ptr1==MAP_FAILED)
    {
        perror("mmap");
        return 0;
    }

    //打开目标文件
    int fd2=open("test2.txt", O_RDWR|O_CREAT|O_TRUNC, 0666);
    if(fd2==-1)
    {
        perror("open");
        return 0;
    }
    //创建目标文件的内存映射区
    void* ptr2=mmap(NULL, size, PROT_READ|PROT_WRITE, MAP_SHARED, fd2, 0);
    if(ptr2==MAP_FAILED)
    {
        perror("mmap");
        return 0;
    }

    //拓展文件大小，避免出现总线错误
    ftruncate(fd2, size);

    //拷贝文件
    memcpy(ptr2, ptr1, size);

    //关闭文件
    close(fd1);
    close(fd2);

    //解除内存映射区
    munmap(ptr1, 4096);
    munmap(ptr2, size);


    return 0;
}
```
