---
title: golang channel的实现
date: 2018-05-06
tags: 
  - golang
---

[TOC]

## 底层实现

官方提倡的一个编程信条——“使用通信去共享内存，而不是共享内存去通信”，这里说的”通信去共享内存”的手段就是channel。channel简单来说就是一个**并发安全的队列**，用Hchan表示：

```go
struct    Hchan
{
    qcount    uint           // 队列中数据个数
    dataqsiz  uint           // channel 大小
    buf       unsafe.Pointer // 存放数据的环形数组
    elemsize  uint16         // channel 中数据类型的大小
    closed    uint32         // 表示 channel 是否关闭
    elemtype  *_type // 元素数据类型
    sendx     uint   // send 的数组索引
    recvx     uint   // recv 的数组索引
    recvq     waitq  // 由 recv 行为（也就是 <-ch）阻塞在 channel 上的 goroutine 队列
    sendq     waitq  // 由 send 行为 (也就是 ch<-) 阻塞在 channel 上的 goroutine 队列

    lock mutex
};
```

其实channel就是一个队列+锁组成的

recvq是读操作阻塞在channel的goroutine队列，sendq是写操作阻塞在channel的goroutine队列。队列用链表WaitQ表示，链表的元素是SudoG，对goroutine + elem封装的一个结构体。

```go
struct WaitQ
{
    SudoG*    first;
    SudoG*    last;
};

struct SudoG
{
    G*    g;        // g and selgen constitute
    uint32    selgen;        // a weak pointer to g
    SudoG*    link;
    int64    releasetime;
    byte*    elem;        // data element
};
```

![img](/Users/sinnera/sinnera.github.io/source/illustrations/channel.png)

对于缓存chan来说，缓存数据实际上是紧挨着Hchan结构体分配的。也就是Hchan中的buf指向的地址，是一个数组，大小为make channel指定的缓冲大小。

chan是FIFO的，那是怎么做到的呢？事实上，Hchan中还有两个关键字段recvx和sendx，在它们的配合下就将该数组构成了一个循环数组，就这样利用数组实现了一个**循环队列**。（sendx和recvx分别为循环队列的head和last，每次从head出，从last进，实现FIFO）

![img](/Users/sinnera/sinnera.github.io/source/illustrations/channel02.png)

## 读写操作

### 写channel

完成写channel操作的函数是`runtime·chansend`，这个函数同时实现了同步/异步写channel，也就是带/不带缓冲的channel的写操作都是在这个函数里实现的。同步写，还是异步写，其实就是判断是否有缓冲区。

1. 加锁，**锁住整个channel结构**（就是上面的贴图模型）。这里可以看出，锁的粒度不小，channel很多时候不一定有直接对共享变量加锁效率高。
2. 判断是否带缓冲区，如果有就做异步写，没有就做同步写。
3. **同步写**，判断recvq中是否存在等待的goroutine：
   - 存在，那么就从recvq等待队列里取出一个等待的goroutine，然后将要写入的元素直接交给(拷贝)这个goroutine（SudoG的elem域），然后再将这个goroutine给设置为ready状态，就可以开始运行了
   - 不存在，那么就将待写入的元素保存在当前执行写的goroutine的结构（SudoG）里，然后将当前goroutine入队到sendq中并被挂起，等待有人来读取元素后才会被唤醒
4. **异步写**，判断缓冲区是否被写满：
   - 未满，就将元素追加到缓冲区中，并且从recvq中试着取出一个等待的goroutine开始进行读操作(如果recvq中有等待读的goroutine的话)
   - 已满，就将当前goroutine入队到sendq中并被挂起等待

### 读channel

读操作与写操作正好相反，过程类似。完成读channel操作的函数是`runtime·chanrecv`, 下面简单的叙述一下读过程：

1. 同样首先加锁

2. 通过是否带缓冲来判断做同步读还是异步读, 类似写过程。

3. **同步**，判断recvq中是否存在等待的goroutine：

   - 存在，就从sendq队列取出一个等待写的goroutine，并把需要写入的数据拿过来(拷贝)，再将取出的goroutine置为ready
   - 不存在，就只能把当前读的goroutine给入队到recvq并被挂起

4. **异步**，判断缓冲区是否为空：

   - 不为空，当然就是取出最前面的元素，同时试着从sendq中取出一个等待写的goroutine唤醒它。

   - 为空，就是将当前读goroutine给入队到recvq并被挂起等待

## 异常处理

### nil channel

按照Go语言的语言规范，读写nil channel是永远阻塞的。其实在函数runtime.chansend和runtime.chanrecv开头就有判断这类情况，如果发现参数c是空的，则直接将当前的goroutine放到等待队列，状态设置为waiting。

因为永远阻塞，所以会报错：fatal error: all goroutines are asleep - deadlock!

```go
if c == nil {
    gopark(nil, nil, "chan send (nil chan)", traceEvGoStop, 2)
    throw("unreachable")
}
```

总结：不允许向nil channel进行读或写，否则代码无法编译通过。

### closed channel

读一个关闭的通道，永远不会阻塞，会返回一个通道数据类型的零值。这个实现也很简单，将零值复制到调用函数的参数ep中。

写一个关闭的通道，则会panic。

关闭一个空通道，也会导致panic。

```go
if c.closed != 0 {
    unlock(&c.lock)
    panic(plainError("send on closed channel"))
}
```

总结：只能对closed channel进行读操作，将返回默认值。写或关闭操作都会导致panic。

## 关闭channel

close channel的工作除了将 c.closed 设置为1，还会**唤醒所有等待在该channel上的goroutine，即recvq和sendq队列上的goroutine**，并使其进入 Grunnable 状态。这时sendq不为空的话，会导致panic，因为closed channel不允许写操作。

关闭channel时一件很危险的事，因为没有一个内置函数可以检查一个channel是否已经关闭，如果close已关闭的channel，则会导致panic。但是，就算有一个简单的 `closed(chan T) bool`函数来检查channel是否已经关闭，它的用处还是很有限的，就像内置的`len`函数用来检查缓冲channel中元素数量一样。原因就在于，已经检查过的channel的状态有可能在调用了类似的方法返回之后就修改了，因此返回来的值已经不能够反映刚才检查的channel的当前状态了。简单来说，不能保证原子性。

当然，可以用下面的方式检查是否已关闭，不过会读一次chan，并不存在只检查是否已关闭的方法。

```go
value, ok := <- channel
if !ok {
    // channel was closed
}
```

同理，判断channel是否为空，也不能用len()，可以用select：

```go
select {
    case x := <-ch:
        fmt.Printf("Value %d was read.\n", x)
    default:
        fmt.Println("No value ready, moving on.")
}
```

### Channel Closing Principle

>  不要从接收端（consumer）关闭channel，也不要关闭有多个并发发送者的channel

上面这个原则，可以解释为：

1. 一个sender，M个receiver，由sender来关闭channel，通知数据已发送完毕
2. N个sender， 一个receiver，receiver通过关闭一个额外的signal channel来通知sender停止写入channel
3. M个receiver，N个sender，它们当中任意一个通过通知一个moderator（仲裁者）关闭额外的signal channel来关闭channel
4. 如果确定不会有goroutine在通信过程中被阻塞，也可以不关闭channel，等待GC对其进行回收。

## select的实现

### 行为

为了便于理解，我们首先给出一个代码片段：

```go
// https://talks.golang.org/2012/concurrency.slide#32
select {
case v1 := <-c1:
    fmt.Printf("received %v from c1\n", v1)
case v2 := <-c2:
    fmt.Printf("received %v from c2\n", v1)
case c3 <- 23:
    fmt.Printf("sent %v to c3\n", 23)
default:
    fmt.Printf("no one was ready to communicate\n")
}
```

上面这段代码中，select 语句有四个 case 子语句，前两个是 receive 操作，第三个是 send 操作，最后一个是默认操作。代码执行到 select 时，case 语句会按照源代码的顺序被评估，且只评估一次，评估的结果会出现下面这几种情况：

1. 除 default 外，如果只有一个 case 语句评估通过，那么就执行这个case里的语句；
2. 除 default 外，如果有多个 case 语句评估通过，那么通过伪随机的方式随机选一个；
3. 如果 default 外的 case 语句都没有通过评估，那么执行default里的语句；
4. 如果没有 default，那么代码块会被阻塞，直到有一个 case 通过评估；否则一直阻塞

### 实现

在Go的语言规范中，select中的case的执行顺序是**随机的**，而不像switch中的case那样一条一条的顺序执行。

select和case关键字使用了下面的结构体：

```go
struct	Scase
{
	SudoG	sg;			// must be first member (cast to Scase)
	Hchan*	chan;		// chan
	byte*	pc;			// return pc
	uint16	kind;
	uint16	so;			// vararg of selected bool
	bool*	receivedp;	// pointer to received bool (recv2)
};

struct    Select
{
    uint16    tcase;            // 总的scase[]数量
    uint16    ncase;            // 当前填充了的scase[]数量
    uint16*    pollorder;       // case的poll次序
    Hchan**    lockorder;       // channel的锁住的次序
    Scase    scase[1];          // 每个case会在结构体里有一个Scase，顺序是按出现的次序
};
```

Scase中有一个chan字段，这个显然就是每个case条件上操作的channel了。

Select中重点关注pollorder、lockorder、scase三个字段：

- scase就是一个数组，数组元素为Scase类型，存储每个case条件
- lockorder指针指向的也是一个数组，元素为`Hchan *`类型，存储每个case条件中操作channel
- pollorder是一个uint16的数组，保存了case执行的顺序

![img](/Users/sinnera/sinnera.github.io/source/illustrations/select1.png)

select执行选择过程:

1. 使用洗牌算法，将case编号打乱，存储在pollorder数组中
2. 给每个channel加锁，对lockorder进行排序去重（多个case可能操作同一个channel）
3. 进入for循环，按pollorder的顺序去遍历所有case，碰到一个可以执行的case后就中断循环
4. 没有找到可以执行的case，但有default条件，执行default
5. 如果没找到可以执行的case，也没有default条件，则把当前的goroutine放到每个case对应的channel的等待队列中，并且挂起该goroutine
6. 被唤醒，继续循环找到可执行的case
7. select执行完退出的时候，不光是释放选择器对象，还会返回pc。这个pc就是本次select执行的case的地址。只有把这个栈地址返回，才能继续执行case条件中的语句。

总结：

- select语句的执行将会对涉及的所有channel加锁，并不是只加锁需要操作的channel。
- 对所有channel加锁之前，存在一个对涉及到的所有channel进行堆排序的过程，目的就是为了去重。
- select并不是直接随机选择一个可执行的case，而是事先将所有case洗牌，再从头到尾选择第一个可执行的case
- 如果select语句是放置在for循环中，长期执行，会不会每次循环都经历选择器的创建到释放的4个阶段？目前必然是这样子的，所以select的使用是有代价的，还不低。

## 常见问题

### channel or mutex

> Use whichever is most expressive and/or most simple.

上面是引用官方博客[Use a sync.Mutex or a channel?](https://github.com/golang/go/wiki/MutexOrChannel)的一句话，意思使用哪个更清晰或更简单，就是用哪个。

channel的成本高于mutex：

- channel内部有mutex
- 出让cpu并且让另一个goroutine获得执行机会，上下文切换成本，高于mutex的检查竞争状态的成本（只需要一个原子操作）
- 代码更加复杂

### select为什么随机选择执行

因为如果顺序执行，会容易总是执行同一个case。这里的随机是使用洗牌算法实现的。

## 参考

[channel in Go's runtime](http://skoo.me/go/2013/09/20/go-runtime-channel)

[select in Go's runtime](http://skoo.me/go/2013/09/26/go-runtime-select)

[深入解析Go-高级数据结构](https://tiancaiamao.gitbooks.io/go-internals/content/zh/07.0.html)

[[Golang]互斥到底该谁做？channel还是Mutex](https://blog.csdn.net/erlib/article/details/44152511)

[如何优雅地关闭Go channel](https://www.jianshu.com/p/d24dfbb33781)