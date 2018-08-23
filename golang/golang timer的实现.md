---
title: golang timer的实现
date: 2018-05-16
tags: 
  - golang
---

[TOC]

## 介绍

定时事件是网络编程中经常遇到的，用于定时执行某些任务，比如定期检测一个客户连接的活动状态。网络编程中三大事件：IO事件、信号、定时事件。

服务器程序一般管理着大量定时事件，如何有效地组织这些定时事件，对服务器性能有着至关重要的影响。为此，我们要将每个定时事件分别封装成**定时器**，并使用某种容器类数据结构，比如**升序链表，时间堆和时间轮**。定时器通常至少要包含两个成员：**一个超时时间和一个任务回调函数**。

在golang中，可以使用Sleep()来暂停goroutine，其实sleep就是一个定时事件，底层被封装成了定时器，也就是timer

## 实现

### 数据结构

timer的实现主要位于runtime/time.go文件中。

```go
//定时器
type timer struct {
	i int // heap index

	// Timer wakes up at when, and then at when+period, ... (period > 0 only)
	// each time calling f(arg, now) in the timer goroutine, so f must be
	// a well-behaved function and not block.
    when   int64 //超时时间(绝对时间)
	period int64
	f      func(interface{}, uintptr) //回调函数
	arg    interface{}                //函数参数
	seq    uintptr
}

//定时器容器
var timers struct {
	lock         mutex //保护增删动作
	gp           *g    //不断循环检查堆
	created      bool
	sleeping     bool
	rescheduling bool
	waitnote     note
	t            []*timer //最小堆
}
```

Timer是定时器，Timers是维护所有Timer的一个集合，就是上面提到的定时器容器。除了可以向Timers中添加Timer外，还要从Timers中删除超时的Timer。这里，Timers采用**小顶堆**来维护，效率更高的有时间轮。

**Timers**

- Timers结构中有一个`Lock`, 用来保护`添加/删除`Timer的。
- `gp`指针维护的是一个goroutine，这个goroutine的主要功能就是检查小顶堆中的Timer是否超时。当然，超时就是删除Timer，并且执行Timer对应的回调函数。
- `t`显然就是存储所有Timer的堆了。

省略几个字段放到下文再介绍。

**Timer**

- `when`就是定时器超时的时间
- `f`和`arg`挂载的是Timer超时后需要执行的函数和函数参数。

### 执行过程

Timers里面的gp，是一个独立执行的goroutine：不选循环检查堆，删除掉那些超时的Timer，并执行对应的回调函数。看下过程：

```go
static void
timerproc(void)
{
    for(;;) {
        for(;;) {
            // 判断Timer是否超时
            t = timers.t[0];
            delta = t->when - now;
            if(delta > 0)
                break;

            // TODO: 删除Timer, 代码被删除

            // 这里的f调用就是执行Timer了
            f(now, arg);
        }

        // 这个过程是，堆中没有任何Timer的时候，就把这个goroutine给挂起，不运行。
        // 添加Timer的时候才会让它ready。
        if(delta < 0) {
            // No timers left - put goroutine to sleep.
            timers.rescheduling = true;
            runtime·park(runtime·unlock, &timers, "timer goroutine (idle)");
            continue;
        }

        // 这里干的时候就让这个goroutine也sleep, 等待最近的Timer超时，再开始执行上面的循环检查。当然，这里的sleep不是用本文的定时器来实现的，而是futex锁实现。
        // At least one timer pending.  Sleep until then.
        timers.sleeping = true;
        runtime·notetsleep(&timers.waitnote, delta);
        }
    }
}
```

具体过程：

1. 判断堆中是否有Timer? 如果没有就将`Timers`的`rescheduling`设置为true的状态，true就代表`gp` goroutine被挂起，需要重新调度。这个重新调度的时刻就是在添加一个Timer进来的时候，会ready这个goroutine。这里挂起goroutine使用的是runtime·park()函数。
2. 如果堆中有Timer存在，就取出堆顶的一个Timer，判断是否超时。超时后，就删除Timer，执行Timer中的回调函数。这一步是循环检查堆，直到堆中没有Timer或者没有超时的Timer为止。
3. 在堆中的Timer还没超时之前，这个goroutine将处于sleep状态，也就是设置`Timers`的`sleeping`为true状态。这个地方是通过runtime·notesleep()函数来完成的，其实现是依赖futex锁。这里，goroutine将sleep多久呢？它将sleep到最近一个Timer超时的时候，就开始执行。

### 添加定时器

```go
static void
addtimer(Timer *t)
{
    if(timers.len >= timers.cap) {
        // TODO 这里是堆没有剩余的空间了，需要分配一个更大的堆来完成添加Timer。
    }

    // 这里添加Timer到堆中.
    t->i = timers.len++;
    timers.t[t->i] = t;
    siftup(t->i);

    // 这个地方比较重要，这是发生在添加的Timer直接位于堆顶的时候，堆顶位置就代表最近的一个超时Timer.
    if(t->i == 0) {
        // siftup moved to top: new earliest deadline.
        if(timers.sleeping) {
            timers.sleeping = false;
            runtime·notewakeup(&timers.waitnote);
        }
        if(timers.rescheduling) {
            timers.rescheduling = false;
            runtime·ready(timers.timerproc);
        }
    }
}
```

从代码可以看到新添加的Timer如果是堆顶的话，会检查`Timers`的sleeping和rescheduling两个状态。上面已经提过了，这两个状态代表`gp` goroutine的状态，如果处于sleeping，那就wakeup它；如果是rescheduling就ready它。这么做的原因就是通知那个wait的goroutine——”堆中有一个Timer了”或者”堆顶的Timer易主了”，你赶紧来检查一下它是否超时。

### 示例：sleep的实现

```go
void
runtime·tsleep(int64 ns, int8 *reason)
{
    Timer t;

    if(ns <= 0)
        return;

    t.when = runtime·nanotime() + ns;
    t.period = 0;
    t.fv = &readyv; //回调函数就是唤醒goroutine
    t.arg.data = g;
    runtime·lock(&timers);
    addtimer(&t);
    runtime·park(runtime·unlock, &timers, reason);
}
```

Sleep()实现总结起来就三大步:

1. 创建一个Timer添加到Timers中
2. 挂起当前goroutine
3. Timer超时ready当前goroutine

## 坑

### 临时timer

Go 语言的 channel 本身是不支持 timeout 的，所以一般实现 channel 的读写超时都采用 select，如下： 

```go
func demo(input chan interface{}) {
    for {
        select {
        case msg <- input:
            println(msg)

        case <-time.After(time.Second * 5):
            println("5s timer")

        case <-time.After(time.Second * 10):
            println("10s timer")
        }
    }
}

go demo(input)
```

写出上面这段程序的目的是从 input channel 持续接收消息加以处理，同时希望每过5秒钟和每过10秒钟就分别执行一个定时任务。

但是当你执行这段程序的时候，只要 input channel 中的消息来得足够快，永不间断，你会发现启动的两个定时任务都永远不会执行；即使没有消息到来，第二个10s的定时器也是永远不会执行的。

原因就是 select 每次执行都会重新执行 case 条件语句，并重新注册到 select 中，因此这两个定时任务在每次执行 select 的时候，都是**启动了一个新的从头开始计时的 Timer 对象**，所以这两个定时任务**永远不会执行**。

怎么解决？很简答，定义两个全局的 Timer，每次执行 select 都重复使用这两个 Timer，而不是每次都生成全新的。这样才可以真正做到在接收消息的同时，还能够定时的执行相应的任务。

```go
func demo(input chan interface{}) {
    t1 := time.NewTimer(time.Second * 5)
    t2 := time.NewTimer(time.Second * 10)

    for {
        select {
        case msg <- input:
            println(msg)

        case <-t1.C:
            println("5s timer")
            t1.Reset(time.Second * 5)

        case <-t2.C:
            println("10s timer")
            t2.Reset(time.Second * 10)
        }
    }
}
```

还是上面那个例子，由于每次执行select语句，都是一个新定时器，for循环中如果事件来的很快，定时器没有超时时，timers里面将保留了大量的定时器，而且时间设置的越长，维护的定时器数量就越多，产生了大量的临时对象。解决方法就是改成全局定时器。

总结：如果case中使用临时定时器，每次select都是生成新的定时器，不但定时器得不到准确执行，还会存在大量临时定时器对象。

## 参考

[timer in Go's runtime](http://skoo.me/go/2013/09/12/go-runtime-timer)
