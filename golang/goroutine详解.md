---
title: goroutine与调度器的实现
date: 2018-03-25
tags: 
  - Go
  - Goroutine
---

[TOC]

## 需要解决的问题

Go语言的协程实现goroutine，在SMP的机器上提供并发编程的支持。goruntine是Go语言运行库提供的功能，不依赖操作系统。可以将goroutine简单看做是Go提供的语法糖，将并发由传统的事件驱动+回调的方式，变成直观的同步方式，降低了并发编程的门槛。

但是由此带来的代价是，需要Go提供调度器解决以下问题：

1. 如何实现goroutine与内核线程的映射
2. 如何保存每个goroutine的上下文
3. 调度时机、调度策略
4. 数据共享带来的锁开销

## 设计与演化

### 线程池

在传统的并发编程中，我们很经常采取多线程+任务队列方式来实现。

```go
type Job interface {
  Run()
}

type JobQueue struct {
    jobs []Job
    lock *sync.Mutex
}

var queue JobQueue

func woker(){
  for {
    queue.lock.Lock()
    if len(queue.jobs) > 0{
      job := queue.jobs[0]
      queue.jobs = queue.jobs[1:]
      queue.lock.Unlock()
      job.Run()
    } else {
      queue.lock.Unlock()
    }
  }
}

for i := 0; i < workersNum; i++ {
  go worker() //开线程执行任务
}
```

简单来概括就是多个线程，不停地从任务队列取任务执行。显然，当线程在执行任务时，可能会失去控制权，它可能一直占用着CPU，或者阻塞于IO。线程的阻塞，导致CPU无法充分利用。

线程少了，CPU使用率低，线程多了，争抢CPU也会造成开销，可以限定线程数量workersNum，对线程池做负载调节。如果有线程阻塞了，可以另开一个新的线程作为替补；阻塞线程恢复正常之后，需要检查下当前工作线程数是否超过workerNum，如果未超过则让它继续工作，否则让它闲置。

```go
func entersyscall(){
  go worker() //另开新的线程
}

func exitsyscall(){
  if len(workers) > workersNum {
    queue.lock.Lock()
    queue.jobs = append(queue.jobs, job) //放回执行完的任务
    queue.lock.Unlock()
    time.Sleep() //闲置
  }
}
```

上面的方式简单易懂，但其实传统的多线程编程并不是采用这种方式，是采用事件+回调来实现的，不够直观，代码难以维护。为什么不采用这种简单方式呢？因为要实现这种方式，背后要解决不少问题，比如保存和恢复上下文环境，需要自己提供调度器解决这些问题。

### Go1.0

Go语言提供的协程调度机制与上面的方式类似，Go1.0中有两个结构体M和G，分别代表操作内核系统线程和goroutine，对应上面的worker和job。参数GOMAXPROCS对应上面的workerNum，GOMAXPROCS可以根据实际情况来设置，一般设置为大于CPU核数，默认是1，最大可设为256。

类似的，scheduler匹配G和M，让闲置的M运行G。M不够时创建新的M；正在运行的M太多时，挂起当前刚闲置的M，放入等待队列中。

每次调度都会涉及相关队列的操作，这些全局数据在多线程的情况下会涉及大量锁操作，这会造成很大的开销。为了减少系统开销，1.0版本做了优化。G中有个sched域，用于保存上下文的，sched中有个atomic字段，是一个volatile的uint32，该字段作用：

```go
// sched中的原子字段是一个原子的uint32，存放下列域
// 15位 mcpu  --正在占用cpu运行的m数量 (进入syscall的m是不占用cpu的)
// 15位 mcpumax  --最大允许这么多个m同时使用cpu
// 1位  waitstop  --有g等待结束
// 1位  gwaiting  --等待队列不为空，有g处于waiting状态
//    [15 bits] mcpu        number of m's executing on cpu
//    [15 bits] mcpumax    max number of m's allowed on cpu
//    [1 bit] waitstop    some g is waiting on stopped
//    [1 bit] gwaiting    gwait != 0
```

每次系统调用都需要用到该字段中的这些信息，它们会决定是否需要进入到调度器层面。直接用CAS操作该字段，通过将它们合在一个字节中使得可以通过一次原子读写获取而不用加锁。这个优化，将极大减少哪些大量系统调带来的锁竞争。

除了系统调用外，操作sched域只会发生在持有调度器锁的时候，因此其他goroutine不会对本goroutine的shced域进行操作。

总结下来，Go1.0调度设计比较简单，就是上述线程池模型。但是存在很多问题：

1. 所有的goroutine只有一个锁来保护，粒度很大，只要goroutine相关操作都会锁住调度
2. 每个M都可以执行任何的G。多核CPU中每个核都去执行不同线程的代码，这显然不能利用缓存局部性，切换开销也变大
3. 内存缓存MCache是M级别的，当M阻塞，相应的内存资源也被浪费了。内存和其他缓存是关联到所有M的，实际上只需要关联到运行G的M上
4. 系统调用时工作线程频繁切换

#### Go1.1

为了解决上个版本存在的问题，Go1.1中新增数据结构P，P介于M和G之间，相当于加了一层，类似于解耦。P（Processer）代表G执行时需要的资源。此时，GOMAXPROCS代表P的数量。

goroutine分配到了各个P中，是P局部的goroutine，只有P才能操作自己的goroutine，不会与其他P产生冲突。全局的grunnable队列依然存在，只有P去访问全局队列时才需要加锁，这样解决了单个全局锁的问题。sched域中的一些信息，挪到到了P中，如gfree和grunnable。MCache也从M移到了P中，解决了内存资源浪费的问题。有了P之后，sched中的atomic也没有存在的必要了，成为了历史。

M执行G之前，它需要关联一个P，只能执行这个P中的G；当M为idle时，它需要新的P；M在系统调用中时，需要新的M来代替执行P，总之M不能闲下来。新建的G或者现有的G变成runnable，它将被分配到一个P中。当P执行完G，G将被P从grunnable队列中移除。如果P的grunnable队列为空，P会采取某种方式从其他P中窃取可运行的G，类似于负载均衡，提高效率。

## 具体实现

用户空间线程和内核空间线程之间的映射关系有：N:1,1:1和M:N
N:1是说，多个（N）用户线程始终在一个内核线程上跑，context上下文切换确实很快，但是无法真正的利用多核。
1：1是说，一个用户线程就只在一个内核线程上跑，这时可以利用多核，但是上下文switch很慢。
M:N是说， 多个goroutine在多个内核线程上跑，这个看似可以集齐上面两者的优势，但是无疑增加了调度的难度。

![img](https://pic1.zhimg.com/80/2f5c6ef32827fb4fc63c60f4f5314610_hd.jpg)

Go的调度器内部有三个重要的结构：M，P，S
M:代表真正的内核OS线程，和POSIX里的thread差不多，真正干活的人
G:代表一个goroutine，它有自己的栈，instruction pointer和其他信息（正在等待的channel等等），用于调度。
P:代表调度的上下文，可以把它看做一个局部的调度器，使go代码在一个线程上跑，它是实现从N:1到N:M映射的关键。

![img](https://pic3.zhimg.com/80/67f09d490f69eec14c1824d939938e14_hd.jpg)

图中看，有2个物理线程M，每一个M都拥有一个context（P），每一个也都有一个正在运行的goroutine。P的数量可以通过GOMAXPROCS()来设置，它其实也就代表了真正的并发度，即有多少个goroutine可以同时运行。图中灰色的那些goroutine并没有运行，而是出于ready的就绪态，正在等待被调度。P维护着这个队列（称之为runqueue），Go语言里，启动一个goroutine很容易：go function 就行，所以每有一个go语句被执行，runqueue队列就在其末尾加入一个goroutine，在下一个调度点，就从runqueue中取出（如何决定取哪个goroutine？）一个goroutine执行。为何要维护多个上下文P？因为当一个OS线程被阻塞时，P可以转而投奔另一个OS线程！
图中看到，当一个OS线程M0陷入阻塞时，P转而在OS线程M1上运行。调度器保证有足够的线程来运行所以的context P。

![img](https://pic1.zhimg.com/80/f1125f3027ebb2bd5183cf8c9ce4b3f2_hd.jpg)

图中的M1可能是被创建，或者从线程缓存中取出。当MO返回时，它必须尝试取得一个context P来运行goroutine，一般情况下，它会从其他的OS线程那里steal偷一个context过来，如果没有偷到的话，它就把goroutine放在一个global runqueue里，然后自己就去睡大觉了（放入线程缓存里）。Contexts们也会周期性的检查global runqueue，否则global runqueue上的goroutine永远无法执行。

![img](https://pic3.zhimg.com/80/31f04bb69d72b72777568063742741cd_hd.jpg)

另一种情况是P所分配的任务G很快就执行完了（分配不均），这就导致了一个上下文P闲着没事儿干而系统却任然忙碌。但是如果global runqueue没有任务G了，那么P就不得不从其他的上下文P那里拿一些G来执行。一般来说，如果上下文P从其他的上下文P那里要偷一个任务的话，一般就‘偷’run queue的一半，这就确保了每个OS线程都能充分的使用。

## 数据结构

Go的调度的实现，涉及到几个重要的数据结构。运行时库用这几个数据结构来实现goroutine的调度，管理goroutine和物理线程的运行。这些数据结构分别是结构体G，结构体M，结构体P，以及Sched结构体。前三个的定义在文件runtime/runtime.h中，而Sched的定义在runtime/proc.c中。Go语言的调度相关实现也是在文件proc.c中。

### 结构体G

G是goroutine的缩写，相当于操作系统中的进程控制块，在这里就是goroutine的控制结构，是对goroutine的抽象。其中包括goid是这个goroutine的ID，status是这个goroutine的状态，如Gidle,Grunnable,Grunning,Gsyscall,Gwaiting,Gdead等。

```
struct G
{
    uintptr    stackguard;    // 分段栈的可用空间下界
    uintptr    stackbase;    // 分段栈的栈基址
    Gobuf    sched;        //进程切换时，利用sched域来保存上下文
    uintptr    stack0;
    FuncVal*    fnstart;        // goroutine运行的函数
    void*    param;        // 用于传递参数，睡眠时其它goroutine设置param，唤醒时此goroutine可以获取
    int16    status;        // 状态Gidle,Grunnable,Grunning,Gsyscall,Gwaiting,Gdead
    int64    goid;        // goroutine的id号
    G*    schedlink;
    M*    m;        // for debuggers, but offset not hard-coded
    M*    lockedm;    // G被锁定只能在这个m上运行
    uintptr    gopc;    // 创建这个goroutine的go表达式的pc
    ...
};

```

结构体G中的部分域如上所示。可以看到，其中包含了栈信息stackbase和stackguard，有运行的函数信息fnstart。这些就足够成为一个可执行的单元了，只要得到CPU就可以运行。

goroutine切换时，上下文信息保存在结构体的sched域中。goroutine是轻量级的`线程`或者称为`协程`，切换时并不必陷入到操作系统内核中，所以保存过程很轻量。看一下结构体G中的Gobuf，其实只保存了当前栈指针，程序计数器，以及goroutine自身。

```
struct Gobuf
{
    // The offsets of these fields are known to (hard-coded in) libmach.
    uintptr    sp;
    byte*    pc;
    G*    g;
    ...
};

```

记录g是为了恢复当前goroutine的结构体G指针，运行时库中使用了一个常驻的寄存器`extern register G* g`，这个是当前goroutine的结构体G的指针。这样做是为了快速地访问goroutine中的信息，比如，Go的栈的实现并没有使用%ebp寄存器，不过这可以通过g->stackbase快速得到。"extern register"是由6c，8c等实现的一个特殊的存储。在ARM上它是实际的寄存器；其它平台是由段寄存器进行索引的线程本地存储的一个槽位。在linux系统中，对g和m使用的分别是0(GS)和4(GS)。需要注意的是，链接器还会根据特定操作系统改变编译器的输出，例如，6l/linux下会将0(GS)重写为-16(FS)。每个链接到Go程序的C文件都必须包含runtime.h头文件，这样C编译器知道避免使用专用的寄存器。

### 结构体M

M是machine的缩写，是对机器的抽象，每个m都是对应到一条操作系统的物理线程。M必须关联了P才可以执行Go代码，但是当它处理阻塞或者系统调用中时，可以不需要关联P。

```
struct M
{
    G*    g0;        // 带有调度栈的goroutine
    G*    gsignal;    // signal-handling G 处理信号的goroutine
    void    (*mstartfn)(void);
    G*    curg;        // M中当前运行的goroutine
    P*    p;        // 关联P以执行Go代码 (如果没有执行Go代码则P为nil)
    P*    nextp;
    int32    id;
    int32    mallocing; //状态
    int32    throwing;
    int32    gcing;
    int32    locks;
    int32    helpgc;        //不为0表示此m在做帮忙gc。helpgc等于n只是一个编号
    bool    blockingsyscall;
    bool    spinning;
    Note    park;
    M*    alllink;    // 这个域用于链接allm
    M*    schedlink;
    MCache    *mcache;
    G*    lockedg;
    M*    nextwaitm;    // next M waiting for lock
    GCStats    gcstats;
    ...
};
```

这里也是截取结构体M中的部分域。和G类似，M中也有alllink域将所有的M放在allm链表中。lockedg是某些情况下，G锁定在这个M中运行而不会切换到其它M中去。M中还有一个MCache，是当前M的内存的缓存。M也和G一样有一个常驻寄存器变量，代表当前的M。同时存在多个M，表示同时存在多个物理线程。

结构体M中有两个G是需要关注一下的，一个是curg，代表结构体M当前绑定的结构体G。另一个是g0，是带有调度栈的goroutine，这是一个比较特殊的goroutine。普通的goroutine的栈是在堆上分配的可增长的栈，而g0的栈是M对应的线程的栈。所有调度相关的代码，会先切换到该goroutine的栈中再执行。

### 结构体P

Go1.1中新加入的一个数据结构，它是Processor的缩写。结构体P的加入是为了提高Go程序的并发度，实现更好的调度。M代表OS线程。P代表Go代码执行时需要的资源。当M执行Go代码时，它需要关联一个P，当M为idle或者在系统调用中时，它也需要P。有刚好GOMAXPROCS个P。所有的P被组织为一个数组，在P上实现了工作流窃取的调度器。

```
struct P
{
    Lock;
    uint32    status;  // Pidle或Prunning等
    P*    link;
    uint32    schedtick;   // 每次调度时将它加一
    M*    m;    // 链接到它关联的M (nil if idle)
    MCache*    mcache;

    G*    runq[256];
    int32    runqhead;
    int32    runqtail;

    // Available G's (status == Gdead)
    G*    gfree;
    int32    gfreecnt;
    byte    pad[64];
};
```

结构体P中也有相应的状态：

```
Pidle,
Prunning,
Psyscall,
Pgcstop,
Pdead,
```

注意，跟G不同的是，P不存在`waiting`状态。MCache被移到了P中，但是在结构体M中也还保留着。在P中有一个Grunnable的goroutine队列，这是一个P的局部队列。当P执行Go代码时，它会优先从自己的这个局部队列中取，这时可以不用加锁，提高了并发度。如果发现这个队列空了，则去其它P的队列中拿一半过来，这样实现工作流窃取的调度。这种情况下是需要给调用器加锁的。

### Sched

Sched是调度实现中使用的数据结构，该结构体的定义在文件proc.c中。

```
struct Sched {
    Lock;

    uint64    goidgen;

    M*    midle;     // idle m's waiting for work
    int32    nmidle;     // number of idle m's waiting for work
    int32    nmidlelocked; // number of locked m's waiting for work
    int3    mcount;     // number of m's that have been created
    int32    maxmcount;    // maximum number of m's allowed (or die)

    P*    pidle;  // idle P's
    uint32    npidle;  //idle P的数量
    uint32    nmspinning;

    // Global runnable queue.
    G*    runqhead;
    G*    runqtail;
    int32    runqsize;

    // Global cache of dead G's.
    Lock    gflock;
    G*    gfree;

    int32    stopwait;
    Note    stopnote;
    uint32    sysmonwait;
    Note    sysmonnote;
    uint64    lastpoll;

    int32    profilehz;    // cpu profiling rate
}
```

大多数需要的信息都已放在了结构体M、G和P中，Sched结构体只是一个壳。可以看到，其中有M的idle队列，P的idle队列，以及一个全局的就绪的G队列。Sched结构体中的Lock是非常必须的，如果M或P等做一些非局部的操作，它们一般需要先锁住调度器。

## 生老病死

#### go关键字

在Go语言中，表达式go f(x,y,z)会启动一个新的goroutine运行函数f(x,y,z)。函数f，变量x、y、z的值是在原goroutine计算的，只有函数f的执行是在新的goroutine中的。显然，新的goroutine不能和当前go线程用同一个栈，否则会相互覆盖。所以对go关键字的调用协议与普通函数调用是不同的。

先看看正常的函数调用，下面是调用f(1, 2, 3)时的汇编代码：

```assembly
    MOVL    $1, 0(SP)
    MOVL    $2, 4(SP)
    MOVL    $3, 8(SP)
    CALL    f(SB)
```

首先将参数1、2、3进栈，然后调用函数f。

下面是go f(1, 2, 3)生成的代码：

```assembly
    MOVL    $1, 0(SP)
    MOVL    $2, 4(SP)
    MOVL    $3, 8(SP)
    PUSHQ   $f(SB)
    PUSHQ   $12
    CALL    runtime.newproc(SB)
    POPQ    AX
    POPQ    AX
```

对比一个会发现，前面部分跟普通函数调用是一样的，将参数存储在正常的位置，接下来的两条指令有些不同，将f和12作为参数进栈而不直接调用f，然后调用函数`runtime.newproc`。12是参数占用的大小。`runtime.newproc`函数接受的参数分别是：参数大小，新的goroutine要运行的函数，函数的n个参数。

在`runtime.newproc`中，会新建一个栈空间，将栈参数的12个字节拷贝到新栈空间中并让栈指针指向参数。新建G，寄存器PC、SP会被保存到G内，函数f被存放在了G的entry域，这时新的G会被放在P的G就绪队列中，后面进行调度时，G便在P对应的M上运行，G将从f开始执行。

总结一个，go关键字的实现仅仅是一个语法糖衣而已，也就是：

```go
    go f(args)
```

可以看作

```c
    runtime.newproc(size, f, args)
```

#### 实例说明

下图是协程可能出现的工作状态, 图中有4个P, 其中M1~M3正在运行G并且运行后会从拥有的P的运行队列继续获取G:

![img](https://images2017.cnblogs.com/blog/881857/201711/881857-20171110171241372-2120016927.png)

只看这张图可能有点难以想象实际的工作流程, 这里我根据实际的代码再讲解一遍:

```go
package main

import (
    "fmt"
    "time"
)

func printNumber(from, to int, c chan int) {
    for x := from; x <= to; x++ {
        fmt.Printf("%d\n", x)
        time.Sleep(1 * time.Millisecond)
    }
    c <- 0
}

func main() {
    c := make(chan int, 3)
    go printNumber(1, 3, c)
    go printNumber(4, 6, c)
    _ = <- c
    _ = <- c
}
```

程序启动时会先创建一个G, 指向的是main(实际是runtime.main而不是main.main, 后面解释):
图中的虚线指的是G待运行或者开始运行的地址, 不是当前运行的地址.

![img](https://images2017.cnblogs.com/blog/881857/201711/881857-20171110171251091-1325784333.png)

M会取得这个G并运行:

![img](https://images2017.cnblogs.com/blog/881857/201711/881857-20171110171257559-269808777.png)

这时main会创建一个新的channel, 并启动两个新的G:

![img](https://images2017.cnblogs.com/blog/881857/201711/881857-20171110171304919-1329887936.png)

接下来`G: main`会从channel获取数据, 因为获取不到, G会**保存状态**并变为等待中(_Gwaiting)并添加到channel的队列:

![img](https://images2017.cnblogs.com/blog/881857/201711/881857-20171110171312638-525155603.png)

因为`G: main`保存了运行状态, 下次运行时将会从`_ = <- c`继续运行.
接下来M会从运行队列获取到`G: printNumber`并运行:

![img](https://images2017.cnblogs.com/blog/881857/201711/881857-20171110171319888-1466945199.png)

printNumber会打印数字, 完成后向channel写数据,
写数据时发现channel中有正在等待的G, 会把数据交给这个G, 把G变为待运行(_Grunnable)并重新放入运行队列:

![img](https://images2017.cnblogs.com/blog/881857/201711/881857-20171110171328216-1977430311.png)

接下来M会运行下一个`G: printNumber`, 因为创建channel时指定了大小为3的缓冲区, 可以直接把数据写入缓冲区而无需等待:

![img](https://images2017.cnblogs.com/blog/881857/201711/881857-20171110171335450-1117872609.png)

然后printNumber运行完毕, 运行队列中就只剩下`G: main`了:

![img](https://images2017.cnblogs.com/blog/881857/201711/881857-20171110171343934-830758071.png)

最后M把`G: main`取出来运行, 会从上次中断的位置`_ <- c`继续运行:

![img](https://images2017.cnblogs.com/blog/881857/201711/881857-20171110171354653-922259524.png)

第一个`_ <- c`的结果已经在前面设置过了, 这条语句会执行成功.
第二个`_ <- c`在获取时会发现channel中有已缓冲的0, 于是结果就是这个0, 不需要等待.
最后main执行完毕, 程序结束.

有人可能会好奇如果最后再加一个`_ <- c`会变成什么结果, 这时因为所有G都进入等待状态, go会检测出来并报告死锁:

```
fatal error: all goroutines are asleep - deadlock!
```

#### 进出系统调用

进入系统调用时，如果系统调用是阻塞的，goroutine会被剥夺CPU，将状态设置成Gsyscall后放到就绪队列。Go的syscall库中提供了对系统调用的封装，它会在真正执行系统调用之前先调用函数runtime.entersyscall，并在系统调用函数返回后调用runtime.exitsyscall函数。这两个函数就是通知Go的运行时库这个goroutine进入了系统调用或者完成了系统调用，调度器会做相应的调度。

**runtime.entersyscall**

首先，将函数的调用者的SP，PC等保存到结构体G的sched域中。同时，也保存到g->gcsp和g->gcpc等，这个是跟垃圾回收相关的。

然后，检查结构体Sched中的sysmonwait域，如果不为0，则将它置为0，并调用runtime·notewakeup。做这这一步的原因是，目前这个goroutine要进入Gsyscall状态了，它将要让出CPU。如果有人在等待CPU的话，会通知并唤醒等待者，马上就有CPU可用了。

最后，将m的MCache置为空，并将m->p->m置为空，表示进入系统调用后结构体M是不需要MCache的，并且P也被剥离了，将P的状态设置为PSyscall，将P从处于syscall或者locked的M中，切换出来交给其它M。

**runtime.exitsyscall**

它会先检查当前M的P和它状态，如果P不空且状态为Psyscall，则说明是从一个非阻塞的系统调用中返回的，这时是仍然有CPU可用的。因此将p->m设置为当前m，将p的mcache放回到m，恢复g的状态为Grunning。

否则，它是从一个阻塞的系统调用中返回的，因此之前m的P已经完全被剥离了。这时会查看调用中是否还有idle的P，如果有，则将它与当前的M绑定。

如果从一个阻塞的系统调用中出来，并且出来的这一时刻又没有idle的P了，要怎么办呢？这种情况代码当前的goroutine无法继续运行了，调度器会将它的状态设置为Grunnable，将它挂到全局的就绪G队列中，然后停止当前m并调用schedule函数。

#### 状态转换图

goroutine的消亡比较简单，注意在函数newproc1，设置了fnstart为goroutine执行的函数，而将新建的goroutine的sched域的pc设置为了函数runtime.exit。当fnstart函数执行完返回时，它会返回到runtime.exit中。这时Go就知道这个goroutine要结束了，runtime.exit中会做一些回收工作，会将g的状态设置为Gdead等，并将g挂到P的free队列中。

从以上的分析中，其实已经基本上经历了goroutine的各种状态变化。在newproc1中新建的goroutine被设置为Grunnable状态，投入运行时设置成Grunning。在entersyscall的时候goroutine的状态被设置为Gsyscall，到出系统调用时根据它是从阻塞系统调用中出来还是非阻塞系统调用中出来，又会被设置成Grunning或者Grunnable的状态。在goroutine最终退出的runtime.exit函数中，goroutine被设置为Gdead状态。

等等，好像缺了什么？是的，Gidle始终没有出现过。这个状态好像实际上没有被用到。只有一个runtime.park函数会使goroutine进入到Gwaiting状态，但是park这个有什么作用我暂时还没看懂...

goroutine的状态变迁图：

![img](http://tiancaiamao.gitbooks.io/go-internals/content/zh/images/5.2.goroutine_state.jpg?raw=true)

## 关键点

#### 栈管理

Go1.4之后，Go语言的stack处理方式由之前的"segmented stacks"（分段栈）改为了"continuous stacks"（连续栈）。关于Go语言对stack的处理机制、发展历史、存在问题等，可以参考《[Go语言是如何处理栈的](https://tonybai.com/2014/11/05/how-stacks-are-handled-in-go/)》。

##### 分段栈

每个go函数入口，都有一小段代码（called prologue），检查是否用光了已分配的栈空间（Go1.4之前默认为8KB），如果用光了则会调用morestack函数。分段栈在栈空间不足时，morestack会一段新的内存空间作为新的栈，栈底的stuct会记录栈的相关信息，比如上一个栈的地址。这时将重启goroutine，从刚才那个导致栈空间不足的函数Func开始执行，这就是“栈分裂”（stack split）。在新栈的底部还被插入了一个栈入口函数lessstack，当函数Func返回时，会调用lessstack，它将调整栈指针，指向前一段栈空间，并释放刚分配的新栈，继续执行当前程序。

这种方式很明显的缺点是，如果遇到某些场景比如循环，会产生频繁的栈分裂，即导致频繁的内存分配和释放，将极大影响性能。

##### 连续栈

Go1.4以及之后的版本，go使用连续栈来管理栈，先分配一块固定大小的栈（栈空间大小默认为2KB），在栈空间不足的情况下，分配一块新的2倍大的栈空间，并将旧的栈内容全部拷贝到新栈中。这样避免了栈分裂导致的频繁内存分配和释放。

这里有个技术难点，旧栈数据复制到新栈的过程，要考虑指针失效问题。Go实现了精确的垃圾回收，运行时知道每一块内存对应的对象的类型信息。在复制之后，会进行指针的调整。具体做法是，对当前栈帧之前的每一个栈帧，对其中的每一个指针，检测指针指向的地址，如果指向地址是落在旧栈范围内的，则将它加上一个偏移使它指向新栈的相应地址。这个偏移值等于新栈基地址减旧栈基地址。

整个过程有点像一次中断，中断处理时保存当时的现场，弄个新的栈，中断恢复时恢复到新栈中运行。栈的收缩是垃圾回收的过程中实现的。当检测到栈只使用了不到1/4时，栈缩小为原来的1/2。

##### 虚拟内存

另外一种不同的栈处理方式就是在虚拟内存中分配大内存段。由于物理内存只是在真正使用时才会被分配，因此看起来好似你可以分配一个大内存段并让操 作系统处理它。下面是这种方法的一些问题

首先，32位系统只能支持4G字节虚拟内存，并且应用只能用到其中的3G空间。由于同时运行百万goroutines的情况并不少见，因此你很可 能用光虚拟内存，即便我们假设每个goroutine的stack只有8K。

第二，然而我们可以在64位系统中分配大内存，它依赖于过量内存使用。所谓过量使用是指当你分配的内存大小超出物理内存大小时，依赖操作系统保证 在需要时能够分配出物理内存。然而，允许过量使用可能会导致一些风险。由于一些进程分配了超出机器物理内存大小的内存，如果这些进程使用更多内存 时，操作系统将不得不为它们补充分配内存。这会导致操作系统将一些内存段放入磁盘缓存，这常常会增加不可预测的处理延迟。正是考虑到这个原因，一 些新系统关闭了对过量使用的支持。

#### 抢占式调度

goroutine本来是设计为协程形式，但是随着调度器的实现越来越成熟，Go在1.2版中开始引入比较初级的抢占式调度。Go在设计之初并没考虑将goroutine设计成抢占式的。用户负责让各个goroutine交互合作完成任务。一个goroutine只有在涉及到加锁，读写通道或者主动让出CPU等操作时才会触发切换。

垃圾回收器是需要stop the world的。如果垃圾回收器想要运行了，那么它必须先通知其它的goroutine合作停下来，这会造成较长时间的等待时间。考虑一种很极端的情况，所有的goroutine都停下来了，只有其中一个没有停，那么垃圾回收就会一直等待着没有停的那一个。抢占式调度可以解决这种问题，在抢占式情况下，如果一个goroutine运行时间过长，它就会被剥夺运行权。

Go程序的初始化过程中，runtime开了一条后台线程，运行一个sysmon函数。这个函数会周期性地做epoll操作，同时它还会检测每个P是否运行了较长时间，是的话则会将P的当前的G的stackguard设置为StackPreempt。这个操作其实是相当于加上一个标记，通知这个G在合适时机进行调度。

Go只是引入了一些很初级的抢占，并没有像操作系统调度那么复杂，没有对goroutine分时间片，设置优先级等。只有长时间阻塞于系统调用，或者运行了较长时间才会被抢占。实际上，为了不破坏原有代码，为了实现抢占式调度，只是在函数入口的morestack中加了检查是否需要调度的一段代码（check flag）。如果一个goroutine运行了很久，但是它并没有调用另一个函数，则它不会被抢占。当然，一个运行很久却不调用函数的代码并不是多数情况。

#### 调度时机

##### 系统调用

golang对操作系统的系统调用作了封装，提供了syscall这样的库让我们执行系统调用。

entersyscall：P中的某个G进入系统调用状态，则会阻塞了该P内的其他的G。P此时会脱离当前的M，带着其他的G寻找新的M。

exitsyscall：M从系统调用返回时，会寻找空闲的P来运行G，如果找不到，则将当前的G扔到global queue中，并将自己挂起。

##### 读写channel

Golang中的channel读写均是同步语义，写满的、读空的channel都会触发协程调度。

无论是无缓冲还是有缓冲channel，当向channel写数据发现channel已满时，都需要将当前写的协程挂起，并进行一次调度，当前M转而执行P内的其他协程。直到有人再读该channel时发现有阻塞等待写的协程时将其唤醒。

从channel读数据和向channel写数据的实现几乎一样，只是方向反了而已。

##### 抢占式调度

sysmon协程会定期做系统状态检查，避免某个G占用了过多的cpu时间，并在某个时刻剥夺其cpu运行时间。

#### work stealing

在多线程计算里, 调度出现了两种模式: work-sharing (工作共享) 和 work-stealing (工作窃取)。

- **work-sharing** 当一个处理器产生新的线程时, 它试图将其中的一些迁移到其他处理器上, 希望它们能被空闲或未充分利用的处理器所利用.
- **work-stealing** 未充分利用的处理器会主动去寻找其他处理器的线程并 `窃取` 一些.

work-stealing 中线程迁移的频率少于 work-sharing. 当所有处理器都有工作要运行时, 没有线程会被迁移. 而一旦有空闲的处理器, 就会考虑迁移。

##### Stealing (窃取)

Go调度器通过引入*P*，实现了work-stealing的调度算法：

- 每个*P*维护一个*G*队列；
- 当一个*G*被创建出来，或者变为可执行状态时，就把他放到*P*的可执行队列中；
- 当一个*G*执行结束时，*P*会从队列中把该*G*取出；如果此时*P*的队列为空，即没有其他*G*可以执行， 就随机选择另外一个*P*，从其可执行的*G*队列中偷取一半。

​       该算法避免了在Goroutine调度时使用全局锁。

##### 自旋线程 (Spinning threads)

`自旋`：当线程需要某个资源，但这个资源没有到位，这时就进行一个死循环，从而不放弃cpu执行时间，也不进行上下文切换。

操作系统线程不应该频繁地在 goroutine(s) 之间切换, 因为这会增加延迟。

为了尽量减少切换，Go 调度器实现了 `自旋线程`。自旋线程消耗一点额外的 CPU，但是它们最小化了 OS 线程的抢占。一个线程是自旋的，如果：

- 分配了 P 的 M 正在寻找一个可执行 goroutine；
- 没有分配 P 的 M 正在寻找可用的 P；
- 调度器还会释放一个附加的线程, 当它正准备一个 goroutine 并且没有空闲的 P 也没有其他自旋线程的时候让它自旋。

任何时候最多有 GOMAXPROCS 个自旋的 M (们)。 当一个自旋的线程找到工作，它就脱离了自旋状态。

## 参考

[《深入解析Go》goroutine调度](https://tiancaiamao.gitbooks.io/go-internals/content/zh/05.0.html)

[goroutine与调度器](http://skoo.me/go/2013/11/29/golang-schedule)

[The Go scheduler](https://link.zhihu.com/?target=http%3A//morsmachine.dk/go-scheduler)

[goroutine背后的系统知识](http://www.infoq.com/cn/articles/knowledge-behind-goroutine)

[Golang 的 goroutine 是如何实现的？](https://www.zhihu.com/question/20862617)

[Golang调度器源码分析](http://ga0.github.io/golang/2015/09/20/golang-runtime-scheduler.html)

[并发之痛 Thread，Goroutine，Actor](http://jolestar.com/parallel-programming-model-thread-goroutine-actor/)

[也谈goroutine调度器](https://tonybai.com/2017/06/23/an-intro-about-goroutine-scheduler/)

[Goroutine是如何工作的](https://tonybai.com/2014/11/15/how-goroutines-work/)

[Go work-stealing 调度器](https://juejin.im/entry/5a71925c6fb9a01c9e4645c4)

[Implemention of golang](https://tracymacding.gitbooks.io/implementation-of-golang/content/)