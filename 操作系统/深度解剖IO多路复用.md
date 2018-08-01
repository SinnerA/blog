---
title: 深度解剖IO多路复用
date: 2018-04-26 02:09:00
tags: OS Network Basics
---

[TOC]

## 概述

IO多路复用能够同时监听多个文件描述符，需要用到IO多路复用的地方：

- TCP服务器要同时处理监听socket和连接socket，这是IO多路复用用的最多的场合。
- 服务器要同时监听多个端口，或者处理多种服务，如client处理多个socket，client要同时处理用户输入和网络连接

Linux实现多路复用的系统调用主要有select、poll和epoll，可以监视多个描述符，一旦某个描述符就绪（一般是读就绪或者写就绪），能够通知程序进行相应的读写操作

<!-- more -->

## 网络内核结构

### sock和socket

对于socket，我们都很熟悉，就是网络编程中的常见的套接字。那么，sock又是什么呢？其实，在内核源码中，socket和sock都是一个struct，包含了网络相关的一些信息（sock侧重网络部分，socket侧重文件部分，因为Unix中万事皆文件）

![image-20180425001111771](/Users/sinnera/sinnera.github.io/source/illustrations/image-20180425001111771.png)

在介绍select、poll、epoll前，有必要先了解下Linux内核（>=2.6）的事件等待和唤醒机制，这是IO多路复用机制存在的本质。Linux通过socket的sleep_list来管理所有等待该socket某个事件的process，事件发生后，通过wakeup机制来异步唤醒sleep_list上的每个节点，顺序遍历sleep_list的每个节点，并调用各自的callback函数，如果遇到节点是_排他节点_，就终止遍历。

**睡眠等待**

1. select、poll、epoll_wait陷入内核，判断监控的socket是否有关心的事件发生了，如果没，则为当前process构建一个wait_entry节点，然后插入到监控socket的sleep_list
2. 进入循环的schedule直到关心的事件发生了
3. 关心的事件发生后，将当前process的wait_entry节点从socket的sleep_list中删除

**唤醒**

1. socket的事件发生了，然后socket顺序遍历其sleep_list，依次调用每个wait_entry节点的callback函数
2. 直到完成队列的遍历或遇到某个wait_entry节点是排他的才停止
3. 一般情况下callback包含两个逻辑：
   - wait_entry自定义的私有逻辑
   - 唤醒的公共逻辑，主要用于将该wait_entry的process放入CPU的就绪队列，让CPU随后可以调度其执行

socket中存在一个系统函数函数Poll，一般由设备驱动提供，用于收集socket发生的事件，对于可读事件来说，简单伪码如下：

```c
poll()
{
    //其他逻辑
    if not_empty(recieve_queque)
    {
        sk_event |= POLL_IN；
    }
   //其他逻辑
}
```

## IO多路复用

在一个高性能的网络服务上，大多情况下一个服务进程(线程)process需要同时处理多个socket，我们需要公平对待所有socket，对于read而言，那个socket有数据可读，process就去读取该socket的数据来处理。于是对于read，一个朴素的需求就是关心的N个socket是否有数据”可读”，也就是我们期待”可读”事件的通知。我们应该block在等待事件的发生上，这个事件简单点就是”关心的N个socket中一个或多个socket有数据可读了”，当block解除的时候，就意味着，我们一定可以找到一个或多个socket上有可读的数据。另一方面，根据上面的socket wakeup callback机制，我们不知道什么时候，哪个socket会有读事件发生，于是，process需要同时插入到这N个socket的sleep_list上等待任意一个socket可读事件发生而被唤醒，当时process被唤醒的时候，其callback里面应该有个逻辑去检查具体那些socket可读了。

## select

```c
#include<sys/select.h>
/* 
    nfds: 被监听的文件描述符的总数，通常设置为最大值+1
    readfds: 指向可读事件对应的文件描述符集合(int数组)，调用select时，传入自己想要监听的文件描述符，
             select调用返回时，内核将修改它们来通知应用程序哪些文件描述符已经就绪。也就是说，它同时充当
             了input param和output param
    writefds: 同readfds
    exceptfds: 同readfds
    timeout: 超时时间
 */
int select(int nfds, fd_set* readfds, fd_set* writefds, fd_set* exceptfds, struct timeval* timeout);
```

调用select的时：

1. select会将需要监控的readfds集合拷贝到内核空间（假设监控的仅仅是socket可读）
2. 然后遍历自己监控的socket sk，挨个调用sk的Poll函数以便检查该sk是否有可读事件，遍历完所有的sk后，如果没有任何一个sk可读，那么select会调用schedule_timeout进入schedule循环，使得process进入睡眠
3. 如果在timeout时间内某个sk上有数据可读了，或者等待timeout了，则调用select的process会被唤醒，接下来select就是遍历监控的sk集合，挨个收集可读事件并返回给用户

简单来说，就是在指定时间内，**轮询所有文件描述符**，测试是否就绪。

通过上面的select过程，相信大家都意识到，select存在两个问题：

1. 被监控的fds需要从用户空间拷贝到内核空间。为了减少数据拷贝带来的性能损坏，内核对被监控的fds集合大小做了限制，并且这个是通过宏控制的，大小不可改变(限制为1024)。
2. 被监控的fds集合中，只要有一个有数据可读，整个socket集合就会被遍历一次用于收集可读事件

到这里，我们有三个问题需要解决：

1. 被监控的fds集合限制为1024，1024太小了，我们希望能够有个比较大的可监控fds集合
2. fds集合需要从用户空间拷贝到内核空间的问题，我们希望不需要拷贝
3. 当被监控的fds中某些有数据可读的时候，我们希望通知更加精细一点，就是我们希望能够从通知中得到有可读事件的fds列表，而不是需要遍历整个fds来收集。

## poll

```c
#include <poll.h>
/*
   fds: pollfd结构类型的数组（见下），它指定了所有我们感兴趣的文件描述符上发生的可读，可写和异常事件。
   nfds: 被监听的文件描述符的总数，通常设置为最大值+1
   timeout: 超时时间

   struct pollfd {
      int fd;
      short events;  //注册的事件
      short revents; //实际发生的事件，由内核填充
   }
*/

int poll(struct pollfd* fds, nfds_t nfds, int timeout);
```

poll和select非常相似，poll改变了fds集合的描述方式，使用了pollfd结构而不是select的fd_set结构，使得poll支持的fds集合限制远大于select的1024。

因此，poll只是解决了fds集合大小1024限制问题，并没有解决其它2个问题。

## epoll

```c
#include <sys/epoll.h>

/*
    epoll与select和poll不同，epoll将用户关心的文件描述符上的事件放在内核里的一个事件表中，从而无须像
    select和poll那样，每次调用时都要重复传入文件描述符集或事件集。因此，epoll需要一个文件描述符来唯一标
    识内核中的这个事件表，该文件描述符由epoll_create来创建(size表示事件表的大小)
*/
int epoll_create(int size);

/*
    epfd: 内核事件表的文件描述符
    fd: 要操作的文件描述符
    op: 操作类型，EPOLL_CTL_ADD，EPOLL_CTL_MOD，EPOLL_CTL_DEL
    event: 指定事件
*/
int epoll_ctl(int epfd, int op, int fd, struct epoll_event* event);

/*
    epfd: 内核事件表的文件描述符
    event: 指定事件
    maxevents: 最多监听事件数量
    timeout: 超时时间
*/
int epoll_wait(int epfd, struct epoll_event* events, int maxevents, int timeout);
```

### 解决问题

epoll如何解决这3个问题的呢？计算机领域中，有两种经典的解决思路：

1. 加中间层（比如缓存思想，tcmalloc，goroutine调度器等等，都是加一个中间层提高效率）
2. 变全局为局部（mysql中表锁改成行锁，goroutine中的P）

epoll正是运用了这两个思路，解决了这些问题：

**fds拷贝**

select和poll中每次都需要准备全量的fds，拷贝到内核。其实，fds变化的频率较低，每次需要关心的fds相差不大。因此，我们可以在一开始把全量的拷贝到内核，之后只需要关系有变化的，新增、删除或者修改，当然新增的时候不可避免也需要拷贝到内核，相比每次都拷贝全量的，大大提高了效率。这就是全局变局部的思想。

epoll_create用于创建结构体eventpoll，关心的所有fds都在这个结构体上。每次更新fds时，使用epoll_ctl进行增删改查，同时加入了**回调函数**，这里引入了**红黑树**进行管理每个epitem，可以提高查询效率，eventpoll存放该红黑树的根节点。红黑树的维护也需要一定代价，因此epoll适合fds变化较少的场景，如果变动频繁，效率不一定比select/poll优势大。

**每次都需要遍历就绪的fds**

select和poll中，每次有就绪的fds，就需要遍历全部的fds，收集准备好的fds。能不能加一个队列，用于自动收集就绪好的fds呢？epoll就是这么干的，这就是中间层思想。

epoll准备了额外的**就绪队列**，当各个socket准备好时，就通过调用epoll_ctl操作中加入的**回调函数**，来将自己添加到就绪队列中。epoll_wait就是遍历就绪队列，而就绪队列中的都是准备好的，因此不存在浪费效率的问题。

**总结**

epoll的逻辑：

1. 通过epoll_create构建了一个文件结构，后续的所有操作都是在这个文件基础上。因此也就没有select中来回在用户空间和内核空间之间拷贝。
2. epoll_ctl在插入事件时，也为该事件添加了回调函数，当该事件发生时，会调用回调函数，将该事件插入到就绪队列中。因此也就避免了select的全部遍历事件。
3. epoll_wait只是返回就绪队列中的事件。

### 数据结构

![image-20180428010807907](/Users/sinnera/sinnera.github.io/source/illustrations/image-20180428010807907.png)

### ET和LT

- LT（水平触发）：可以触发多次
- ET（边沿触发）：只能触发一次
- ET模式下，一个socket上的事件还是可能触发多次，在并发程序中有问题，我们希望一个socket在任一时刻都只能被一个线程处理，可以使用epoll的EPOLLONESHOT事件
- 对于注册了EPOLLONESHOT事件的文件描述符，最多触发其上注册的一个事件，且只触发一次。这样就保证了任一时刻都只被一个线程处理，注意，某个线程每次处理完之后，需要重置EPOLLONESHOT事件，以便之后的线程能够继续处理。


#### 概念

**Edge Triggered (ET) 边沿触发**

socket的接收缓冲（发送）区状态变化时触发读（写）事件，即空的接收缓冲区刚接收到数据时触发读（写）事件

仅在缓冲区状态变化时触发事件，比如数据缓冲去从无到有的时候(不可读-可读)

>类似于最近在做的IM的对话列表，只有新对话产生时（从无到有），才更新对话列表，每次更新消耗性能

**Level Triggered (LT) 水平触发**

只要socket接收（发送）缓冲区不为空，有数据可读（写），则读（写）事件一直触发

epoll上的就绪队列什么时候将事件移除？其实，移除时机就是ET和LT实现的本质。epoll_wait遍历完就绪队列后，如果是LT模式，会将事件重新放入就绪队列，而ET模式则直接移除。

select和poll是通过调用系统函数Poll来收集事件的，因此不能进行移除操作，只要缓存区不为空，该事件将一直存在，可以看作是LT模式。

**注意**

1. 使用ET时，需要一直读或写，直到EAGAIN或EWOULDBLOCK（对于非阻塞IO，表示数据已全部读取完毕）
2. ET模式下，处理失误容易造成饿死现象。假设同时有三个请求到达，epoll_wait返回listen_fd可读，这个时候，如果仅仅accept一次拿走一个请求去处理，那么就会留下两个请求，如果这个时候一直没有新的请求到达，那么再次调用epoll_wait是不会通知listen_fd可读的，于是epoll_wait只能睡眠到超时才返回，遗留下来的两个请求一直得不到处理，处于饿死状态
3. ET本身并不会造成饥饿，由于事件只通知一次，开发者一不小心就容易遗漏了待处理的数据，像是饥饿，实质是bug
4. LT不容易遗漏事件、不易产生bug

**总结**

ET理论上可以比LT少带来一些系统调用，所以更省一些。在绝大多少情况下，ET模式并不会比LT模式更为高效，LT是足够的。同时，ET模式带来了不好理解的语意，这样容易造成编程上面的复杂逻辑。因此，通常情况下建议还是采用LT模式。

### 总结

1. 当用户场景下有大量短连接时，epoll对于红黑树频繁的插入和删除操作也会有性能退化
2. 当用户场景下处理大量fd但仅有少量活跃连接时，epoll则相对更高效
3. 对于ET和LT模式的使用，LT对于未处理的事件不会丢掉，使用更简单，ET下若用户漏掉处理，就绪事件则会丢失掉，编程处理上有一定复杂性

### 常见问题

Q：为什么ET要设置为非阻塞？

A：

> **这是从实际应用考虑的，阻塞跟ET搭配起来会有问题的**
>
> 只说读的情况：
>
> **ET只有来数据时才会提醒**，因此读数据时，你**必须一次性地把缓冲区的数据读完**（如果不读完，则缓冲区还残留着未读数据，然后对端又不继续发数据的话，`ET`是不会再通知你，也就是说你将永远读不到残留数据）
>
> **为了一次性把缓冲区数据读完，你必须要写一个while循环来read，直到缓冲区里面数据被读完，**
>
> **如果设成阻塞的话**，你的程序就无法知道数据什么时候被读完，因为当数据读完时，会**卡在while里面的read**，一直在等数据，永远退不出`while`
>
> **如果设成非阻塞**，当数据被读完，read就会返回，然后将`errno`设成`EAGAIN`并退出`while`，**这才是正确的逻辑**

## 比较

![WX20171007-210244@2x](/Users/sinnera/sinnera.github.io/source/illustrations/WX20171007-210244@2x.png)

> 如果没有大量的idle -connection或者dead-connection，epoll的效率并不会比select/poll高很多，但是当遇到大量的idle- connection，就会发现epoll的效率大大高于select/poll。

## 参考

[大话 Select、Poll、Epoll](https://cloud.tencent.com/developer/article/1005481)

[epoll源码学习](http://wangzhilong.github.io/2015/09/30/epoll/)

[Linux内核中网络数据包的接收-第二部分 select/poll/epoll](http://blog.51cto.com/dog250/1735579)

[The Network Layer (2.4)](http://www.oocities.org/marco_corvi/games/lkpe/socket/network.htm)
