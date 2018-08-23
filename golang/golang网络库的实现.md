---
title: golang网络库的实现
date: 2018-04-26 23:07:00
tags: Golang 源码
---

[TOC]

## 概述

目前主流web server比如：Nginx、Redis、memcached等，它们都是基于”事件驱动 + 回调函数”的方式实现，也就是采用epoll等作为网络收发数据包的核心驱动。

不过Go的设计者似乎认为I/O多路复用的这种通过回调机制**割裂控制流**的方式依旧复杂，且有悖于“一般逻辑”设计，为此Go语言将该“复杂性”隐藏在Runtime中了：Go开发者无需关注socket是否是 non-block的，也无需亲自注册文件描述符的回调，只需在每个连接对应的goroutine中以**同步**的方式对待socket处理即可，这可以说大大降低了开发人员的心智负担。

一个典型的Go server端程序大致如下：

```go
import (
	"log"
	"net"
)

func main() {
	ln, err := net.Listen("tcp", ":8080")
	if err != nil {
        	log.Println(err)
        	return
	}
	for {
        	conn, err := ln.Accept()
        	if err != nil {
            	log.Println(err)
            	continue
        	}

        	go echoFunc(conn)
	}
}

func echoFunc(c net.Conn) {
	buf := make([]byte, 1024)

	for {
        n, err := c.Read(buf) //注意：同步阻塞操作，没有数据一直等待
        if err != nil {
            log.Println(err)
            return
        }

        c.Write(buf[:n])
	}
}
```

每收到一个新的连接，就创建一个goroutine去服务这个连接，因此所有的业务逻辑都可以同步、顺序的编写到echoFunc函数中，再也不用去关心网络IO是否会阻塞的问题。不管业务多复杂，Go语言的并发服务器的编程模型都是长这个样子。

看下传统的epoll伪代码

```go
func main(){
	listen()
    addfd() //set non_block
    for {
        ret = epoll_wait()
        if ret > 0 {
            echoFunc()
        }
    }
}

func echoFunc(){
    for {
        n, err := recv(buf)
        if n < 0 && err == EAGAIN { //注意：同步非阻塞操作，没有数据直接返回EAGAIN
            log.Println(err)
            return
        }

        c.send(buf[:n])
    }
}
```

两者对比下，看起来类似。但是go是阻塞等待的，传统的是非阻塞直接返回的，这是他们两本质区别。另外，go的使用也非常简单，不需要像传统epoll一样，需要关心：设置成非阻塞IO，读取时候的EAGAIN，ET还是LT等等

## 实现

Go提供的网络接口，在用户层是阻塞的，这样最符合人们的编程习惯。在runtime层面，是用epoll/kqueue实现的非阻塞io，为性能提供了保障。

可以大胆的推测一下，golang网络IO读写操作的实现应该是：当在一个fd上读写遇到EAGAIN错误的时候，将goroutine给park住，直到这个fd上再此发生了读写事件后，再将此goroutine给ready激活重新运行。事实上的实现大概也是这个样子的。

### 三个层次

从最原始的epoll系统调用，到提供给用户的网络库函数，可以分成三个封装层次。这三个层次自底向上分别是：

1. 依赖于系统的api封装
2. 平台独立的runtime封装
3. 提供给用户的库的封装

#### epoll

runtime/netpoll_epoll.c文件中实现了Linux上的epoll。只有4个函数，分别是runtime·netpollinit、runtime·netpollopen、runtime·netpollclose和runtime·netpoll。init函数就是创建epoll对象，open函数就是添加一个fd到epoll中，close函数就是从epoll删除一个fd，netpoll函数就是从epoll wait得到所有发生事件的fd，并将每个fd对应的goroutine(用户态线程)通过链表返回。

#### runtime

runtime/netpoll.goc源文件就是整个事件驱动抽象层的实现，抽象层的核心数据结构是：

```go
struct PollDesc
{
	PollDesc* link;	// in pollcache, protected by pollcache.Lock
	Lock;		// protectes the following fields
	uintptr	fd;
	bool	closing;
	uintptr	seq;	// protects from stale timers and ready notifications
	G*	rg;	// G waiting for read or READY (binary semaphore)
	Timer	rt;	// read deadline timer (set if rt.fv != nil)
	int64	rd;	// read deadline
	G*	wg;	// the same for writes
	Timer	wt;
	int64	wd;
};
```

每个添加到epoll中的fd都对应了一个PollDesc结构实例，PollDesc维护了**读写此fd的goroutine**这一非常重要的信息。在fd上读写遇到EAGAIN时，就将当前goroutine存储到这个fd对应的PollDesc中，并且把goutine变成Gwaiting，当在fd上再次发生事件之后，可以通过PollDesc找到之前阻塞的goroutine，激活从新运行。

#### net包

在net包中网络文件描述符都是用一个netFD结构体来表示的(net/fd_unix.go)，其中有个成员就是pollDesc。

```go
// 网络文件描述符
type netFD struct {
    sysmu  sync.Mutex
    sysref int

    // must lock both sysmu and pollDesc to write
    // can lock either to read
    closing bool

    // immutable until Close
    sysfd       int
    family      int
    sotype      int
    isConnected bool
    sysfile     *os.File
    net         string
    laddr       Addr
    raddr       Addr

    // serialize access to Read and Write methods
    rio, wio sync.Mutex

    // wait server
    pd pollDesc
}
```

netFD对象实现有自己的init方法，还有完成基本IO操作的Read和Write方法，当然除了这三个方法以外，还有很多非常有用的方法供用户使用。

通过netFD对象的定义可以看到每个fd都关联了一个pollDesc实例，通过上文我们知道pollDesc对象最终是对epoll的封装。

```go
func (fd *netFD) init() error {
	if err := fd.pd.Init(fd); err != nil {
		return err
	}
	return nil
}
```

netFD对象的init函数仅仅是调用了pollDesc实例的Init函数，作用就是将fd添加到epoll中，如果这个fd是第一个网络socket fd的话，这一次init还会担任创建epoll实例的任务。要知道在Go进程里，只会有一个epoll实例来管理所有的网络socket fd，这个epoll实例也就是在第一个网络socket fd被创建的时候所创建。

```go
for {
	n, err = syscall.Read(int(fd.sysfd), p)
	if err != nil {
		n = 0
		if err == syscall.EAGAIN {
			if err = fd.pd.WaitRead(); err == nil {
				continue
			}
		}
	}
	err = chkReadErr(n, err, fd)
	break
}
```

上面代码段是从netFD的Read方法中摘取，重点关注这个for循环中的syscall.Read调用的错误处理。当有错误发生的时候，会检查这个错误是否是syscall.EAGAIN，如果是，则调用WaitRead将当前读这个fd的goroutine给park住，直到这个fd上的读事件再次发生为止。

当这个socket上有新数据到来的时候，WaitRead调用返回，继续for循环的执行。这样的实现，就让调用netFD的Read的地方变成了同步“阻塞”方式编程，不再是同步非阻塞的编程方式了。netFD的Write方法和Read的实现原理是一样的，都是在碰到EAGAIN错误的时候将当前goroutine给park住直到socket再次可写为止。

### poller

后台有一个poller会不停地进行poll，所有的文件描述符都被添加到了这个poller中的，当某个时刻一个文件描述符准备好了，poller就会唤醒之前因它而阻塞的goroutine，于是goroutine重新运行起来。

在proc.c文件中，runtime.main函数的第一行代码就是

```go
newm(sysmon, nil);
```

这个意思就是新建一个M并让它运行sysmon函数，前面说过M就是机器的抽象，它会直接开一个物理线程。sysmon里面是个死循环，每睡眠一小会儿就会调用runtime.netpoll函数（epoll的封装），这个sysmon就是所谓的poller。

在sysmon函数中，会不停地调用runtime.netpoll，这个函数对就绪的网络连接进行epoll，返回可运行的goroutine。epoll只能知道哪个fd就绪了，通过PollDesc就可以得到其中的goroutine了。

## 流程图

![image-20180426233815056](/Users/sinnera/sinnera.github.io/source/illustrations/image-20180426233815056.png)

## 参考

[深入Go语言网络库的基础实现](http://skoo.me/go/2014/04/21/go-net-core)

[深入解析Go-非阻塞IO](https://tiancaiamao.gitbooks.io/go-internals/content/zh/08.1.html)
