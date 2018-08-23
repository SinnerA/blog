---
title: IO多路复用
date: 2017-10-07 17:17:08
tags: OS Basics
---

[TOC]

## 概述

IO多路复用能够同时监听多个文件描述符，需要用到IO多路复用的地方：

- TCP服务器要同时处理监听socket和连接socket，这是IO多路复用用的最多的场合。
- 服务器要同时监听多个端口，或者处理多种服务，如client处理多个socket，client要同时处理用户输入和网络连接

Linux实现多路复用的系统调用主要有select、poll和epoll，可以监视多个描述符，一旦某个描述符就绪（一般是读就绪或者写就绪），能够通知程序进行相应的读写操作

<!-- more -->

**文件描述符就绪条件**

- socket可读
  - 有新的连接请求
  - 对端关闭（读操作将返回0）
  - socket内核接收缓冲区中的字节数大于或等于其低水平位标记SO_RCVLOWAT（读操作返回大于0）
  - 有未处理的错误
- socket可写
  - 建立连接之后
  - socket写操作被关闭
  - socket内核发送缓冲区中的字节数大于或等于其低水平标记位SO_SNDLOWAT（写操作返回大于0）

## 网络内核结构

### sock和socket

对于socket，我们都很熟悉，就是网络编程中的常见的套接字。那么，sock又是什么呢？其实，在内核源码中，socket和sock都是一个struct，包含了网络相关的一些信息（sock侧重网络部分，socket侧重文件部分，因为Unix中万事皆文件）

![image-20180425001111771](https://github.com/SinnerA/blog/tree/master/illustrations/image-20180425001111771.png)

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

## IO多路复用

在一个高性能的网络服务上，大多情况下一个服务进程(线程)process需要同时处理多个socket，我们需要公平对待所有socket，对于read而言，那个socket有数据可读，process就去读取该socket的数据来处理。于是对于read，一个朴素的需求就是关心的N个socket是否有数据”可读”，也就是我们期待”可读”事件的通知。我们应该block在等待事件的发生上，这个事件简单点就是”关心的N个socket中一个或多个socket有数据可读了”，当block解除的时候，就意味着，我们一定可以找到一个或多个socket上有可读的数据。另一方面，根据上面的socket wakeup callback机制，我们不知道什么时候，哪个socket会有读事件发生，于是，process需要同时插入到这N个socket的sleep_list上等待任意一个socket可读事件发生而被唤醒，当时process被唤醒的时候，其callback里面应该有个逻辑去检查具体那些socket可读了。

socket中存在一个poll函数，用于收集socket发生的事件，对于可读事件来说，简单伪码如下：

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

## select

### 原型

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

//example
while(1) {
    FD_SET(connfd, &read_fds)     //每次都要重置，因为事件发生之后，文件描述符会被内核修改
    FD_SET(connfd, &exception_fds)
    ret = select(connfd + 1, &read_fds, NULL, &exception_fds, NULL);
    if (ret > 0){
       if(FD_ISSET(connfd, &read_fds)) {
            ret = recv(connfd, buf, sizeof(buf)-1, 0);
       } else if (FD_ISSET(connfd, *exception_fds)) {
         	ret = recv(connfd, buf, sizeof(buf)-1, MSG_OOB)
       }
    }
}
```

### 步骤

当用户process调用select的时候，select会将需要监控的readfds集合拷贝到内核空间（假设监控的仅仅是socket可读），然后遍历自己监控的socket sk，挨个调用sk的poll函数以便检查该sk是否有可读事件，遍历完所有的sk后，如果没有任何一个sk可读，那么select会调用schedule_timeout进入schedule循环，使得process进入睡眠。如果在timeout时间内某个sk上有数据可读了，或者等待timeout了，则调用select的process会被唤醒，接下来select就是遍历监控的sk集合，挨个收集可读事件并返回给用户了，相应的伪码如下：

```c
for (sk in readfds)
{
    sk_event.evt = sk.poll();
    sk_event.sk = sk;
    ret_event_for_process;
}
```

在指定时间内，**轮询所有文件描述符**，测试是否就绪

### 缺点

1. 单个进程能够监视的文件描述符的数量存在最大限制，通常是1024，当然可以更改数量，但由于select采用轮询的方式扫描文件描述符，文件描述符数量越多，性能越差；(在linux内核头文件中，有这样的定义：#define __FD_SETSIZE    1024)
2. 内核 / 用户空间内存拷贝问题，select需要复制大量的句柄数据结构，产生巨大的开销；
3. select返回的是含有整个句柄的数组，应用程序需要遍历整个数组才能发现哪些句柄发生了事件；
4. select的触发方式是水平触发，应用程序如果没有完成对一个已经就绪的文件描述符进行IO操作，那么之后每次select调用还是会将这些文件描述符通知进程。

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

与select类似

相比select模型，poll使用链表保存文件描述符，因此没有了监视文件数量的限制，但其他三个缺点依然存在。

拿select模型为例，假设我们的服务器需要支持100万的并发连接，则在__FD_SETSIZE 为1024的情况下，则我们至少需要开辟1k个进程才能实现100万的并发连接。除了进程间上下文切换的时间消耗外，从内核/用户空间大量的无脑内存拷贝、数组轮询等，是系统难以承受的。因此，基于select模型的服务器程序，要达到10万级别的并发访问，是一个很难完成的任务。

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

Linux特有的IO复用函数

内核中维护了一个eventList，有一个文件描述符，来唯一标识内核中的这个eventList

一组函数：epoll_create, epoll_ctl, epoll_wait

两种模式：

- LT（水平触发）：可以触发多次
- ET（边沿触发）：只能触发一次
- ET模式下，一个socket上的事件还是可能触发多次，在并发程序中有问题，我们希望一个socket在任一时刻都只能被一个线程处理，可以使用epoll的EPOLLONESHOT事件
- 对于注册了EPOLLONESHOT事件的文件描述符，最多触发其上注册的一个事件，且只触发一次。这样就保证了任一时刻都只被一个线程处理，注意，某个线程每次处理完之后，需要重置EPOLLONESHOT事件，以便之后的线程能够继续处理。



由于epoll的实现机制与select/poll机制完全不同，上面所说的 select的缺点在epoll上不复存在。

设想一下如下场景：有100万个客户端同时与一个服务器进程保持着TCP连接。而每一时刻，通常只有几百上千个TCP连接是活跃的(事实上大部分场景都是这种情况)。如何实现这样的高并发？

在select/poll时代，服务器进程每次都把这100万个连接告诉操作系统(从用户态复制句柄数据结构到内核态)，让操作系统内核去查询这些套接字上是否有事件发生，轮询完后，再将句柄数据复制到用户态，让服务器应用程序轮询处理已发生的网络事件，这一过程资源消耗较大，因此，select/poll一般只能处理几千的并发连接。

epoll的设计和实现与select完全不同。epoll通过在Linux内核中申请一个简易的文件系统(文件系统一般用什么数据结构实现？B+树)。把原先的select/poll调用分成了3个部分：

1）调用epoll_create()建立一个epoll对象(在epoll文件系统中为这个句柄对象分配资源)

2）调用epoll_ctl向epoll对象中添加这100万个连接的套接字

3）调用epoll_wait收集发生的事件的连接

如此一来，要实现上面说是的场景，只需要在进程启动时建立一个epoll对象，然后在需要的时候向这个epoll对象中添加或者删除连接。同时，epoll_wait的效率也非常高，因为调用epoll_wait时，并没有一股脑的向操作系统复制这100万个连接的句柄数据，内核也不需要去遍历全部的连接。

**下面来看看Linux内核具体的epoll机制实现思路。**

当某一进程调用epoll_create方法时，Linux内核会创建一个eventpoll结构体，这个结构体中有两个成员与epoll的使用方式密切相关。eventpoll结构体如下所示：

```
struct eventpoll{
    ....
    /*红黑树的根节点，这颗树中存储着所有添加到epoll中的需要监控的事件*/
    struct rb_root  rbr;
    /*双链表中则存放着将要通过epoll_wait返回给用户的满足条件的事件*/
    struct list_head rdlist;
    ....
};
```

每一个epoll对象都有一个独立的eventpoll结构体，用于存放通过epoll_ctl方法向epoll对象中添加进来的事件。这些事件都会挂载在红黑树中，如此，重复添加的事件就可以通过红黑树而高效的识别出来(红黑树的插入时间效率是log(n)，其中n为树的高度)。

而所有添加到epoll中的事件都会与设备(网卡)驱动程序建立回调关系，也就是说，当相应的事件发生时会调用这个回调方法。这个回调方法在内核中叫ep_poll_callback,它会将发生的事件添加到rdlist双链表中。

在epoll中，对于每一个事件，都会建立一个epitem结构体，如下所示：

```
struct epitem{
    struct rb_node  rbn;//红黑树节点
    struct list_head    rdllink;//双向链表节点
    struct epoll_filefd  ffd;  //事件句柄信息
    struct eventpoll *ep;    //指向其所属的eventpoll对象
    struct epoll_event event; //期待发生的事件类型
}
```

当调用epoll_wait检查是否有事件发生时，只需要检查eventpoll对象中的rdlist双链表中是否有epitem元素即可。如果rdlist不为空，则把发生的事件复制到用户态，同时将事件数量返回给用户。

![epoll.jpg](http://static.open-open.com/lib/uploadImg/20140911/20140911103834_133.jpg)

epoll数据结构示意图

**从上面的讲解可知：通过红黑树和双链表数据结构，并结合回调机制，造就了epoll的高效。**

OK，讲解完了Epoll的机理，我们便能很容易掌握epoll的用法了。一句话描述就是：三步曲。

第一步：epoll_create()系统调用。此调用返回一个句柄，之后所有的使用都依靠这个句柄来标识。

第二步：epoll_ctl()系统调用。通过此调用向epoll对象中添加、删除、修改感兴趣的事件，返回0标识成功，返回-1表示失败。

第三部：epoll_wait()系统调用。通过此调用收集收集在epoll监控中已经发生的事件。



**三种机制的比较**

![WX20171007-210244@2x](/Users/sinnera/Downloads/WX20171007-210244@2x.jpg)

**总结**

在 select/poll中，进程只有在调用一定的方法后，内核才对所有监视的文件描述符进行扫描，而**epoll事先通过epoll_ctl()来注册一个文件描述符，一旦基于某个文件描述符就绪时，内核会采用类似callback的回调机制，迅速激活这个文件描述符，当进程调用epoll_wait() 时便得到通知**。(`此处去掉了遍历文件描述符，而是通过监听回调的的机制`。这正是epoll的魅力所在。)

**epoll的优点主要是一下几个方面：**

1. 监视的描述符数量不受限制，它所支持的FD上限是最大可以打开文件的数目，这个数字一般远大于2048,举个例子,在1GB内存的机器上大约是10万左 右，具体数目可以cat /proc/sys/fs/file-max察看,一般来说这个数目和系统内存关系很大。select的最大缺点就是进程打开的fd是有数量限制的。这对 于连接数量比较大的服务器来说根本不能满足。虽然也可以选择多进程的解决方案( Apache就是这样实现的)，不过虽然linux上面创建进程的代价比较小，但仍旧是不可忽视的，加上进程间数据同步远比不上线程间同步的高效，所以也不是一种完美的方案。
2. IO的效率不会随着监视fd的数量的增长而下降。epoll不同于select和poll轮询的方式，而是通过每个fd定义的回调函数来实现的。只有就绪的fd才会执行回调函数。

> 如果没有大量的idle -connection或者dead-connection，epoll的效率并不会比select/poll高很多，但是当遇到大量的idle- connection，就会发现epoll的效率大大高于select/poll。



## 参考

[大话 Select、Poll、Epoll](https://cloud.tencent.com/developer/article/1005481)

[Linux内核中网络数据包的接收-第二部分 select/poll/epoll](http://blog.51cto.com/dog250/1735579)

[The Network Layer (2.4)](http://www.oocities.org/marco_corvi/games/lkpe/socket/network.htm)

[socket和sock的一些分析](http://abcdxyzk.github.io/blog/2015/06/12/kernel-net-sock-socket/)

[Linux 网络栈剖析](https://www.ibm.com/developerworks/cn/linux/l-linux-networking-stack/)
