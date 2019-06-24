# Linux进程间通信-eventfd

eventfd是linux 2.6.22后系统提供的一个轻量级的进程间通信的系统调用，eventfd通过一个进程间共享的64位计数器完成进程间通信，这个计数器由在linux内核空间维护，用户可以通过调用write方法向内核空间写入一个64位的值，也可以调用read方法读取这个值。

## 创建方法

```c++
#include <sys/eventfd.h>
int eventfd(unsigned int initval, int flags);
```

创建的时候可以传入一个计数器的初始值initval。
第二个参数flags在linux 2.6.26之前的版本是没有使用的，必须初始化为0，在2.6.27之后的版本flag才被使用。

**EFD_CLOEXEC(2.6.27～):**

> A  copy  of  the  file descriptor created by eventfd() is inherited by the child produced by fork(2).  The duplicate file descriptor is associated with the same eventfd object.  File descriptors created by eventfd() are preserved across execve(2), unless the close-on-exec flag has been set.

eventfd()会返回一个文件描述符，如果该进程被fork的时候，这个文件描述符也会复制过去，这时候就会有多个的文件描述符指向同一个eventfd对象，如果设置了EFD_CLOEXEC标志，在子进程执行exec的时候，会清除掉父进程的文件描述符

**EFD_NONBLOCK(2.6.27～):** 就如它字面上的意思，如果没有设置了这个标志位，那read操作将会阻塞直到计数器中有值。如果没有设置这个标志位，计数器没有值的时候也会立即返回-1；

**EFD_SEMAPHORE(2.6.30～):** 这个标志位会影响read操作，具体可以看read方法中的解释

## 提供的方法

**read:** 读取计数器中的值

- 如果计数器中的值大于0  
  - 设置了EFD_SEMAPHORE标志位，则返回1，且计数器中的值也减去1。  
  - 没有设置EFD_SEMAPHORE标志位，则返回计数器中的值，且计数器置0。  
- 如果计数器中的值为0  
  - 设置了EFD_NONBLOCK标志位就直接返回-1。  
  - 没有设置EFD_NONBLOCK标志位就会一直阻塞直到计数器中的值大于0。  

**write:** 向计数器中写入值

- 如果写入值的和小于0xFFFFFFFFFFFFFFFE，则写入成功  
- 如果写入值的和大于0xFFFFFFFFFFFFFFFE  
  - 设置了EFD_NONBLOCK标志位就直接返回-1。  
  - 如果没有设置EFD_NONBLOCK标志位，则会一直阻塞知道read操作执行  

**close:** 关闭文件描述符

## 一个小Demo

```c
#include <sys/eventfd.h>
#include <unistd.h>
#include <iostream>

int main() {
    int efd = eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC);
    eventfd_write(efd, 2);
    eventfd_t count;
    eventfd_read(efd, &count);
    std::cout << count << std::endl;
    close(efd);
}
```

上面的程序中我们创建了一个eventfd,并将它的文件描述符保存在efd中，然后调用eventfd_write向计数器中写入数字2，然后调用eventfd_read读取计数器中的值并打印处理，最后关闭eventfd。
运行结果：

```text
count=2
```

### 多次read和write

```c++
#include <sys/eventfd.h>
#include <unistd.h>
#include <iostream>

int main() {
    int efd = eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC);
    eventfd_write(efd, 2);
    eventfd_write(efd, 3);
    eventfd_write(efd, 4);
    eventfd_t count;
    int read_result = eventfd_read(efd, &count);
    std::cout << "read_result=" << read_result << std::endl;
    std::cout << "count=" << count << std::endl;
    read_result = eventfd_read(efd, &count);
    std::cout << "read_result=" << read_result << std::endl;
    std::cout << "count=" << count << std::endl;
    close(efd);
}
```

运行结果：

```text
read_result=0
count=9
read_result=-1
count=9
```

从运行结果我们可以看出当多次调用eventfd_write的时候，计数器一直在累加，但是eventfd_read只需调用一次就可以将计数器中的数取出来，如果再次调用eventfd_read将会返回失败。

### EFD_NONBLOCK标志位的作用

```c++
#include <sys/eventfd.h>
#include <unistd.h>
#include <iostream>

int main() {
    int efd = eventfd(0, EFD_CLOEXEC);
    eventfd_write(efd, 2);
    eventfd_write(efd, 3);
    eventfd_write(efd, 4);
    eventfd_t count;
    int read_result = eventfd_read(efd, &count);
    std::cout << "read_result=" << read_result << std::endl;
    std::cout << "count=" << count << std::endl;
    read_result = eventfd_read(efd, &count);
    std::cout << "read_result=" << read_result << std::endl;
    std::cout << "count=" << count << std::endl;
    close(efd);
}
```

运行结果：

```text
read_result=0
count=9
```

和前一个运行结果直接返回-1相比，如果去掉EFD_NONBLOCK标志位，程序会在计数器没有值的情况下一直阻塞在eventfd_read方法。

### EFD_SEMAPHORE标志位的作用

```c++
#include <sys/eventfd.h>
#include <unistd.h>
#include <iostream>

int main() {
    int efd = eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC | EFD_SEMAPHORE);
    eventfd_write(efd, 2);
    eventfd_t count;
    int read_result = eventfd_read(efd, &count);
    std::cout << "read_result=" << read_result << std::endl;
    std::cout << "count=" << count << std::endl;
    read_result = eventfd_read(efd, &count);
    std::cout << "read_result=" << read_result << std::endl;
    std::cout << "count=" << count << std::endl;
    read_result = eventfd_read(efd, &count);
    std::cout << "read_result=" << read_result << std::endl;
    std::cout << "count=" << count << std::endl;
    close(efd);
}
```

运行结果：

```text
read_result=0
count=1
read_result=0
count=1
read_result=-1
count=1
```

可以看到设置了EFD_SEMAPHORE后，每次读取到的值都是1，且read后计数器也递减1
