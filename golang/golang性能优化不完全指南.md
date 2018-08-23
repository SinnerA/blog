---
title: golang性能优化不完全指南
date: 2018-07-22
tags: Golang 性能优化
---

[TOC]

## 概述

首先，Go的性能优化不能脱离业务场景，实际场景中往往只有一些关键点的优化能显著提升性能，其他大部分优化带来的提升效果不明显，这就是二八原则，优化的太细致有时反而影响代码可读性。因此如何进行性能优化，不仅是个技术活，还要权衡其他方面：业务场景、代码可读性、二八原则。

一般的，项目各项业务功能稳定之后，可以通过模拟线上情况进行压测，通过压测来暴露性能问题。需要关注的主要指标：Latency、QPS、Throughput，影响这些指标的因素无非就是：CPU、内存、网络、IO。

在实践中，进行性能优化的方案：

1. 业务场景的优化：同步变异步、单个变批量、lazy模式等
2. 避免代码层面的陷阱：GC、锁、goroutine阻塞等
3. 使用工具定位问题：go runtime提供的`pprof`、`trace`和`GODEBUG`，调试工具dlv等

## 业务场景

业务场景的优化，要根据实际场景出发，必须对业务上下游都非常熟悉，热点模块要针对性优化。

一般来说，只要设计合理的架构和业务模块的分层，基本就决定了性能不会是太大问题，有问题也容易优化和扩展。因此，前期的设计工作非常重要。

开发实践中遇到的几个场景：

1. 每当发出一条群聊消息，需要notify给所有群成员，之前是单个notify，导致吞吐率很低，且压测时CPU占用很高。之后观察得知，因为业务模块与notify模块是RPC调用，序列化和反序列化消耗了大量的CPU。改成批量notify之后，吞吐率提升了一个量级。
2. redis存储session改成pipeline模式，提高了效率
3. kafka在produce时同步变异步，提高了kafka的吞吐率
4. 对话和消息的拉取，采用lazy模式，过期了或者接受到notify之后，再去主动拉取

业务场景的优化跟具体项目有关，本文主要关注golang性能优化，这里就不再做过多介绍。

## 代码优化

代码优化一般情况下从GC和锁两方面进行优化，还有要注意一些常见的坑。

### GC

内存的频繁申请和释放会触发GC，因此应该尽量避免频繁创建临时堆对象（如&abc{}, new, make等）以减少垃圾收集时的扫描时间。

> 下一次垃圾回收发生在程序被分配了一块与其当前所用内存成比例的额外内存时。这个比例通常是由 GOGC 的环境变量（默认值是100）控制的。如果 GOGC=100，而且程序使用了 4M 堆内存，当程序使用达到 8M 时，运行时（runtime）就会再次触发垃圾回收器。这使垃圾回收的消耗与分配的消耗保持线性比例。调整 GOGC，会改变线性常数和使用的额外内存的总量。
>
> 只有清除是依赖于堆总量的，且清除与正常的程序运行同时发生。如果你可以承受额外的内存开销，设置 GOGC 到以一个较高的值（200, 300, 500,等）是有意义的。例如，GOGC=300 可以在延迟相同的情况下减小垃圾回收开销高达原来的二分之一（但会占用两倍大的堆）。
>
> GC 是并行的，而且一般在并行硬件上具有良好可扩展性。所以给 GOMAXPROCS 设置较高的值是有意义的，就算是对连续的程序来说也能够提高垃圾回收速度。但是，要注意，目前垃圾回收器线程的数量被限制在 8 个以内。

总体原则：减少对象分配个数（一次分配够，小对象合并成大对象，对象重用），频繁使用的对象避免使用指针

一般性建议：

1. make已知大小的slice和map时，应该指定容量，避免append时触发内存分配
2. 使用标准库包含的 [sync.Pool](http://tip.golang.org/pkg/sync/#Pool) 类型可以实现垃圾回收期间多次重用同一个对象
3. 将小对象组合成大对象（比如struct），减少了内存分配次数并且降低了GC的压力（更快）。然而，这样的优化方式可能会影响代码的可读性，因此要合理地使用它。
4. 够用的情况下，尽可能使用小的数据类型，比如用int8代替int。
5. 不包含任何指针的对象(注意 strings,slices,maps 和 chans 包含隐含指针)，将不会被垃圾回收器扫描到。比如，1GB 的slice实际上不会影响垃圾回收时间。因此被频繁使用的对象尽量避免使用指针，这样能减少GC的时间。可以使用索引替换指针，将对象分割为其中之一不含指针的两部分。

### 锁

锁是用来保护临界区的，在高并发场景下，锁竞争的开销会造成延迟，锁的使用应当小心谨慎，能不用则不用。

总体原则：尽量不用锁，减少锁的次数，减小锁的粒度

一般性建议：

1. 针对于主要为读取的对象，使用sync.RWMutex而不是sync.Mutex，并发读不需要加锁。

2. 可以使用CAS的场景，尽量用CAS代替锁，减少锁的使用。不过，在被操作对象被频繁变更的情况下，CAS操作并不那么容易成功，这时可以利用for循环多次尝试。

3. 在某些情况下，可以使用COW来实现无锁更新和访问，golang的指针赋值是原子操作，不需要加锁。如果受保护的数据结构很少被修改，可以为它制作一份副本，然后就可以这样更新它：

   ```go
   type Config struct {
       Routes   map[string]net.Addr
       Backends []net.Addr
   }
   
   var config unsafe.Pointer  // actual type is *Config
   
   // Worker goroutines use this function to obtain the current config.
   func CurrentConfig() *Config {
       return (*Config)(atomic.LoadPointer(&config))
   }
   
   // Background goroutine periodically creates a new Config object
   // as sets it as current using this function.
   func UpdateConfig(cfg *Config) {
       atomic.StorePointer(&config, unsafe.Pointer(cfg))
   }
   ```

   这种模式可以防止写入阻塞读取。

4. 分割是另一种用于减少共享可变数据结构竞争和阻塞的通用技术。下面是一个展示如何分割哈希表（hashmap）的例子：

   ```go
   type Partition struct {
       sync.RWMutex
       m map[string]string
   }
   
   const partCount = 64
   var m [partCount]Partition
   
   func Find(k string) string {
       idx := hash(k) % partCount
       part := &m[idx]
       part.RLock()
       v := part.m[k]
       part.RUnlock()
       return v
   }
   ```

5. 本地缓存和更新的批处理有助于减少对不可分解的数据结构的争夺，批处理减少了锁的次数。下面你将看到如何分批处理向通道发送的内容：

   ```go
   const CacheSize = 16
   
   type Cache struct {
       buf [CacheSize]int
       pos int
   }
   
   func Send(c chan [CacheSize]int, cache *Cache, value int) {
       cache.buf[cache.pos] = value
       cache.pos++
       if cache.pos == CacheSize {
           c <- cache.buf
           cache.pos = 0
       }
   }
   ```

   这种技术并不仅限于通道，它还能用于批量更新映射（map）、批量分配等等。

### goroutine

#### 阻塞

阻塞有时候是正常现象，当一个goroutine阻塞时，底层的工作线程就会简单地转换到另一个goroutine，这并不影响性能。time.Ticker和sync.WaitGroup上发生的阻塞都是正常的，而发生在sync.Mutex或sync.RWMutex上的阻塞通常是不利的。

goroutine阻塞会导致：

1. 程序与处理器之间不成比例，原因是缺乏工作。调度器追踪工具可以帮助确定这种情况。
2. 过多的goroutine阻塞/解除阻塞消耗了CPU时间。CPU分析器可以帮助确定这种情况（在系统组件中找）。

goroutine数量太多也不行，会导致耗尽内存资源，或者CPU使用率过高以至于系统无法工作，因此goroutine数量也需要控制。换句话说，到达一定数量之后需要主动阻塞后面goroutine的执行，可以**用缓存channel实现**。

一般性建议：

1. 在生产者消费者情景中使用充足的缓冲chan，无缓冲的chan限制了程序的并发性。
2. goroutine数量太多时，使用channel控制goroutine数量

#### 泄露（Leak）

goroutine永远堵塞在channel上，或者陷入死循环，那么 goroutine 会发生泄露，跟内存不一样，泄露的goroutine不会被回收，将占用大量内存，导致系统阻塞。

举一个例子：

```go
ch := make(chan T)
go produce(ch) {
  // 生产者往ch里写数据
  ch <- T{}
}
go consume(ch) {
  // 消费者从ch里读出数据
  <-ch
  err := doSomeThing()
}
```

消费者发生err并退出了，不再读ch，导致生产者阻塞在`ch <- T{}`上面，然后生产者的goroutine就泄漏了，里面引用的内存永远无法释放。这是一个十分常见的场景，比如说开多个worker，worker处理好数据后写channel，主线程读channel，但是主线程处理过程中出错退出，如果处理不当worker就可能泄漏的。

导致goroutine泄露的主要原因：

1. 发送到一个没有接收者的 channel
2. 从没有发送者的 channel 中接收数据
3. 写入或读取nil channel
4. 死循环

一般性建议：

1. 使用一个有足够缓存所有结果的chan。 

2. 绝对不能由消费者关chan，因为生产者在不知情的情况下向关闭的channel写数据会panic

   ```go
   func produce(ch chan<- T) {
       defer close(ch) // 生产者写完数据关闭channel
       ch <- T{}
   }
   func consume(ch <-chan T) {
       for _ = range ch { // 消费者用for-range读完里面所有数据
       }
   }
   ch := make(chan T)
   go produce(ch)
   consume(ch)
   ```

3. 利用close chan来广播取消动作。向关闭的chan读数据永远不会阻塞。

   ```go
   func produce(ch chan<- T, cancel chan struct{}) {
       select {
         case ch <- T{}:
         case <- cancel: // 用select同时监听cancel动作
       }
   }
   func consume(ch <-chan T, cancel chan struct{}) {
       v := <-ch
       err := doSomeThing(v)
       if err != nil {
           close(cancel) // 能够通知所有produce退出
           return
       }
   }
   for i:=0; i<10; i++ {
       go produce()
   }
   consume()
   ```

#### 竞态（Race）

竞争是指没有加锁，多个goroutine对同一个变量进行操作，发生了竞争关系，导致了不可预料的结果。

使用`go test -race`可以进行竞态检查

一般使用锁或者chan来避免竞态

### 其他

1. 用[]byte替代string
2. …...

## 性能分析

**性能分析不是没有开销的**。虽然性能分析对程序的影响并不严重，但是毕竟有影响，特别是内存分析的时候增加采样率的情况。大多数工具甚至直接就不允许你同时开启多个性能分析工具。如果你同时开启了多个性能分析工具，那很有可能会出现他们互相观察对方的开销从而导致你的分析结果彻底失去意义。

所以，**一次只分析一个东西**。

### pprof

[`pprof`](https://github.com/google/pprof) 源自 [Google Performance Tools](https://github.com/gperftools/gperftools/wiki) 工具集。Go runtime 中内置了 `pprof` 的性能分析功能。这包含了两部分：

- 使用每个 Go 程序中内置 `runtime/pprof` 包生成性能数据文件
- 然后用 [`go tool pprof`](https://blog.golang.org/profiling-go-programs) 来分析性能数据文件

#### 维度

##### CPU性能

最常用的就是 CPU 性能分析，当 CPU 性能分析启用后，Go runtime 会每 `10ms` 就暂停一下，记录当前运行的 Go routine 的调用堆栈及相关数据。当性能分析数据保存到硬盘后，我们就可以分析代码中的热点了。

一个函数如果出现在数据中的次数越多，就越说明这个函数调用栈占用了更多的运行时间。

##### 内存性能

内存性能分析则是在**堆(Heap)分配**的时候，记录一下调用堆栈。默认情况下，是每 `1000` 次分配，取样一次，这个数值可以配置。

**栈(Stack)分配**由于会随时释放，因此**不会**被内存分析所记录。

由于内存分析是**取样**方式，并且也因为其记录的**是分配内存，而不是使用内存**。因此使用内存性能分析工具来准确判断程序具体的内存使用是比较困难的。

内存分析器采取抽样的方式，也就是说，它仅仅从一些内存分配的子集中收集信息。有可能对一个对象的采样与被采样对象的大小成比例。你可以通过使用go test --memprofilerate标识，或者通过程序启动时的运行配置中的MemProfileRate变量来改变调整这个采样的比例。如果比例为1，则会导致全部申请的信息都会被收集，但是这样的话将会使得执行变慢。默认的采样比例是**每512KB**的内存申请就采样一次。

分析器倾向于在性能分析中对较大的对象采样。但是需要注意的是大的对象会影响内存消耗和垃圾回收时间，大量的小的内存分配会影响运行速度（某种程度上也会影响垃圾回收时间）。所以最好同时考虑它们。

看内存占用的时候最好从几个维度一起看：

1. 是程序占用的系统内存
2. Go的堆内存
3. 实际使用到的内存

从系统申请的内存会在Go的内存池管理，整块的内存页，长时间不被访问并满足一定条件后，才归还给操作系统。又因为有GC，堆内存也不能代表内存占用，清理过之后剩下的，才是实际使用的内存。

##### 阻塞性能

阻塞分析是一个很独特的分析。它有点类似于CPU性能分析，但它所记录的是goroutine等待资源所花的时间。

阻塞分析对分析程序**并发瓶颈**非常有帮助。阻塞性能分析可以显示出什么时候出现了大批的 goroutine 被阻塞了。阻塞包括：

- 发送、接受无缓冲的channel
- 发送给一个满缓冲的 channel，或者从一个空缓冲的 channel 接收
- 试图获取已被另一个 goroutine锁定的 `sync.Mutex` 的锁

阻塞性能分析是特殊的分析工具，在排除CPU和内存瓶颈前，不应该用它来分析。

#### 具体方法

##### 分析函数

对一个函数进行性能分析的办法就是使用 `testing` 包，也就是`go test -xxx`，其中xxx是选项：

-cpuprofile=xxxx：生成CPU性能分析数据，并写入文件xxxx

-memprofile=xxxx：生成内存性能分析数据，并写入文件xxxx

-memprofilerate=N：调整采样率为1/N

-blockprofile=xxxx：生成阻塞性能分析数据，并写入文件xxxx

举例:

```bash
//-run=XXX 是说只运行 Benchmarks，而不要运行任何 Tests
go test -run=XXX -bench=IndexByte -cpuprofile=/tmp/c.p bytes 

//分析生成的性能数据文件，go tool pprof (应用程序)（应用程序的prof文件）
go tool pprof bytes.test /tmp/c.p
```

> 执行go tool pprof，会进入到pprof的交互模式，执行web命令就会生成svg文件，svg文件是可以在浏览器下看的（需要安装graphviz）

##### 分析整个应用

如果想分析整个应用，则可以使用 `runtime/pprof` 包。当然这比较底层，Dave Cheney 在几年前还写了个包 [`github.com/pkg/profile`](https://github.com/pkg/profile)，可以简化使用。

只需要在启动的时候加入 `defer profile.Start().Stop()` 即可。

```go
import "github.com/pkg/profile"

func main() {
        defer profile.Start().Stop()
        ...
}
```

go tool pprof的使用跟上面一样。

##### 分析已经在运行的应用

`pprof` 适合在开发的时候进行分析，从运行到结束。

但是如果应用已经在服务器运行，我们希望远程启用调试进行在线分析，这种情况，可以通过 `http` 远程调试。

```go
import _ "net/http/pprof"

func main() {
        log.Println(http.ListenAndServe("localhost:3999", nil))
        ...
}
```

然后使用 `pprof` 工具来查看一段 `30秒` 的：

```bash
go tool pprof http://localhost:3999/debug/pprof/profile //CPU
go tool pprof http://localhost:3999/debug/pprof/heap    //内存
go tool pprof http://localhost:3999/debug/pprof/block   //阻塞
```

##### go tool pprof

go tool pprof命令适用于分析性能数据文件，生成报告，或者图形化指标。

需要注意的是，在分析内存时，可以将分配的**字节数**或者分配的**对象数**形象化（分别是以--inuse/alloc_space和--inuse/alloc_objects为标志）。

```bash
go tool pprof --inuse_space //字节数
go tool pprof --inuse_objects //对象数
```

大体上，如果你想减小内存消耗量，那么你需要查看程序正常运行时--inuse_space收集的概要。如果你想提升程序的运行速度，就要查看在程序特征运行时间后或程序结束之后--alloc_objects收集的概要。

##### go torch

go-torch是`Uber`公司开源的一款针对Go语言程序的火焰图生成工具，比svg更加直观。

需要安装FlameGraph和go-torch，使用`go-torch`代替`go tool pprof`生成图形化报告：

```
go-torch -u http://localhost:9090 -t 30

go tool pprof --seconds 25 http://localhost:9090/debug/pprof/profile
```

一般的，`go-torch`更加广泛的被使用，因为它能够更加方便的定位问题。

### trace

在 Go 1.5的时候，[Dmitry Vyukov](https://github.com/dvyukov) 在 runtime 里添加了一个新的性能分析工具，[execution tracer profile](https://golang.org/doc/go1.5#trace_command)。

```
$ go test -trace=trace.out path/to/package
$ go tool trace [flags] pkg.test trace.out
```

这个工具可以用来分析程序动态执行的情况，并且分析的精度达到纳秒级别：

- goroutine创建、启动、结束
- gorouting阻塞、恢复
- 网络阻塞
- 系统调用（syscall）
- GC事件

这个工具目前介绍的文档不多，<https://golang.org/pkg/runtime/trace/>。只有很少量的文档，或许应该有一些文档、博客啥的描述一下。这里有几个文档可以看看：

- https://docs.google.com/document/u/1/d/1FP5apqzBgr7ahCCgFO-yoVhk4YZrNIDNf9RybngBc14/pub
- <https://www.dotconferences.com/2016/10/rhys-hiltner-go-execution-tracer>

通过 `-traceprofile=` 来标志生成性能分析文件。

比如这里我们构建包 `cmd/compile/internal/gc`，并生成构建的性能分析数据文件。

```
$ go build -gcflags=-traceprofile=/tmp/t.p cmd/compile/internal/gc
$ go tool trace /tmp/t.p
2017/10/23 20:59:09 Parsing trace...
2017/10/23 20:59:10 Serializing trace...
2017/10/23 20:59:10 Splitting trace...
2017/10/23 20:59:11 Opening browser
```

这会打开一个浏览器，显示：

- View trace
- Goroutine analysis
- Network blocking profile
- Synchronization blocking profile
- Syscall blocking profile
- Scheduler latency profile

点进去就可以看到各种分析。

![trace](https://blog.lab99.org/pics/golang/profile/trace.jpg)

另外，Dave Cheney做的 `pkg/profile` 也支持生成 trace profile 了。

```go
import "github.com/pkg/profile"

...

func main() {
        defer profile.Start(profile.TraceProfile).Stop()
        ...
}
```

### GODEBUG（追踪器）

除了性能分析工具以外，还有另外几种工具可用——追踪器。它们可以追踪垃圾回收，内存分配和goroutine调度状态。要启用追踪需要将GODEBUG=xxx=1加入环境变量，再运行程序。

要启用垃圾回收器（GC）追踪你需要将GODEBUG=gctrace=1加入环境变量，然后运行：

```shell
$ GODEBUG=gctrace=1 ./myserver
```

然后程序在运行中会输出类似结果：

```shell
gc9(2): 12+1+744+8 us, 2 -> 10 MB, 108615 (593983-485368) objects, 4825/3620/0 sweeps, 0(0) handoff, 6(91) steal, 16/1/0 yields
gc10(2): 12+6769+767+3 us, 1 -> 1 MB, 4222 (593983-589761) objects, 4825/0/1898 sweeps, 0(0) handoff, 6(93) steal, 16/10/2 yields
gc11(2): 799+3+2050+3 us, 1 -> 69 MB, 831819 (1484009-652190) objects, 4825/691/0 sweeps, 0(0) handoff, 5(105) steal, 16/1/0 yields
```

垃圾收集的信息都会被输出出来，可以帮助GC排障。如果发现GC一直都在很忙碌的工作，那恐怕内存管理上有可以改进的地方。

类似的:

内存跟踪：GODEBUG=allocfreetrace=1

groutine调度跟踪：GODEBUG=schedtrace=1000

随意组合：GODEBUG = gctrace = 1，allocfreetrace = 1，schedtrace = 1000

### dlv（delve）

[dlv](https://github.com/derekparker/delve)是一个专门为 Go而生的 debug 工具，最大的亮点就是对goroutine环境调试支持比较完善。

当程序正在运行时，可以通过attach上去进行调试。常用功能：

1. 设断点：b
2. 继续：c
3. 查看当前调用栈信息：bt
4. 输出变量信息：print  xxx
5. 查看所有goroutine的信息：goroutines
6. 查看具体某个gouroutine信息：goroutine x
7. 查看当前是在哪个goroutine上：goroutine

## 参考

[Profiling Go Programs](https://blog.golang.org/profiling-go-programs)

[Debugging performance issues in Go programs](https://software.intel.com/en-us/blogs/2014/05/10/debugging-performance-issues-in-go-programs)

[视频笔记：7种 Go 程序性能分析方法 - Dave Cheney](https://blog.lab99.org/post/golang-2017-10-20-video-seven-ways-to-profile-go-apps.html)

[Go的50度灰：Golang新开发者要注意的陷阱和常见错误](http://colobu.com/2015/09/07/gotchas-and-common-mistakes-in-go-golang/)

[Golang程序调试工具介绍(gdb vs dlv)](http://lday.me/2017/02/27/0005_gdb-vs-dlv/#)

[Goroutine陷阱](http://lanlingzi.cn/post/technical/2016/0703_goroutine/)

[Go程序内存泄漏的分析以及避免](http://www.zenlife.tk/go-leak.md)

[Goroutine 泄露](https://studygolang.com/articles/12495)