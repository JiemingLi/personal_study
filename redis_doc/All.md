



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

## 主从复制

主要存储数据的节点叫做主节点 (master），把其他通过复制主节点数据的副本节点叫做从节点 (slave），在 Redis 中一个主节点可以拥有多个从节点，一个从节点也可以是其他服务器的主节点

![主从同步-从从模式.png](https://gitee.com/JieMingLi/document-pics/raw/master/369eda70-800a-11ea-b751-6ff511beda88.png)

### **作用**

1， 一主多从，主从同步。

2， 主负责写，从负责读，提升Redis的性能和吞吐量。

3， 利用哨兵可以实现主从切换，做到高可用

### 实现

**Redis 2.8以前**

Redis的同步功能分为同步(sync)和命令传播(command propagate)。

**同步操作:**

1. 通过从服务器发送到SYNC命令给主服务器
2. 服务器生成RDB文件并发送给从服务器，同时发送保存所有写命令给从服务器
3. 从服务器清空之前数据并执行解释RDB文件
4. 保持数据一致(还需要命令传播过程才能保持一致)

命令传播：同步操作完成后，主服务器执行写命令，该命令发送给从服务器并执行，使主从保存一致。

缺点：

1， 没有全量同步和增量同步的概念，从服务器在同步时，会清空所有数据。

2， 主从服务器断线后重复制，主服务器会重新生成RDB文件和重新记录缓冲区的所有命令，并全量同步到从服务器上，增加带宽成本。

**Redis 2.8以后**

服务器 runId ：无论主库还是从库都有自己的 RUN ID，RUN ID 启动时自动产生，RUN ID 由40个随机的十六进

> 1. 当从库对主库初次复制时，主库将自身的 RUN ID 传送给从库，从库会将 RUN ID 保存；
> 2. 当从库断线重连主库时，从库将向主库发送之前保存的 RUN ID；
>    - 从库 RUN ID 和主库 RUN ID 一致，说明从库断线前复制的就是当前的主库；主库尝试执行 增量同步操作；
>    - 若不一致，说明从库断线前复制的主库并不时当前的主库，则主库将对从库执行全量同步操作；

 复制偏移量 offset：主从都会维护一个复制偏移量；

- 主库向从库发送N个字节的数据时，将自己的复制偏移量上加N；
- 从库接收到主库发送的N个字节数据时，将自己的复制偏移量加上N；

> 通过比较主从偏移量得知主从之间数据是否一致；偏移量相同则数据一致；偏移量不同则数据不一

使用PSYNC命令，具备完整重同步和部分重同步模式。

- Redis 的主从同步，分为**全量同步**和**增量同步**。
- 只有从机第一次连接上主机是**全量同步**。
- 断线重连有可能触发**全量同步**也有可能是**增量同步**( master 判断 runid 是否一致，主要是为了让新的slave都可以进行同步数据)。![在这里插入图片描述](https://gitee.com/JieMingLi/document-pics/raw/master/b89341cfdb414723891504a2fb4c571d.png)

**Redis的全量同步过程主要分三个阶段:**

- **同步快照阶段**：Master 创建并发送**快照**RDB给 Slave ， Slave 载入并解析快照。 Master 同时将  此阶段所产生的新的写命令存储到缓冲区。
- **同步写缓冲阶段**： Master 向 Slave 同步存储在缓冲区的写操作命令。
- **同步增量阶段:**  Master 向 Slave 同步写操作命令。![image-20221101013034023](https://gitee.com/JieMingLi/document-pics/raw/master/image-20221101013034023.png)

**增量同步**

- Redis增量同步主要指Slave完成初始化后开始正常工作时， Master 发生的写操作同步到 Slave 的 过程，通常情况下， Master 每执行一个写命令就会向 Slave 发送相同的**写命令**，然后 Slave 接收并执 行。
- 如果全量复制过程中，master-slave 网络连接断掉，那么 slave 重新连接 master 时，可能会触发增量同步。

**增量同步的场景：**

1. 主从第一次连接成功，并且进行完全量同步之后，主服务器接收到一条命令就传一条命令给从服务器。
2. 在全量同步的过程中发生断网，并且重新连接网络的时候，会进行增量同步，但是有可能为全量同步，取决于主服务的复制积压缓冲区有没有填满。
   1. 从服务器中途断开，比如 突然断电等，连接上以后，这个时候就可以尝试增量同步，如果可以增量，那么就不用动用全量这个重型操作，在主从服务器建立连接后，他们之间都维护长连接并彼此发送心跳命令，其实主从服务器彼此都有心跳机制，各自模拟成对方的客户端进行通信，从服务每个 1秒 发送 `REPLCONF ACK <offset>` 信息，像主服务器报告当前复制偏移量。
   2. 在主服务器有一个复制积压缓冲区（1M大的循环队列），因为主服务器在命令传播时，不仅会将命令发送给所有的从服务器，还会将命令写入 复制积压缓冲区中，复制积压缓冲区 最多可以备份1M大小的数据，如果主从服务器断线时间过长，复制积压缓冲区 的数据会被新数据覆盖（大于1M），那么当从主从中断连接起，此时从服务器无法增量同步，只能全量同步。

[全量同步和增量同步的区别](https://blog.csdn.net/qq_21539375/article/details/124719577)

**心跳检测的作用**

在命令传播阶段，从服务器默认会以每秒1次的频率向主服务器发送命令，有以下的作用：

1. 检测主从服务器的网络连接状态

2. 检测命令丢失，补发数据
   1. 主从已经连接后，主服务器传播给从服务器的写命令在半路丢失（网络故障），那么当从服务器向主服务器发 送REPLCONF ACK命令时，主服务器将发觉从服务器当前的复制偏移量少于自己的复制偏移量， 然后主服务器就会根据从服务器提交的复制偏移量，在复制积压缓冲区里面找到从服务器缺少的数 据，并将这些数据重新发送给从服务器。
   2. 在主从全量复制过程中，断网后再连接触发的增量同步。

### 原理

> Redis 源码中采用的基于**状态机**跳转的设计思路和主从复制的实现。

每一个 Redis 实例在代码中都对应一个 **redisServer 结构体**，这个结构体包含了和 Redis 实例相关的各种配置，比如实例的 RDB、AOF 配置、主从复制配置、切片集群配置等。与主从复制状态机相关的变量是 **repl_state**，Redis 在进行主从复制时，从库就是根据这个变量值的变化，来实现不同阶段的执行和跳转。

```c
struct redisServer {
   ...
   /* 复制相关(slave) */
    char *masterauth;               /* 用于和主库进行验证的密码*/
    char *masterhost;               /* 主库主机名 */
    int masterport;                 /* 主库端口号r */
    …
    client *master;        /* 从库上用来和主库连接的客户端 */
    client *cached_master; /* 从库上缓存的主库信息 */
    int repl_state;          /* 从库的复制状态机 */
   ...
}
```

**初始化**

redis实例启动后调用 server.c 中的 initServerConfig 函数，初始化 redisServer 结构体，此时实例会把状态机的初始状态设置为 **REPL_STATE_NONE**

```c
void initServerConfig(void) {
   …
   server.repl_state = REPL_STATE_NONE;
   …
}
```

客户端向从服务器发送slaveof(replicaof) 主机地址(127.0.0.1) 端口(6379)时:从服务器将主机 ip(127.0.0.1)和端口(6379)保存到redisServer的masterhost和masterport中，然后从服务器将向发送SLAVEOF命令的客户端返回OK，表示复制指令已经被接收，而实际上复制工作是在 OK返回之后进行。

在保存完端口和ip之后，会把状态机的设置为**REPL_STATE_CONNECT**，此时代表着redis服务器主从复制的初始化阶段就完成。

![img](https://gitee.com/JieMingLi/document-pics/raw/master/7c46c8f72f4391d29a6bcdyy8a64e6e1-20221014000136-q0a9fkf.jpg)

**建立socket连接**

建立连接的时机：Redis有周期事件，Redis 实例在运行时，按照一定时间周期重复执行的任务，其中周期事件中就包括 replicationCron()，默认1000ms执行一次。

```c
int serverCron(struct aeEventLoop *eventLoop, long long id, void *clientData) {
   …
   run_with_period(1000) replicationCron();
   …
}
```

replicationCron() 任务的函数实现逻辑是在 server.c 中，在replicationCron()中会检查从库的复制状态机状态。如果状态机状态是 REPL_STATE_CONNECT，那么从库就开始和主库建立连接，通过调用connectWithMaster()函数进行调用。

```c
replicationCron() {
   …
   /* 如果从库实例的状态是REPL_STATE_CONNECT，那么从库通过connectWithMaster和主库建立连接 */
    if (server.repl_state == REPL_STATE_CONNECT) {
        serverLog(LL_NOTICE,"Connecting to MASTER %s:%d",
            server.masterhost, server.masterport);
        if (connectWithMaster() == C_OK) { 
            serverLog(LL_NOTICE,"MASTER <-> REPLICA sync started");
        }
    }
    …
}
```

从库实例调用 connectWithMaster 函数，在connectWithMaster 函数里会先通过 anetTcpNonBlockBestEffortBindConnect 函数和主库建立连接。连接成功后创建出客户端socket，并监听该socket的读写事件，监听后到读写事件后使用回调syncWithMaster函数。注册完之后把状态机置为 REPL_STATE_CONNECTING

```c
int connectWithMaster(void) {
    int fd;
    //从库和主库建立连接
 fd = anetTcpNonBlockBestEffortBindConnect(NULL, server.masterhost,server.masterport,NET_FIRST_BIND_ADDR);
    …
 
//在建立的连接上注册读写事件，对应的回调函数是syncWithMaster
 if(aeCreateFileEvent(server.el,fd,AE_READABLE|AE_WRITABLE,syncWithMaster, NULL) ==AE_ERR)
    {
        close(fd);
        serverLog(LL_WARNING,"Can't create readable event for SYNC");
        return C_ERR;
    }
 
    //完成连接后，将状态机设置为REPL_STATE_CONNECTING
    …
    server.repl_state = REPL_STATE_CONNECTING;
    return C_OK;
}
```

当从库实例的状态变为 REPL_STATE_CONNECTING 时，建立连接的阶段就完成。

![img](https://gitee.com/JieMingLi/document-pics/raw/master/dd6176abeb3ba492f15a93bd0b4a84aa-20221014000136-5oorkbx.jpg)

**主动握手**

当主从库建立网络连接后，从库实例其实并没有立即开始进行数据同步，而是会先和主库之间进行握手通信。

握手的目的：从库和主库进行验证，以及从库将自身的 IP 和端口号发给主库。

在建立完连接后，从库实例的状态处于**REPL_STATE_CONNECTING**状态，回调函数syncWithMaster判断如果状态处于**REPL_STATE_CONNECTING**则会发送PING给主库，并将状态机置为 **REPL_STATE_RECEIVE_PONG**。

当从库收到主库返回的 PONG 消息后，从库会依次给主库发送验证信息、端口号、IP、对 RDB 文件和无盘复制（runId和offset）的支持情况。

每次发送都对应着状态机状态的变迁，例如：当从库要给主库发送验证信息前，会将自身状态机置为 REPL_STATE_SEND_AUTH，然后，从库给主库发送实际的验证信息。验证信息发送完成后，从库状态机会变迁为 REPL_STATE_RECEIVE_AUTH，并开始读取主库返回验证结果信息。除了验证下信息，端口号，ip和rdb文件等都是很类似。

![img](https://gitee.com/JieMingLi/document-pics/raw/master/c2946565d547bd52063ff1a79ec426cf-20221014000136-acvcy0i.jpg)

**复制类型判断与执行阶段**

主从完成握手后，状态机为 **REPL_STATE_RECEIVE_CAPA**，然后从库的状态变迁为 **REPL_STATE_SEND_PSYNC**，表明要开始向主库发送 PSYNC 命令，开始实际的数据同步。

方式：从库会调用 slaveTryPartialResynchronization 函数，向主库发送 PSYNC 命令，并且状态机的状态会置为 **REPL_STATE_RECEIVE_PSYNC**

```c
 /* 1, 从库状态机进入REPL_STATE_RECEIVE_CAPA. */
 if (server.repl_state == REPL_STATE_RECEIVE_CAPA) {
  …
  //2, 读取主库返回的CAPA消息响应，并修改状态
       server.repl_state = REPL_STATE_SEND_PSYNC;
  }
  //3, 从库状态机变迁为REPL_STATE_SEND_PSYNC后，开始调用slaveTryPartialResynchronization函数向主库发送PSYNC命令，进行数据同步
  if (server.repl_state == REPL_STATE_SEND_PSYNC) {
    	 // 4， 调用slaveTryPartialResynchronization，发送PSYNC命令
       if (slaveTryPartialResynchronization(fd,0) == PSYNC_WRITE_ERROR)  
       {
             …
       }
       server.repl_state = REPL_STATE_RECEIVE_PSYNC;
          return;
  }
```

从库调用slaveTryPartialResynchronization，向主库发送PSYNC命令，主库收到命令后，会根据从库发送的主库 ID、复制进度值 offset，来判断是进行全量复制还是增量复制，或者是返回错误。

slaveTryPartialResynchronization代码如下所示

```c
int slaveTryPartialResynchronization(int fd, int read_reply) {
   …
   //发送PSYNC命令
   if (!read_reply) {
      //从库第一次和主库同步时，设置offset为-1
  server.master_initial_offset = -1;
  …
  //调用sendSynchronousCommand发送PSYNC命令
  reply =
  sendSynchronousCommand(SYNC_CMD_WRITE,fd,"PSYNC",psync_replid,psync_offset,NULL);
   …
   //发送命令后，等待主库响应
   return PSYNC_WAIT_REPLY;
   }
 
  //读取主库的响应
  reply = sendSynchronousCommand(SYNC_CMD_READ,fd,NULL);
 
 //主库返回FULLRESYNC，全量复制
  if (!strncmp(reply,"+FULLRESYNC",11)) {
   …
   return PSYNC_FULLRESYNC;
   }
 
  //主库返回CONTINUE，执行增量复制
  if (!strncmp(reply,"+ CONTINUE",11)) {
  …
  return PSYNC_CONTINUE;
   }
 
  //主库返回错误信息
  if (strncmp(reply,"-ERR",4)) {
     …
  }
  return PSYNC_NOT_SUPPORTED;
}
```

第一次调用slaveTryPartialResynchronization，发送完命令之后等待主库相应（此时状态机的状态为REPL_STATE_RECEIVE_PSYNC）

由于socket的读写事件都绑定了syncWithMaster函数，所以在主服务发送数据回来之后，从服务器的监听到ae_readable事件并继续回调syncWithMaster函数，根据主服务发送的结果进行不同的操作。

注意⚠️：如果是全量复制的话，当主库对从库的 PSYNC 命令返回 FULLRESYNC 时，从库会在和主库的网络连接上注册 readSyncBulkPayload 回调函数，并将状态机置为 **REPL_STATE_TRANSFER**，表示开始进行实际的数据同步，比如主库把 RDB 文件传输给从库。

![img](https://gitee.com/JieMingLi/document-pics/raw/master/f6e25eb125f0d70694d92597dca3e197-20221014000136-30ld6cc.jpg)

syncWithMaster部分代码如下所示

```c
//读取PSYNC命令的返回结果
psync_result = slaveTryPartialResynchronization(fd,1);
//PSYNC结果还没有返回，先从syncWithMaster函数返回处理其他操作
if (psync_result == PSYNC_WAIT_REPLY) return;
//如果PSYNC结果是PSYNC_CONTINUE，从syncWithMaster函数返回，后续执行增量复制
if (psync_result == PSYNC_CONTINUE) {
       …
       return;
}
 
//如果执行全量复制的话，针对连接上的读事件，创建readSyncBulkPayload回调函数
if (aeCreateFileEvent(server.el,fd, AE_READABLE,readSyncBulkPayload,NULL)
            == AE_ERR)
    {
       …
    }
//将从库状态机置为REPL_STATE_TRANSFER
    server.repl_state = REPL_STATE_TRANSFER;
```

当主服务器把RDB文件生成成功发送给从服务器的时候，会触发从服务器的socket的ae_readable事件，然后执行readSyncBulkPayload回调函数。

readSyncBulkPayload函数清理从服务器的清理数据库，并且解析和载入RDB文件导入数据，最后再调用replicationCreateMasterClient方法，这个方法就把已连接的socket绑定到命令请求处理器（readQueryFromClient），用来监听后续的增量同步。

### 总体流程

![复制流程](https://gitee.com/JieMingLi/document-pics/raw/master/redis_replication.png)

### 参考

[redis复制源码解析](http://www.web-lovers.com/redis-source-replication.html)

[redis复制状态机](https://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/Redis%20%E6%BA%90%E7%A0%81%E5%89%96%E6%9E%90%E4%B8%8E%E5%AE%9E%E6%88%98/21%20%20%E4%B8%BB%E4%BB%8E%E5%A4%8D%E5%88%B6%EF%BC%9A%E5%9F%BA%E4%BA%8E%E7%8A%B6%E6%80%81%E6%9C%BA%E7%9A%84%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.md)

## 哨兵执行流程

**启动并初始化**Sentinel

每个Sentinel会创建2个连向主服务器的网络连接

1. 命令连接:用于向主服务器发送命令，并接收响应;
2. 订阅连接:用于订阅主服务器的—sentinel—:hello频道。

**获取主服务器信息**

Sentinel默认每10s一次，向被监控的主服务器发送info命令，获取主服务器和其下属从服务器的信息。

**获取从服务器信息**

当Sentinel发现主服务器有新的从服务器出现时，Sentinel还会向**从服务器**建立命令连接和订阅连接。 

在命令连接建立之后，Sentinel还是默认10s一次，向从服务器发送info命令，**并记录从服务器的信息。**

**向主服务器和从服务器发送消息**(以订阅的方式)

默认情况下，Sentinel每2s一次，向所有被监视的主服务器和从服务器所订阅的—sentinel—:hello频道

上发送消息，消息中会携带Sentinel自身的信息和主服务器的信息。

```c
PUBLISH _sentinel_:hello "< s_ip > < s_port >< s_runid >< s_epoch > < m_name > <m_ip >< m_port ><m_epoch>"
```

**接收来自主服务器和从服务器的频道信息**

当Sentinel与主服务器或者从服务器建立起订阅连接之后，Sentinel就会通过订阅连接，向服务器发送 以下命令:

```c
subscribe —sentinel—:hello
```

这也就是说，对于每个与 Sentinel 连接的服务器，Sentinel 既通过命今连接向服务器的_sentinel:hello频道发送信息，又通过订阅连接从服务器的 sentinel:hello
频道接收消息。

对于监视同一个服务器的多个 Sentinel来说，一个 Sentinel 发送的信息会被其他Sentinel 接收到，这些信息会被用于更新其他 Sentinel 对发送信息 Sentinel 的认知，也会被用于更新其他 Sentinel 对被监视服务器的认知（更新每个sentinel实例的sentinel字典）

当 Sentinel 通过频道信息发现一个新的 Sentinel时，它不仅会为新 Sentinel 在 sentinels字典中创建相应的实例结构，还会创建一个连向新 Sentinel的命令连接，而新 Sentinel 也同样会创建连向这个Sentinel 的命令连接，最终监视同一主服务器的多个 Sentinel 将形成相互连接的网络：Sentinel A 有连向 Sentinel B的命令连接，而 Sentinel B也有连向Sentinel A 的命令连接。

![image-20221102021907112](https://gitee.com/JieMingLi/document-pics/raw/master/image-20221102021907112.png)

创建命令连接的目的：

1，实现主观下线检测和客观下线检测。

2，sentinel选举出新的leader进行故障转移。

**检测主观下线状态**

Sentinel每秒一次向所有与它建立了命令连接的实例(主服务器、从服务器和其他Sentinel)发送PING命令

1， 实例在down-after-milliseconds毫秒内返回无效回复(除了+PONG、-LOADING、-MASTERDOWN外)

2， 实例在down-after-milliseconds毫秒内无回复(超时)

Sentinel就会认为该实例主观下线(**SDown**)

**检查客观下线状态**

当一个Sentinel将一个主服务器判断为主观下线后，Sentinel会向同时监控这个主服务器的所有其他Sentinel发送查询命令（通过命令连接）

```c
SENTINEL is-master-down-by-addr <ip> <port> <current_epoch> <runid>
```

其他Sentinel回复

```c
<down_state>< leader_runid >< leader_epoch >
```

判断它们是否也认为主服务器下线。如果达到Sentinel配置中的**quorum**数量的Sentinel实例都判断主服 务器为主观下线，则该主服务器就会被判定为客观下线(**ODown**)。

**选举** **Leader Sentinel**

当一个主服务器被判定为客观下线后，监视这个主服务器的所有Sentinel会通过选举算法(raft)，**选出一个Leader Sentinel去执行failover(故障转移)操作。**

**故障转移**

当选举出Leader Sentinel后，Leader Sentinel会对下线的主服务器执行故障转移操作，主要有三个步骤:

1. 它会将失效 Master 的其中一个 Slave 升级为新的 Master , 并让失效 Master 的其他 Slave 改为复 制新的 Master ;
2. 当客户端试图连接失效的 Master 时，集群也会向客户端返回**新 Master 的地址**，使得集群可以使 用现在的 Master 替换失效 Master 。
3. Master 和 Slave 服务器切换后， Master 的 redis.conf 、 Slave 的 redis.conf 和 sentinel.conf 的配置文件的内容都会发生相应的改变，即， Master 主服务器的 redis.conf配置文件中会多一行 replicaof 的配置， sentinel.conf 的监控目标会随之调换。

**如何选择主服务器**

哨兵leader根据以下规则从客观下线的主服务器的从服务器中选择出新的主服务器。

1. 过滤掉主观下线的节点
2. 选择slave-priority最高的节点，如果由则返回没有就继续选择
3. 选择出复制偏移量最大的系节点，因为复制偏移量越大则数据复制的越完整，如果由就返回了，没有就继续
4. 选择run_id最小的节点，因为run_id越小说明重启次数越少

## 哨兵启动流程

### 初始化

哨兵实例是属于运行在一种特殊模式下的 Redis server，哨兵实例的初始化入口函数也是 main（在 server.c 文件中）。

在main 函数在运行时，就会通过对运行参数的判断，来执行哨兵实例对应的运行逻辑。main函数中调用 **checkForSentinelMode 函数**，来判断当前运行的是否为哨兵实例

```c
server.sentinel_mode = checkForSentinelMode(argc,argv); // argc：启动命令字符串，argv：启动命令参数
```

checkForSentinelMode函数会根据以下两个条件判断当前是否运行了哨兵实例

- 条件一：执行的命令本身，也就是 argv[0]，是否为“redis-sentinel”。
- 条件二：执行的命令参数中，是否有“–sentinel”。

```c
int checkForSentinelMode(int argc, char **argv) {
    int j
    //第一个判断条件，判断执行命令本身是否为redis-sentinel
    if (strstr(argv[0],"redis-sentinel") != NULL) return 1;
    for (j = 1; j < argc; j++)
        //第二个判断条件，判断命令参数是否有"--sentienl"
        if (!strcmp(argv[j],"--sentinel")) return 1;
    return 0;
}
```

只要2个条件满足一个，server.sentinel_mode就会被设置为1，表明当前运行的是哨兵实例。在代码其他地方都会根据这值坐判断。

### 初始化配置项

哨兵实例运行时所用的配置项和 Redis 实例是有区别的，所以，main 函数会专门调用 initSentinelConfig 和 initSentinel 两个函数，来完成哨兵实例专门的配置项初始化

```c
if (server.sentinel_mode) {
   initSentinelConfig();
   initSentinel();
}
```

**initSentinelConfig 函数**的作用

1. 将当前 server 的端口号，改为哨兵实例专用的端口号（26379）
2. 把 server 的 protected_mode 设置为 0，即允许外部连接哨兵实例，而不是只能通过 127.0.0.1 本地连接 server。
3.  会初始化一个执行命令表，并保存在全局变量 server 的 commands 成员变量中。（这个命令表本身是一个哈希表，每个哈希项的键对应了一个命令的名称，而值对应了该命令实际的实现函数。）

**initSentinel 函数**的作用

1. initSentinel 函数会替换 server 能执行的命令表，哨兵实例是运行在特殊模式的 Redis server，它执行的命令和 Redis 实例也是有区别的，所以 initSentinel 函数会把 server.commands 对应的命令表清空

```c
  dictEmpty(server.commands,NULL);
    for (j = 0; j < sizeof(sentinelcmds)/sizeof(sentinelcmds[0]); j++) {
        …
        struct redisCommand *cmd = sentinelcmds+j;
        retval = dictAdd(server.commands, sdsnew(cmd->name), cmd);
        …
    }
```

哨兵实例执行的一些命令，其名称虽然和 Redis 实例命令表中的命令名称一样，但它们的实现函数是**针对哨兵实例专门实现的**。

```c
struct redisCommand sentinelcmds[] = {
    {"ping",pingCommand,1,"",0,NULL,0,0,0,0,0},
    {"sentinel",sentinelCommand,-2,"",0,NULL,0,0,0,0,0},
    …
    {"publish",sentinelPublishCommand,3,"",0,NULL,0,0,0,0,0},
    {"info",sentinelInfoCommand,-1,"",0,NULL,0,0,0,0,0},
    {"role",sentinelRoleCommand,1,"l",0,NULL,0,0,0,0,0},
    …
};
```

2.  initSentinel 函数在替换了命令表后，紧接着它会开始初始化哨兵实例用到的各种属性信息。

   哨兵实例定义了 **sentinelState 结构体**，保存哨兵实例的 ID、用于故障切换的当前纪元、监听的主节点、正在执行的脚本数量，以及与其他哨兵实例发送的 IP 和端口号等信息

   ```c
   struct sentinelState {
       char myid[CONFIG_RUN_ID_SIZE+1];  //哨兵实例ID
       uint64_t current_epoch;         //当前纪元
       dict *masters;      //监听的主节点的哈希表，比如，它会为监听的主节点创建一个哈希表，哈希项的键记录了主节点的名称，而值记录了对应的数据结构指针。
       int tilt;           //是否处于TILT模式
       int running_scripts;    //运行的脚本个数
       mstime_t tilt_start_time;  //tilt模式的起始时间
       mstime_t previous_time;     //上一次执行时间处理函数的时间
       list *scripts_queue;         //用于保存脚本的队列
       char *announce_ip;  //向其他哨兵实例发送的IP信息
       int announce_port;  //向其他哨兵实例发送的端口号
       …
   } sentinel;
   
   
   
   ```

   流程图如下所示

   ![img](https://gitee.com/JieMingLi/document-pics/raw/master/6e692yy58da223d98b2d4d390c8e97ac-20221014000200-kbhzhzq.jpg)

3. main 函数还会调用 initServer 函数完成 server 本身的初始化操作，这部分哨兵实例也是会执行的。然后，main 函数就会调用 **sentinelIsRunning 函数**（在 sentinel.c 文件中**）启动哨兵实例。**

### 启动哨兵实例

sentinelIsRunning函数执行的逻辑

1. 确认哨兵实例的配置文件存在并且可以正常写入
2. 它会检查哨兵实例是否设置了 ID。如果没有设置 ID 的话，sentinelIsRunning 函数就会为哨兵实例随机生成一个 ID。
3. 调用 sentinelGenerateInitialMonitorEvents 函数给每个被监听的主节点发送事件信息

![img](https://gitee.com/JieMingLi/document-pics/raw/master/6a0241e822b02db8d907f7fdda48cebd-20221014000200-klz396c.jpg)

#### 如何获取主节点

在**initSentinel 函数**中，会初始化哨兵实例的数据结构 sentinel.masters（哈希表），sentinel.masters保存主节点。

每个主节点会使用 **sentinelRedisInstance 结构**来保存，在sentinelRedisInstance 结构中，就包含了被监听主节点的地址信息。这个地址信息是由 sentienlAddr 结构体保存的，其中包括了节点的 IP 和端口号。

```c
typedef struct sentinelAddr {
    char *ip;
    int port;
} sentinelAddr;
```

**sentinelRedisInstance 结构**还保存一些和主节点、故障切换相关的其他信息，比如主节点名称、ID、监听同一个主节点的其他哨兵实例、主节点的从节点、主节点主观下线和客观下线的时长

```c
typedef struct sentinelRedisInstance {
    int flags;      //实例类型、状态的标记, flags 设置为 SRI_MASTER、SRI_SLAVE 或 SRI_SENTINEL 这三种宏定义（在 sentinel.c 文件中）时，就分别表示当前实例是主节点、从节点或其他哨兵。
    char *name;     //实例名称
    char *runid;    //实例ID
    uint64_t config_epoch;  //配置的纪元
    sentinelAddr *addr; //实例地址信息
    ...
    mstime_t s_down_since_time; //主观下线的时长
    mstime_t o_down_since_time; //客观下线的时长
    ...
    dict *sentinels;    //监听同一个主节点的其他哨兵实例
   dict *slaves;   //主节点的从节点
   ...
}
```

> sentinelRedisInstance 是一个通用的结构体，**它不仅可以表示主节点，也可以表示从节点或者其他的哨兵实例**。

在执行sentinelIsRunning中如果需要获取主节点的信息，只需要从 sentinel.masters 结构中获取主节点对应的 sentinelRedisInstance 实例就可以。

#### 发送消息

获取到主节点后，就可以通过sentinelGenerateInitialMonitorEvents发送消息

```c
void sentinelGenerateInitialMonitorEvents(void) {
    dictIterator *di;
    dictEntry *de;

    di = dictGetIterator(sentinel.masters);//获取masters的迭代器
    while((de = dictNext(di)) != NULL) { //获取被监听的主节点
        sentinelRedisInstance *ri = dictGetVal(de);
        sentinelEvent(LL_WARNING,"+monitor",ri,"%@ quorum %d",ri->quorum);   //发送+monitor事件
    }
    dictReleaseIterator(di);
}
```

通过调用 sentinelEvent 函数来实际发送事件信息，sentinelEvent如下

```c
//  level 表示当前的日志级别，type 表示发送事件信息所用的订阅频道，ri 表示对应交互的主节点，fmt 则表示发送的消息内容。
void sentinelEvent(int level, char *type, sentinelRedisInstance *ri, const char *fmt, ...) 
```

sentinelEvent 函数会判断监听实例的类型是否为主节点。然后如果是主节点，sentinelEvent 函数会把监听实例的名称、IP 和端口号加入到待发送的消息中

```c
...
//如果传递消息以"%"和"@"开头，就判断实例是否为主节点
if (fmt[0] == '%' && fmt[1] == '@') {
   //判断实例的flags标签是否为SRI_MASTER，如果是，就表明实例是主节点
   sentinelRedisInstance *master = (ri->flags & SRI_MASTER) ?
                                         NULL : ri->master;
   //如果当前实例是主节点，根据实例的名称、IP地址、端口号等信息调用snprintf生成传递的消息msg
   if (master) {
      snprintf(msg, sizeof(msg), "%s %s %s %d @ %s %s %d", sentinelRedisInstanceTypeStr(ri), ri->name, ri->addr->ip, ri->addr->port,
                master->name, master->addr->ip, master->addr->port);
  }
        ...
}
...
```

生成完消息后，sentinelEvent 函数会调用 pubsubPublishMessage 函数（在 pubsub.c 文件中），将消息发送到对应的频道中。

```c
 if (level != LL_DEBUG) {
        channel = createStringObject(type,strlen(type));
        payload = createStringObject(msg,strlen(msg));
        pubsubPublishMessage(channel,payload);
        ...
  }
```

![img](https://gitee.com/JieMingLi/document-pics/raw/master/8f50ed685a3c66b34a2d1b2697b6e96a-20221014000200-vnsw674.jpg)

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

