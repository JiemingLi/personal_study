

# AOF重写

 Redis 将生成新 AOF 文件替换旧 AOF 文件的功能命名为 “AOF 文件重写”，AOF 文件重写并不需要对现有的 AOF 文件进行任何读取、分析或者写入操作，这个功能是通过读取服务器当前的数据库状态来实现。

例如目前redis服务器存在list的键，分别用六条push命令添加 "C" "D" "E""E""G"，如果服务器想要用尽量少的命令来记录 list键的状态，那么最简单高效的办法不是去读取和分析现有 AOF 文件的内容，而是直接从数据库中读取键 1ist 的值，然后用一条RPUSH list "C" "D" "E""E""G"命令来代替保存在 AOF 文件中的六条命令，这样就可以将保存 list键所需的命令从六条减少为一条了。



问题：如果redis服务器一直接受命令，不断往服务器里写入数据，那么aof的重写时间将会非常久，阻塞了服务器的正常使用。

解决：使用子进程进行重写（[copy on write技术](https://blog.csdn.net/Muscleape/article/details/105670481)）

# 混合持久化

混合持久化本质是通过 **AOF 后台重写**（bgrewriteaof 命令）完成的，不同的是当开启混合持久化时，fork 出的子进程先将当前全量数据以 RDB 方式写入新的 AOF 文件，然后再将 AOF 重写缓冲区（aof_rewrite_buf_blocks）的增量命令以 AOF 方式写入到文件，写入完成后通知主进程将新的含有 RDB 格式和 AOF 格式的 AOF 文件替换旧的的 AOF 文件。

[混合持久化](https://zhuanlan.zhihu.com/p/340082703)



## AOF 重写缓冲区内容过多怎么办

**在子进程进行aof重写结束之后，主进程把aof重写缓冲区的数据写入重写后的aof文件的时候会阻塞。**

若重写缓冲区的内容过多，从而导致主进程阻塞时间过长，导致redis服务器响应出问题。

**Redis 中已经针对这种情况进行了优化：**

1、在进行 AOF 后台重写时，Redis 会创建一组用于父子进程间通信的管道，同时会新增一个文件事件，该文件事件会将写入 AOF 重写缓冲区的内容通过该管道发送到子进程。

2、在重写结束后，子进程会通过该管道尽量从父进程读取更多的数据，每次等待可读取事件1ms，如果一直能读取到数据，则这个过程最多执行1000次，也就是1秒。如果连续20次没有读取到数据，则结束这个过程。

通过这些优化，Redis 尽量让 AOF 重写缓冲区的内容更少，以减少主进程阻塞的时间。



# AOF和RDB文件的特点

## RDB

1. 二进制文件，内存体积小
2. 适用于灾难恢复，因为体积小，传输快
3. redis服务器启动的时候加载快 
4. 数据完整性会丢一段时间内的数据

## AOF

1. 对于相同的数据集来说，AOF文件比RDB文件大
2. 通过配置appendsyn，可以保证最多丢失1s的数据，数据比RDB文件要更全

# 数据结构

hyperhyperlog：统计数据相关

geo：地理位置相关，实现附近的人查询距离等

zset和pub- subscribe实现延迟队列

布隆过滤器

# 过期策略和淘汰策略

# 持久化

# 通信协议

redis协议格式，解析，处理，响应

应用系统和Redis通过Redis协议(RESP)进行交互。

请求以字符串数组的形式来表示要执行命令的参数

客户端和服务器发送的命令或数据一律以 \r\n (CRLF)结尾。

在这个协议中， 所有发送至 Redis 服务器的参数都是二进制安全(binary safe)的。

简单，高效，易读。

## 6个规范

1、间隔符号，在Linux下是\r\n，在Windows下是\n

2、简单字符串 Simple Strings, 以 "+"加号 开头

3、错误 Errors, 以"-"减号 开头

4、整数型 Integer， 以 ":" 冒号开头

5、大字符串类型 Bulk Strings, 以 "$"美元符号开头，长度限制**512M** 

6、数组类型 Arrays，以 "*"星号开头

**示例** 

```tex
redis> SET mykey Hello
"OK"
```

实际发送的数据

```tex
*3\r\n$3\r\nSET\r\n$5\r\nmykey\r\n$5\r\nHello\r\n
*3 // 数组有3个参数
$3 // 第1个参数有3个字符
SET
$5 // 第2个参数有5个字符
mykey
$5 // 第3个参数有5个字符
Hello 
```

**好处：**

为了保障服务器能最快速的理解命令的含义而制定

如果没有这个通讯协议，那么 Redis 服务器端要遍历所有的空格以确认此条命令的含义，这样会加大服务器的运算量

直接发送通讯协议，相当于把服务器端的解析工作交给了**每一个客户端**，这样会很大程度的提高 Redis 的运行速度

# 命令处理的流程

流程：redis服务器启动监听，接收命令并解析，执行命令请求，返回命令结果

> socket 小知识：每个 socket 被创建后，会分配两个缓冲区，输入缓冲区和输出缓冲区。 写入函数并不会立即向网络中传输数据，而是先将数据写入缓冲区中，再由 TCP 协议将数据从缓冲区发送到目标机器。一旦将数据写入到缓冲区，函数就可以成功返回，不管它们有没有到达目标机器，也不管它们何时被发送到网络，这些都是 TCP 协议负责的事情。 注意：数据有可能刚被写入缓冲区就发送到网络，也可能在缓冲区中不断积压，多次写入的数据被一次性发送到网络，这取决于当时的网络情况、当前线程是否空闲等诸多因素，不由程序员控制。 读取函数也是如此，它也是从输入缓冲区中读取数据，而不是直接从网络中读取。

![image-20221026092128975](https://gitee.com/JieMingLi/document-pics/raw/master/image-20221026092128975.png)

## **服务器启动监听**

1， 调用 initServer方法

2， 创建eventLoop(事件机制) ？

3， 注册时间事件处理器？

4， 注册文件事件(socket)处理器？

5， 监听 socket 建立连接

**redis建立client**

1， redis-cli建立socket

2， redis-server为每个连接(socket)创建一个 Client 对象

3， 创建文件事件监听socket

4， 指定事件处理函数

## 获取命令

客户端和服务端建立好连接之后，服务端注册监听client的读写事件。

当有命令发送来的时候，socket将数据发送到client的缓冲区，服务端接收到事件消息，并且从缓冲区读取数据到内存。

## **解析命令**

1， 将输入缓冲区中的数据解析成对应的resp格式的命令

2， 判断单条还是多条命令，并调用不同的解析器解析

```tex
// 示例：set name:1 zhaoyun
*3(/r/n)
$3(/r/n)
set(/r/n)
$7(/r/n)
name:10(/r/n)
$7(/r/n)
zhaoyun(/r/n)
```

1， 解析命令参数的个数

首字符必须是“*”，使用"\r"定位到行尾，之间的数就是参数数量

2， 循环解析请求参数

首字符必须是"$"，使用"/r"定位到行尾，之间的数是参数的长度，从/n后到下一个"$"之间就是参数的值，循环解析直到没有"$"。

## **执行命令**

协议的执行包括**命令调用**和**返回结果**

**命令调用**

1， 判断参数个数和取出的参数是否一致

2， RedisServer解析完命令后,会调用函数`processCommand`处理该命令请求，如下图所示



![redis执行流程](https://gitee.com/JieMingLi/document-pics/raw/master/redis%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B.png)

3， 查找命令（set），不存在则抛异常

4， 参数数目校验，参数数目和解析出来的参数个数要匹配，如果不匹配则返回:“wrong number of arguments”错误。

5， 权限校验，最大内存校验，集群校验，持久化校验等

**返回结果**

5类:状态回复、错误回复、整数回复、批量回复、多条批量回复。

# 事件处理机制

1，socket， reactor模型

2，IO多路复用模型

3， 时间事件，文件事件，事件处理器，redis事件处理过程

4， 时间事件和文件事件的流程

------

Redis服务器是典型的事件驱动系统

有两大类事件，**文件事件和时间事件。**

1， 文件事件：Socket的读写事件，也就是IO事件，服务器对socket的操作抽象，有以下两大类型：

- readable：套接字变得可读（客户端对套接字进行write和close）和可连接（客户端对套接字进行connect）
- writeable：套接字变得可写（客户端对套接字进行read操作）

2， 时间事件：服务器对定时任务的抽象，类似`servercon`操作。

## 文件事件

**文件事件处理器**

定义：Redis基于`Reactor`模式开发出自己的网络事件处理器。

![image-20221027003049808](https://gitee.com/JieMingLi/document-pics/raw/master/image-20221027003049808.png)

### **IO多路复用**

负责监听多个文件事件，并通过任务队列，有序，同步地把文件事件向文件事件分派器进行传输，文件分派器收到文件事件后根据文件事件的类型（readable，writeable，可连接的，断开连接的）指派给对应的事件处理器。

> 复用指的是一个线程处理，多路指的是多个socket。I/O多路复用就是通过一种机制，一个进程可以监视多个描述符(socket)，一旦某个描述符就绪(一 般是读就绪或者写就绪)，能够通知程序进行相应的读写操作。

![图片](https://gitee.com/JieMingLi/document-pics/raw/master/640.png)

**只有上一个文件事件被事件处理器处理完之后，IO多路复用程序才会继续向文件分派器分配文件事件。**

**redis实现**

select，poll，epoll、kqueue都是IO多路复用的机制。

![image-20221027004939685](https://gitee.com/JieMingLi/document-pics/raw/master/image-20221027004939685.png)

```c
#ifdef HAVE_EVPORT
#include "ae_evport.c"
#else
    #ifdef HAVE_EPOLL
    #include "ae_epoll.c"
    #else
        #ifdef HAVE_KQUEUE
        #include "ae_kqueue.c"
        #else
        #include "ae_select.c"
        #endif
    #endif
#endif
```

Redis中支持多种IO复用，源码中使用相应的宏定义进行选择，编译时就可以获取当前系统支持的最优的IO复用函数来使用，从而实现了Redis的优秀的可移植特性。

> [select,poll,epoll如何使用](https://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/Redis%20%E6%BA%90%E7%A0%81%E5%89%96%E6%9E%90%E4%B8%8E%E5%AE%9E%E6%88%98/09%20%20Redis%E4%BA%8B%E4%BB%B6%E9%A9%B1%E5%8A%A8%E6%A1%86%E6%9E%B6%EF%BC%88%E4%B8%8A%EF%BC%89%EF%BC%9A%E4%BD%95%E6%97%B6%E4%BD%BF%E7%94%A8select%E3%80%81poll%E3%80%81epoll%EF%BC%9F.md)

### **文件事件分派器**

收到文件事件后根据文件事件的类型和具体的操作指派给对应的事件处理器。

事件的可读写是从服务器角度看的，分派看到的事件类型包括：

1. AE_READABLE 客户端写数据、关闭连接、新连接到达
2. AE_WRITEABLE 客户端读数据

> 当一个套接字连接同时可读可写时，服务器会优先处理读事件再处理写事件，也就是**读优先**。

### 事件处理器

> 各大事件处理器只是只是一段程序代码，由主线程，也只有1个线程执行对应的处理器，下文说的服务端就是指主线程。

#### 连接应答处理器

当 Redis 启动后，服务器程序的 main 函数会调用 initSever 函数来进行初始化，initServer 函数会根据启用的 IP 端口个数，为每个 IP 端口上的网络事件创建对 AE_READABLE 事件的监听并且注册 AE_READABLE 事件的处理 handler，这个handlle就是acceptTcpHandler

![img](https://gitee.com/JieMingLi/document-pics/raw/master/a37876d1e4330668ff8cab34af4d6098-20221013235144-5ddc2rn.jpg)

redis服务器初始化的时候，会把监听端口的套接字的`AE_READABLE`的文件事件和连接应答处理器关联，当客户端对服务端发起连接操作，套接字在服务端产生`AE_READABLE`事件，IO多路复用程序把该事件发给TaskQueue，文件分派器判断其是`AE_READABLE`事件并且是`connect`操作，所以把该事件分发给连接应答处理器。

功能：服务器在收到连接请求后会进行连接操作，然后为**客户端创建套接字**（新的文件描述符）和客户端的状态，并且把创建出来的客户端套接字的`AE_READABLE`事件和命令请求处理器关联。

#### 命令请求处理器

功能：主线程从套接字读取数据到数据缓冲区，并且处理读取进来的数据（读取完数据，解析完命令之再调用程序执行命令），处理结束后把套接字的`AE_WRITEABLE`事件和命令回复处理器关联。

客户端发送命令给套接字的时候，socket在服务器侧产生`AE_READABLE`文件事件，IO多路复用程序把该事件发给TaskQueue，文件分派器把该事件分发给命令请求处理器。

#### 命令回复处理器

功能：主线程把执行结果写入socket缓冲区，把套接字的`AE_WRITEABLE`事件和命令回复处理器取消关联。

因为在客户端发送请求命令给服务器的时候，会经过TaskQueue和事件分派器，然后交由<u>命令请求处理器</u>处理，主线程执行命令请求处理器之后会得到**命令执行后的结果**以及把`AE_WRITEABLE`事件和命令回复处理器关联，所以当客户端准备好接收命令的回复的时候，产生`AE_WRITEABLE`事件，被分发给命令回复处理器。

> 主线程只有上一个事件处理完之后，下一个事件才会到事件分排器，然后再到对应的事件处理器

#### 客户端与服务器完整连接示例

redis服务器初始化的时候会把`AE_READABLE`的文件事件和连接应答处理器关联，所以在redis运行中，客户端只要连接redis服务器，IO多路复用程序就会监听到socket的`AE_READABLE`事件，分发给TaskQueue和事件分派器，最后到达**连接应答处理器** ，建立完连接后主线程会把**新建出来的客户端的套接字**的`AE_READABLE`事件和命令请求处理器关联。

建立连接后的客户端，发功get请求，IO多路复用程序就会监听到客户端socket的`AE_READABLE`事件，通过TaskQueue和事件分派器，分发到**命令请求处理器**，主线程调用命令请求处理器（解析命令，执行命令等操作），执行完程序后主线程会把`AE_READABLE`事件和命令回复处理器关联。

当客户端尝试读取结果时产生`AE_WRITEABLE`事件，IO多路复用程序就会监听到并将该事件交给**命令回复处理器**。此时主线程端触发命令回复响应，将数据结果写入套接字，写入完成之后主线程解除该套接字与命令回复处理器之间的关联；

### 源码浅析

> [参考](https://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/Redis%20%E6%BA%90%E7%A0%81%E5%89%96%E6%9E%90%E4%B8%8E%E5%AE%9E%E6%88%98/10%20%20Redis%E4%BA%8B%E4%BB%B6%E9%A9%B1%E5%8A%A8%E6%A1%86%E6%9E%B6%EF%BC%88%E4%B8%AD%EF%BC%89%EF%BC%9ARedis%E5%AE%9E%E7%8E%B0%E4%BA%86Reactor%E6%A8%A1%E5%9E%8B%E5%90%97%EF%BC%9F.md)

Redis 的网络框架实现了 Reactor 模型并且自行开发实现了一个事件驱动框架，框架由Redis 代码实现文件是[ae.c](https://github.com/redis/redis/blob/5.0/src/ae.c)实现。

因为Reactor模型需要进行事件注册，捕获，分发，处理等操作，而且对于整个框架来说还需要一直运行。

相应地，redis定义了**事件的数据结构（aeFileEvent）、框架主循环函数(aeMain)、事件捕获分发函数(aeProcessEvents)、事件和 handler 注册函数**。

#### 事件数据结构：aeFileEvent

aeFileEvent 是一个结构体，它定义了 4 个成员变量 mask、rfileProce、wfileProce 和 clientData，

```c
typedef struct aeFileEvent {
    int mask; /* one of AE_(READABLE|WRITABLE|BARRIER) */ // 事件的类型，主要是READABLE和WRITABLE
    aeFileProc *rfileProc; // AE_READABLE对应的处理函数（也就是 Reactor 模型中的 handler）
    aeFileProc *wfileProc; // AE_WRITABLE对应的处理函数
    void *clientData; // 客户端的数据指针
} aeFileEvent;
```

#### 主循环：aeMain

```c
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        aeProcessEvents(eventLoop, AE_ALL_EVENTS|
                                   AE_CALL_BEFORE_SLEEP|
                                   AE_CALL_AFTER_SLEEP);
    }
}
```

服务器程序的 main 函数在完成 Redis server 的初始化后，会调用 aeMain 函数开始执行事件驱动框架。

在主循环调用事件捕获和分发的函数aeProcessEvents

#### 事件捕获和分发：aeProcessEvents

主要功能：捕获事件、判断事件类型和调用具体的事件处理函数，从而实现事件的处理。

```c
int aeProcessEvents(aeEventLoop *eventLoop, int flags)
{
    int processed = 0, numevents;

    /* 若没有事件处理，则立刻返回*/
    if (!(flags & AE_TIME_EVENTS) && !(flags & AE_FILE_EVENTS)) return 0;

    /* Note that we want to call select() even if there are no
     * file events to process as long as we want to process time
     * events, in order to sleep until the next time event is ready
     * to fire. */
  	/*如果有IO事件发生，或者紧急的时间事件发生，则开始处理*/
    if (eventLoop->maxfd != -1 ||
        ((flags & AE_TIME_EVENTS) && !(flags & AE_DONT_WAIT))) {
        int j;
        struct timeval tv, *tvp;
        int64_t usUntilTimer = -1;

        // 情况1
        if (flags & AE_TIME_EVENTS && !(flags & AE_DONT_WAIT))
            usUntilTimer = usUntilEarliestTimer(eventLoop);

        if (usUntilTimer >= 0) {
            tv.tv_sec = usUntilTimer / 1000000;
            tv.tv_usec = usUntilTimer % 1000000;
            tvp = &tv;
        } else {
            /* If we have to check for events but need to return
             * ASAP because of AE_DONT_WAIT we need to set the timeout
             * to zero */
            if (flags & AE_DONT_WAIT) {
                tv.tv_sec = tv.tv_usec = 0;
                tvp = &tv;
            } else {
                /* Otherwise we can block */
                tvp = NULL; /* wait forever */
            }
        }

        if (eventLoop->flags & AE_DONT_WAIT) {
            tv.tv_sec = tv.tv_usec = 0;
            tvp = &tv;
        }

        if (eventLoop->beforesleep != NULL && flags & AE_CALL_BEFORE_SLEEP)
            eventLoop->beforesleep(eventLoop);

        /* Call the multiplexing API, will return only on timeout or when
         * some event fires. */
      	// 调用多路复用 API，将仅在超时或某些事件触发时返回。
        // 调用aeApiPoll函数捕获事件
        numevents = aeApiPoll(eventLoop, tvp);

        /* After sleep callback. */
        // 情况2
        if (eventLoop->aftersleep != NULL && flags & AE_CALL_AFTER_SLEEP)
            eventLoop->aftersleep(eventLoop);

        for (j = 0; j < numevents; j++) {
            int fd = eventLoop->fired[j].fd;
            aeFileEvent *fe = &eventLoop->events[fd];
            int mask = eventLoop->fired[j].mask;
            int fired = 0; /* Number of events fired for current fd. */
            int invert = fe->mask & AE_BARRIER;
            if (!invert && fe->mask & mask & AE_READABLE) {
                fe->rfileProc(eventLoop,fd,fe->clientData,mask);
                fired++;
                fe = &eventLoop->events[fd]; /* Refresh in case of resize. */
            }

            /* Fire the writable event. */
          	// 处理写事件
            if (fe->mask & mask & AE_WRITABLE) {
                if (!fired || fe->wfileProc != fe->rfileProc) {
                    fe->wfileProc(eventLoop,fd,fe->clientData,mask);
                    fired++;
                }
            }

            /* If we have to invert the call, fire the readable event now
             * after the writable one. */
          	// 处理读事件
            if (invert) {
                fe = &eventLoop->events[fd]; /* Refresh in case of resize. */
                if ((fe->mask & mask & AE_READABLE) &&
                    (!fired || fe->wfileProc != fe->rfileProc))
                {
                    fe->rfileProc(eventLoop,fd,fe->clientData,mask);
                    fired++;
                }
            }

            processed++;
        }
    }
    /* Check time events */
    /* 检查是否有时间事件，若有则调用processTimeEvents函数处理 */
    // 情况3
    if (flags & AE_TIME_EVENTS)
        processed += processTimeEvents(eventLoop);

    return processed; /* return the number of processed file/time events */
}
```

可以看到主要有三个 if 条件分支，分别对应了以下三种情况：

- 情况一：既没有时间事件，也没有网络事件；
- 情况二：有 IO 事件或者有需要紧急处理的时间事件；
- 情况三：只有普通的时间事件。

重点观察情况二，Redis 需要捕获发生的网络事件，并进行相应的处理，**aeApiPoll 函数会被调用，用来捕获事件**

由注释可知，aeApiPoll是Redis在各个平台上的IO多路复用程序，在Linux上为epoll，对应的redis的文件是**e_epoll.c**，Linux 上提供了 **epoll_wait API**，用于检测内核中发生的网络 IO 事件。在[ae_epoll.c](https://github.com/redis/redis/blob/5.0/src/ae_epoll.c)文件中，**aeApiPoll 函数**就是封装了对 **epoll_wait** 的调用。

**aeApiPoll 函数**

```c
static int aeApiPoll(aeEventLoop *eventLoop, struct timeval *tvp) {
    aeApiState *state = eventLoop->apidata;
    int retval, numevents = 0;

  	//调用epoll_wait获取监听到的事件 
    retval = epoll_wait(state->epfd,state->events,eventLoop->setsize,
            tvp ? (tvp->tv_sec*1000 + (tvp->tv_usec + 999)/1000) : -1);
    if (retval > 0) {
        int j;

      	//获得监听到的事件数量
        numevents = retval;
      	//针对每一个事件，进行处理
        for (j = 0; j < numevents; j++) {
            int mask = 0;
            struct epoll_event *e = state->events+j;
            if (e->events & EPOLLIN) mask |= AE_READABLE;
            if (e->events & EPOLLOUT) mask |= AE_WRITABLE;
            if (e->events & EPOLLERR) mask |= AE_WRITABLE|AE_READABLE;
            if (e->events & EPOLLHUP) mask |= AE_WRITABLE|AE_READABLE;
          	// 保存每个待处理事件的文件描述符
            eventLoop->fired[j].fd = e->data.fd;
            eventLoop->fired[j].mask = mask;
        }
    } else if (retval == -1 && errno != EINTR) {
        panic("aeApiPoll: epoll_wait, %s", strerror(errno));
    }
    return numevents;
}
```

从main调用到aeApiPoll的流程图

![img](https://gitee.com/JieMingLi/document-pics/raw/master/923921e50b117247de69fe6c657845e0-20221013235145-ocaj662.jpg)

#### 事件注册：aeCreateFileEvent 函数

当 Redis 启动后，服务器程序的 main 函数会调用 initSever 函数来进行初始化，而在初始化的过程中，aeCreateFileEvent 就会被 initServer 函数调用，用于注册要监听的事件，以及相应的事件处理函数。

initServer 函数会根据启用的 IP 端口个数，为每个 IP 端口上的网络事件，调用 aeCreateFileEvent，创建对 AE_READABLE 事件的监听，并且注册 AE_READABLE 事件的处理 handler，也就是 acceptTcpHandler 函数。

![img](https://gitee.com/JieMingLi/document-pics/raw/master/a37876d1e4330668ff8cab34af4d6098-20221013235144-5ddc2rn-20221029004423061.jpg)

 initServer 中调用 aeCreateFileEvent 的部分片段如下

```c
void initServer(void) {
    for (j = 0; j < server.ipfd_count; j++) {
      	// 服务器初始化，为每个端口的网络事件设置连接监听。
      	// 连接事件类型：AE_READABLE
        // 对应的处理函数：acceptTcpHandler
        if (aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE,
            acceptTcpHandler,NULL) == AE_ERR)
            {
                serverPanic("Unrecoverable error creating server.ipfd file event.");
            }
  }
}
```

**AE_READABLE 事件就是客户端的网络连接事件，而对应的处理函数就是接收 TCP 连接请求**。

aeCreateFileEvent如下代码所示

```c
int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask,
        aeFileProc *proc, void *clientData)
{
    if (fd >= eventLoop->setsize) {
        errno = ERANGE;
        return AE_ERR;
    }
    aeFileEvent *fe = &eventLoop->events[fd];
		// Linux 提供了 epoll_ctl API，用于增加新的观察事件
    // aeApiAddEvent 函数，对 epoll_ctl 进行调用。
    if (aeApiAddEvent(eventLoop, fd, mask) == -1)
        return AE_ERR;
    fe->mask |= mask;
    if (mask & AE_READABLE) fe->rfileProc = proc;
    if (mask & AE_WRITABLE) fe->wfileProc = proc;
    fe->clientData = clientData;
    if (fd > eventLoop->maxfd)
        eventLoop->maxfd = fd;
    return AE_OK;
}
```

aeApiAddEvent代码如下所示

```c
static int aeApiAddEvent(aeEventLoop *eventLoop, int fd, int mask) {
    aeApiState *state = eventLoop->apidata;
    struct epoll_event ee = {0}; /* avoid valgrind warning */
    int op = eventLoop->events[fd].mask == AE_NONE ?
            EPOLL_CTL_ADD : EPOLL_CTL_MOD;

    ee.events = 0;
    mask |= eventLoop->events[fd].mask; /* Merge old events */
    if (mask & AE_READABLE) ee.events |= EPOLLIN;
    if (mask & AE_WRITABLE) ee.events |= EPOLLOUT;
    ee.data.fd = fd;
  	// Linux 提供了 epoll_ctl API，用于增加新的观察事件
    if (epoll_ctl(state->epfd,op,fd,&ee) == -1) return -1;
    return 0;
}
```

由上可知，aeCreateFileEvent 就会调用 aeApiAddEvent，然后 aeApiAddEvent 再通过调用 epoll_ctl来注册希望监听的事件和相应的处理函数。等到 aeProceeEvents 函数d捕获到实际事件时（aeApiPoll），它就会调用注册的函数对事件进行处理了（fe->wfileProc ｜ fe->rfileProc）。

#### 小结

1， Redis在监听端口的socket的文件事件处理完之后，会新建出客户端的套接字并且把它的ae_readable事件和命令处理器关联（acceptTcpHandler中调用aeCreateEventFile函数，fe->rfileProc）

2， **主循环**继续执行aeProceeEvents等待捕获连接成功后新建出来的客户端的ae_readable事件，然后执行命令，为了让客户端可以读取到数据，会把客户端的套接字的ae_writeable事件和命令回复处理器关联（命令处理器中继续调用aeCreateEventFile函数，fe->rfileProc）

3， **主循环**继续执行aeProceeEvents，等客户端可以读取数据之后，产生ae_writeable事件，主线程然后捕获到该事件，并执行命令回复处理（把数据写入socket缓冲区），最后再把客户端的套接字的ae_writeable事件和命令回复处理器取消关联（命令回复处理器调用aeDelEventFIle函数，fe->wfileProc）。

### 总流程图

![redis_file_event_process](https://gitee.com/JieMingLi/document-pics/raw/master/redis_file_event_process.jpeg)

## 时间事件

分为定时事件和周期事件

**时间事件的组成**

- id(全局唯一id)
- when (毫秒时间戳，记录了时间事件的到达时间) 
- timeProc(时间事件处理器，当时间到达时，Redis就会调用相应的处理器来处理事件)

![图片](https://gitee.com/JieMingLi/document-pics/raw/master/640-20221029013309458.png)

```c
/* Time event structure
* 时间事件结构
*/
typedef struct aeTimeEvent {
// 时间事件的唯一标识符
long long id; /* time event identifier. */
// 事件的到达时间，存贮的是UNIX的时间戳
long when_sec; /* seconds */
long when_ms; /* milliseconds */
// 事件处理函数，当到达指定时间后调用该函数处理对应的问题 aeTimeProc *timeProc;
// 事件释放函数
aeEventFinalizerProc *finalizerProc;
// 多路复用库的私有数据
void *clientData;
// 指向下个时间事件结构，形成链表
struct aeTimeEvent *next;
} aeTimeEvent;
```

Redis的时间事件是存储在链表中的，并且是按照ID存储的，新事件在头部旧事件在尾部，但是并不是按照即将被执行的顺序存储的。第一个元素50ms后执行，但是第三个可能30ms后执行，这样的话Redis每次从链表中获取最近要执行的事件时，都需要进行O(N)遍历。（可以使用最小栈的思路）

>  Redis中的时间事件数量并不多，即使进行O(N)遍历性能损失也微乎其微，也就不必每次插入新事件时进行链表重排。

Redis存储时间事件的无序链表如图：

![图片](https://gitee.com/JieMingLi/document-pics/raw/master/640-20221029014218015.png)

### 周期事件

周期性事件:让一段程序每隔指定时间就执行一次（循环执行）

当一个时间事件到达后，服务器会根据时间处理器的返回值，对时间事件的 when 属性进行更新，让这 个事件在一段时间后再次达到。

**serverCron**

serverCron就是一个典型的周期性事件。

redis服务器需要对自身的资源与配置进行定期的调整，从而确保服务器的长久运行，这些操作由redis.c中的**serverCron**函数实现，主要有以下操作：

1. **更新**redis服务器各类**统计信息**，包括时间、内存占用、数据库占用等情况。（更新信息）
2. **清理**数据库中的**过期键值对**。（清理过期的key-value）
3. 关闭和清理连接失败的客户端。（清理过期的client）
4. 尝试进行aof和rdb持久化操作。（try rdb or aof）
5. 如果服务器是主服务器，会定期将数据向从服务器做同步操作。（master sync）
6. 如果处于集群模式，对集群定期进行同步与连接测试操作。（cluster sync and test）

redis服务启动后，会周期性调用serverCron函数，保证服务器的文件，默认10s一次，可以通过配置文件修改。

### 定时事件

定时事件:让一段程序在**指定的时间**之后执行一次（定1个1次性的闹钟）

该事件在达到后删除，之后不会再重复。

## 单线程模式中文件事件和时间事件的调度和执行

问题：Redis服务中因为包含了时间事件和文件事件，事情也就变得复杂了，服务器要决定何时处理文件事件、何时处理时间事件、并且还要明确知道处理时间的时间长度

Redis服务器会轮流处理文件事件和时间事件，这两种事件的处理都是同步、有序、原子地执行的，服务器也不会终止正在执行的事件，也不会对事件进行抢占。

**事件调度的规则**

文件事件是随机出现的，如果处理完成一次文件事件后，仍然没有其他文件事件到来，服务器将继续等待文件事件的到来和时间事件的接近。在文件事件的不断执行中，时间会逐渐向最早的时间事件所设置的到达时间逼近并最终来到到达时间，这时服务器就可以开始处理到达的时间事件了。

> 由于时间事件在文件事件之后执行，并且事件之间不会出现抢占，所以时间事件的实际处理时间一般会比设定的时间稍晚一些。

在文件事件的源码分析中可知，主线程循环调用aeProcessEvents，而aeProcessEvents负责事件的捕获和分发处理，其中就包括文件事件和时间事件。

也就是说redis对事件的调度机制都在aeProcessEvents代码里（*可以参考源码分析中关于aeProcessEvents的分析*）

```c
int aeProcessEvents(aeEventLoop *eventLoop, int flags)
{
    int processed = 0, numevents;

    /* 若没有事件处理，则立刻返回*/
    if (!(flags & AE_TIME_EVENTS) && !(flags & AE_FILE_EVENTS)) return 0;

    /* Note that we want to call select() even if there are no
     * file events to process as long as we want to process time
     * events, in order to sleep until the next time event is ready
     * to fire. */
  	/*如果有IO事件发生，或者紧急的时间事件发生，则开始处理*/
    if (eventLoop->maxfd != -1 ||
        ((flags & AE_TIME_EVENTS) && !(flags & AE_DONT_WAIT))) {
        int j;
        struct timeval tv, *tvp;
        int64_t usUntilTimer = -1;

        // 情况1
        if (flags & AE_TIME_EVENTS && !(flags & AE_DONT_WAIT))
          	//计算距离下一次发生时间事件的时间间隔
            usUntilTimer = usUntilEarliestTimer(eventLoop);
				
       	// 	转化时间
        if (usUntilTimer >= 0) {
            tv.tv_sec = usUntilTimer / 1000000;
            tv.tv_usec = usUntilTimer % 1000000;
            tvp = &tv;
        } else { //没有时间事件，所以下一次时间的间隔为0
            /* If we have to check for events but need to return
             * ASAP because of AE_DONT_WAIT we need to set the timeout
             * to zero */
          	//马上返回，不阻塞
            if (flags & AE_DONT_WAIT) {
                tv.tv_sec = tv.tv_usec = 0;
                tvp = &tv;
            } else {
                /* Otherwise we can block */
              	//阻塞到文件事件发生
                tvp = NULL; /* wait forever */
            }
        }

        if (eventLoop->flags & AE_DONT_WAIT) {
            tv.tv_sec = tv.tv_usec = 0;
            tvp = &tv;
        }

        if (eventLoop->beforesleep != NULL && flags & AE_CALL_BEFORE_SLEEP)
            eventLoop->beforesleep(eventLoop);

        /* Call the multiplexing API, will return only on timeout or when
         * some event fires. */
      	// 调用多路复用 API，将仅在超时或某些事件触发时返回。
        // 调用aeApiPoll函数捕获事件
      	// 等待文件事件发生，tvp为超时时间，超时马上返回(tvp为0表示马上，为null表示阻塞到事件发生)
        numevents = aeApiPoll(eventLoop, tvp);

        /* After sleep callback. */
        // 情况2
        if (eventLoop->aftersleep != NULL && flags & AE_CALL_AFTER_SLEEP)
            eventLoop->aftersleep(eventLoop);

      	//处理触发的文件事件
        for (j = 0; j < numevents; j++) {
            int fd = eventLoop->fired[j].fd;
            aeFileEvent *fe = &eventLoop->events[fd];
            int mask = eventLoop->fired[j].mask;
            int fired = 0; /* Number of events fired for current fd. */
            int invert = fe->mask & AE_BARRIER;
            if (!invert && fe->mask & mask & AE_READABLE) {
                fe->rfileProc(eventLoop,fd,fe->clientData,mask);
                fired++;
                fe = &eventLoop->events[fd]; /* Refresh in case of resize. */
            }

            /* Fire the writable event. */
          	// 处理写事件
            if (fe->mask & mask & AE_WRITABLE) {
                if (!fired || fe->wfileProc != fe->rfileProc) {
                    fe->wfileProc(eventLoop,fd,fe->clientData,mask);
                    fired++;
                }
            }

            /* If we have to invert the call, fire the readable event now
             * after the writable one. */
          	// 处理读事件
            if (invert) {
                fe = &eventLoop->events[fd]; /* Refresh in case of resize. */
                if ((fe->mask & mask & AE_READABLE) &&
                    (!fired || fe->wfileProc != fe->rfileProc))
                {
                    fe->rfileProc(eventLoop,fd,fe->clientData,mask);
                    fired++;
                }
            }

            processed++;
        }
    }
    /* Check time events */
    /* 检查是否有时间事件，若有则调用processTimeEvents函数处理 */
    if (flags & AE_TIME_EVENTS)
      	// processTimeEvents： 取得当前时间，循环时间事件链表，如果当前时间>=预订执行时间，则执行时间处理函数。
        processed += processTimeEvents(eventLoop);
		
   	//	返回处理事件的个数
    return processed; /* return the number of processed file/time events */
}
```

**总结：**

aeProcessEvents 都会先 **计算最近的时间事件发生所需要等待的时间** ，然后调用 aeApiPoll 方法在这 段时间中等待事件的发生，在这段时间中如果发生了文件事件，就会优先处理文件事件，否则就会一直 等待，直到最近的时间事件需要触发。

![图片](https://gitee.com/JieMingLi/document-pics/raw/master/640-20221029020447956.png)

# 发布与订阅

# pipeline

# 慢查询日志

# 事务

# 高可用

1， 主从配置，数据同步问题，leader宕机问题

2， 哨兵模式；sentinel建立连接（订阅链接，命令链接）；如何和其他sentinel通信；如何判断节点下线（流程）；通过投票（raft）选出来的sentinel，在slave中如何选出master，master恢复后怎么做

3， 集群分区，客户端分区（中心化），proxy分区（中心化），官方cluster分区（去中心化）

# 分布式锁

1， setnx各类问题和解决思路

2， redlock解决了什么问题

3， redission解决了什么问题

# 其他

1， 大key

2， 缓存雪崩，缓存击穿，缓存穿透

3， 如何保证一致性（延迟双删）

4， redis为什么这么快

5， 热key

# redis中使用多线程的场景

# gossip，raft协议在redis的应用

