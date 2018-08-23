---
title: golang锁的实现
date: 2018-05-13
tags: 
  - golang
---

[TOC]

## 锁

### 进程间通信

多个进程（线程）通过共享内存（或者共享文件）的方式进行通信就会出现竞争条件。竞争条件的意思是说两个或者多个进程读写某些共享数据，而最后的结果取决于进程运行的精确时序。

我们可以简单地把程序代码分成两部分：不会导致竞争条件的程序片段和会导致竞争条件的程序片段。**会导致竞争条件的程序片段就叫做临界区**。避免竞争条件只需要阻止多个进程同时读写共享的数据就可以了，也就是保证同时只有一个进程处于临界区内。

![img](/Users/sinnera/sinnera.github.io/source/illustrations/critical_zone.png)

### 锁的定义

> In computer science, a lock or mutex (from mutual exclusion) is a synchronization mechanism for enforcing limits on access to a resource in an environment where there are many threads of execution. A lock is designed to enforce a mutual exclusion concurrency control policy.

简单来说，锁就是保证并发情况下，只有一个进程处于临界区的一种机制。

#### 实现

##### 硬件

许多现代计算机系统都提供了特殊硬件指令以允许能**原子地**（不可中断地）检查和修改字的内容或交换两个字的内容（比如CAS，compare and swap）。其实很多锁的软件方案都是通过调用**原子操作**来实现的。

##### 软件

**信号量（semaphore）**。信号量是E. W. Dijkstra在1965年提出的一种方法，它使用一个整型变量来累计唤醒次数。一个信号量的取值可以为0（表示没有保存下来的唤醒次数）或者为正值（表示有一个或者多个唤醒操作）。信号量除了初始化外只能通过两个标准**原子操作**：wait()和signal()来访问。这些操作原来被称为P（荷兰语Proberen，测试）和V（荷兰语Verhogen，增加）。wait()和signal()的定义可表示如下：

```
wait(S) {
    while(S<=0)
        ;   //no-op //忙等待
    S--
}

signal(S) {
    S++
}
```

这里需要注意的是，在wait()和signal()操作中，对信号量整型值的修改必须不可分开地执行。另外对于wait(S)，对S的整型值的测试（S<=0）和对其可能的修改(S–)，也必须不被中断地执行。

其实，准确地说是信号量的实现方式有两种：wait的时候**忙等待**或者**阻塞自己**。

- 忙等待，就是一直在while循环中，直到singal唤醒自己。（自旋锁）
- 阻塞自己，每个信号量关联一个等待进程链表，wait的时候，进程加入等待链表，并block自己。等待signal时从等待链表中取出进程并唤醒。

```
//忙等待
wait(S) {
    while(S<=0)
        ;   //no-op
    S--
}

//阻塞
wait(semaphore *S) {
    S->value--;
    if (S->value < 0) {
        add this process to S->list;
        block()
    }
}
```

忙等待和阻塞自己的方式各有优劣：

- 忙等待会使CPU空转，好处是如果在当前时间片内锁被其他进程释放，当前进程直接就能拿到锁而不需要CPU进行进程调度了。适用于锁占用时间较短的情况，且不适合于单处理器。
- 阻塞不会导致CPU空转，但是block自己导致的**进程切换**也需要代价，比如上下文切换，CPU Cache Miss。

可以看出，这两种方式本质的区别就是，会不会导致进程切换。

**互斥量（mutext）**。互斥量可以认为是取值只有0和1的信号量，是信号量的一种。我们经常使用就是这个。使用其来同步的代码如下

```
do {
    waiting(mutex);
        //critical section
    signal(mutex);
        //remainder section
}while(TRUE)：
```

#### 种类

根据表现形式，常见的锁有互斥锁、读写锁、自旋锁。

**互斥锁**。只有取得互斥锁的进程才能进入临界区，无论读写。

**读写锁**。读写锁要根据进程进入临界区的具体行为（读，写）来决定锁的占用情况。这样锁的状态就有三种：读模式加锁、写模式加锁、无锁。

1. 无锁。读/写进程都可以进入。
2. 读模式锁。读进程可以进入。写进程不可以进入。
3. 写模式锁。读/写进程都不可以进入。

**自旋锁**。自旋锁是指在进程**试图取得锁失败的时候选择忙等待而不是阻塞自己**。选择忙等待的优点在于如果该进程在其自身的CPU时间片内拿到锁（说明锁占用时间都比较短），则相比阻塞**少了上下文切换**。注意这里还有一个隐藏条件：多处理器。因为单个处理器的情况下，由于当前自旋进程占用着CPU，持有锁的进程只有等待自旋进程耗尽CPU时间才有机会执行，这样CPU就空转了。

**注：自旋锁其实就是互斥锁的一种实现方式，就是上面提到的忙等待方式**

### 效率和粒度

总所周知，锁是有开销的，但是如果硬件支持的话，锁的开销就是一个CAS，相比之下，锁竞争的开销更大。关于锁竞争慢，可以参考这篇文章：[Locks Aren’t Slow; Lock Contention Is](http://preshing.com/20111118/locks-arent-slow-lock-contention-is/)。简单来说，**频繁的锁竞争会导致CPU中的进程上下文切换，同时还有缓存的失效。**

关于锁的粒度，用下面一张图来表示比较直观：不可过细也不可过糙。
![img](/Users/sinnera/sinnera.github.io/source/illustrations/lock_granularity.jpg)

## golang中的锁

golang里面的锁有两个特性： 

1. 不支持嵌套锁 
2. 可以一个goroutine lock，另一个goroutine unlock

### Mutex

golang中的互斥锁定义在`src/sync/mutex.go`

```go
// A Mutex is a mutual exclusion lock.
// Mutexes can be created as part of other structures;
// the zero value for a Mutex is an unlocked mutex.
//
// A Mutex must not be copied after first use.
type Mutex struct {
    state int32  //各种状态：阻塞在mutex的goroutine数量，当前goroutine是否被唤醒，mutex占用状态
    sema  uint32 //信号量
}

const (
    mutexLocked = 1 << iota // mutex is locked
    mutexWoken
    mutexWaiterShift = iota
)
```

看上去也是使用信号量的方式来实现的。sema就是信号量，一个非负数；state表示Mutex的状态。mutexLocked表示锁是否可用（0可用，1被别的goroutine占用），mutexWoken=2表示mutex是否被唤醒，mutexWaiterShift=2表示统计阻塞在该mutex上的goroutine数目需要移位的数值。将3个常量映射到state上就是

```
state:   |32|31|...|3|2|1|
         \__________/ | |
               |      | |
               |      | mutex的占用状态（1被占用，0可用）
               |      |
               |      mutex的当前goroutine是否被唤醒
               |
               当前阻塞在mutex上的goroutine数
```

#### Lock

1. Lock函数的入口处先调用CAS尝试去获得锁，如果m.state==0，则将其置为1，并返回

2. 若m.state != 0，表示锁已经被占用，则进入一个for循环

3. for循环中：

   - 首先mutex的等待goroutine数目加1，也就是修改state

   - 然后判断awoke = true时，要将m.state的标志位取消掉

   - 最后，调用CAS试图将m.state设置成new的值。

   - - 如果m.state之前的值也就是old如果没有被占用，则表示当前goroutine拿到了锁，则break

     - 否则调用runtime_Semacquire函数，该函数作用其实就是semaphore的wait(&s)：如果*s<0，则将当前goroutine塞入信号量s关联的goroutine waiting list，并休眠。

#### Unlock

也是有个for循环：

1. 如果阻塞在该锁上的goroutine数目为0或者mutex处于lock或者唤醒状态，则返回。
2. 这里先将阻塞在mutex上的goroutine数目减一，然后将mutex置于唤醒状态。唤醒的是哪个goroutine？goroutine wait list其实是FIFO的，先进先出。

### RWMutex

读写锁是一种同步机制，允许多个读操作同时读取数据，但是只允许一个写操作写数据。读写锁要根据进程进入临界区的具体行为（读，写）来决定锁的占用情况。这样锁的状态就有三种了：读模式加锁、写模式加锁、无锁。

- 无锁。读/写进程都可以进入。
- 读模式锁。读进程可以进入。写进程不可以进入。
- 写模式锁。读/写进程都不可以进入。

互斥锁只容许一个进程/线程处于代码临界区，如果程序中对于共享数据的读取操作特别多，这样就很不合理了。所以对于共享数据的读**取操作比较多**的情况下采用读写锁。**读写锁对于读操作不会导致锁竞争和进程/线程切换**

#### 读写锁实现

读写锁的核心是记录reader个数。有很多种实现，下面介绍两种简单的实现。

##### two mutex

使用两个互斥锁和一个整数记录reader个数，一个互斥锁r用来保证reader个数r_cnt的更新，一个w用来保证writer互斥。伪代码如下：

```
//读锁
RLock()
    r.lock()
    r_cnt++
    if r_cnt==1:
        w.lock()
    r.unlock()

RUnlock()
    r.lock()
    r_cnt--
    if r_cnt==0:
        w.unlock()
    r.unlock()
//写锁
Lock()
    w.lock()

Unlock()
    w.unlock()
```

##### mutex and condition

另外一种常见的实现方式是使用mutex和condition variable，思路很简单：RLock的时候如果发现有writer，则阻塞到condition variable；Lock的时候如果发现有writer，则阻塞。

```
RLock()
    m.lock()
    while w:
        c.wait(m)
    r++
    m.unlock()

RUnlock()
    m.lock()
    r--
    if r==0:
        c.signal_all()
    m.unlock()

Lock()
    m.lock()
    while w || r > 0 :
        c.wait(m)
    w = true
    m.unlock()

Unlock()
    m.lock()
    w = false
    c.signal_all()
    m.unlock()
```

#### go中RWMutex的实现

读写锁的定义在sync/rwmutex.go文件中。

```go
type RWMutex struct {
    w           Mutex  // 互斥锁
    writerSem   uint32 // 写锁信号量
    readerSem   uint32 // 读锁信号量
    readerCount int32  // 读锁计数器
    readerWait  int32  // 获取写锁时需要等待的读锁释放数量
}
```

从定义中可以看出RWMutex内部使用了Mutex，mutex其实是用来对写操作进行互斥的。

实现上采用了mutex和condition variable结合的方法。读写互斥锁的实现比较有技巧性一些，大体思路：

1. 读锁不能阻塞读锁，引入readerCount实现
2. 读锁需要阻塞写锁，直到所以读锁都释放，引入readerSem实现
3. 写锁需要阻塞读锁，直到所以写锁都释放，引入wirterSem实现
4. 写锁需要阻塞写锁，引入Metux实现

## 总结

1. 锁分为：互斥锁、读写锁、自旋锁（互斥锁的一种实现）
2. 锁的实现：硬件支持、信号量、互斥量（信号量的一种）
3. 互斥锁实现：忙等待（自旋锁）、阻塞自己（上下文切换）
4. 读写锁实现：两个mutex分别保护reader和writer、mutex和cond
5. golang中的Mutex是采用忙等待方式实现的。无冲突直接加锁，有冲突则进行自旋，因为大多数的Mutex保护的代码段都很短，经过短暂的自旋就可以获得
6. golang中的RWMutex的实现，引入了Mutex对writer进行加锁，还有计数变量

## 参考

[当我们谈论锁，我们谈什么](http://legendtkl.com/2016/10/13/about-lock/)

[golang中的锁源码实现：Mutex](http://legendtkl.com/2016/10/23/golang-mutex/)

[读写锁以及golang的实现](http://legendtkl.com/2016/10/26/rwmutex/)

[golang RWMutex读写锁分析](http://www.cnblogs.com/zongjiang/p/6577635.html)