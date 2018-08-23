---
title: Linux内存管理 - 4. malloc的实现
date: 2018-05-28
tags: Linux内核 内存管理
---

[TOC]

## 内存管理概述

### 方法

#### C风格

C 风格的内存管理程序主要实现 malloc()和 free()函数。内存管理程序主要通过调用 brk() 或者 mmap()进程添加额外的虚拟内存。 ptmalloc，BSD malloc， TCMalloc都属于这一类内存管理程序。

 如果有大量的不固定的内存引用，经常难以知道它们何时被释放。因为管理内存的问题，很多 程序倾向于使用它们自己的内存管理规则。 

#### 内存池

内存池是一种半内存管理方法。内存池帮助某些程序进行自动内存管理，这些程序会经 历一些特定的阶段，而且每个阶段中都有分配给进程的特定阶段的内存。 

优缺点：

- 可以预先分配，分配回收更快
- 程序结构发生变化，需要重新设计内存池

#### 引用计数

在引用计数中，所有共享的数据结构都有一个域来包含当前活动“引用”结构的次数。 当向一个程序传递一个指向某个数据结构指针时，该程序会将引用计数增加 1。 

优缺点：

- 简单易用
- 不要忘记调用引用计数函数
- 多线程中要加锁，难以使用

#### 垃圾回收

垃圾收集(Garbage collection)是全自动地检测并移除不再使用的数据对象。垃圾收集 器通常会在当可用内存减少到少于一个具体的阈值时运行。通常，它们以程序所知的可用的 一组“基本”数据——栈数据、全局变量、寄存器——作为出发点。然后它们尝试去追踪通 过这些数据连接到每一块数据。收集器找到的都是有用的数据;它没有找到的就是垃圾，可 以被销毁并重新使用这些无用的数据。 为了有效地管理内存，很多类型的垃圾收集器都需要 知道数据结构内部指针的规划，所以，为了正确运行垃圾收集器，它们必须是语言本身的一 部分。 

优缺点：

- 永远不必担心内存的双重释放或者对象的生命周期

- 使用大部分收集器时，您都无法干涉何时释放内存

### 设计目标

#### 浪费最小空间

内存管理器必然要使用一些自己的数据机构，这本身也要占用内存

碎片率是重要考虑的点

#### 分配释放速度

内存分配/释放是常用的操作。按着 2/8 原则，常用的操作就是性能热点，热点函数的 性能对系统的整体性能尤为重要。

#### 局部性

内存管理算法要考虑访问内存的局部性，减少cache miss和page fault

#### 其他

兼容性，移植性等

### 常见C内存管理程序

ptmalloc（glibc）、tcmalloc、jemalloc

Glibc 在内存回收方面做得不太好，常见的一个问题，申请很多内存，然后又释放，只是有一小块没释放，这时候 Glibc 就必须要等待这一小块也释放了，也把整个大块释放，极端情况下，可能会造成几个G的浪费。 

## brk和mmap

linux系统向用户提供申请的内存有brk(sbrk)和mmap函数。

1. brk(sbrk)的作用主要是扩展heap的上界brk，也就是说在进程空间的heap中申请内存

2. mmap有两种用法：

   - 映射磁盘文件到内存中
   - 匿名映射，匿名映射不映射磁盘文件，而是向映射区申请一块内存

   这里使用的是mmap是第二种用法，即向进程空间的映射区申请内存

申请小内存（<128KB），malloc使用sbrk分配内存；申请大内存（>128KB），使用mmap函数申请内存；但是这只是分配了虚拟内存，还没有映射到物理内存，当访问申请的内存时，才会因为缺页异常，内核分配物理内存。

## malloc的实现

### 原理

malloc采用的是内存池的实现方式，先申请一大块内存，然后将内存分成不同大小的内存块，然后用户申请内存时，直接从内存池中选择一块相近的内存块即可。

#### chunk

malloc利用chunk结构来管理内存块，malloc就是由不同大小的chunk链表组成的。chunk结构：

![img](/Users/sinnera/sinnera.github.io/source/illustrations/chunk.png)

chunk中包含一些控制信息，用这样的方法来记录分配的信息，以便完成分配和释放工作。chunk指针指向chunk开始的地方，图中的mem指针才是真正返回给用户的内存指针。

chunk 的第二个域的3个标记位：

1. **P**：上一个chunk是否在使用中
2. **M**：chunk是否是mmap得到的，还是heap得到的
3. **A**：chunk是不是属于主分配区

#### chunk bins

malloc将内存分成了大小不同的chunk，然后通过bins来组织起来，先来看下bin结构：

![bin结构](http://7xjnip.com1.z0.glb.clouddn.com/ldw-%E9%80%89%E5%8C%BA_0193.png) 

malloc将相似大小的chunk用双向链表链接起来，这样一个链表被称为一个bin。malloc一共维护了128个bin，并使用一个数组来存储这些bin。

##### unsorted bin

unsorted bin 的队列使用 bins 数组的第一个

如果被用户释放的 chunk 大于 max_fast,或者 fast bins 中的空闲 chunk 合并后,这些 chunk 首先会被放到 unsorted bin 队列中,在进行 malloc 操作的时候,如果在 fast bins 中没有找到合适的 chunk,则malloc 会先在 unsorted bin 中查找合适的空闲 chunk,然后才查找 bins。

如果 unsorted bin 不能满足分配要求。 malloc便会将 unsorted bin 中的 chunk 加入 bins 中。然后再从 bins 中继续进行查找和分配过程。

从这个过程可以看出来,unsorted bin 可以看做是 bins 的一个缓冲区,增加它只是为了加快分配的速度。

#####small large bin

数组从2开始编号，前64个bin为small bins，同一个small bin中的chunk具有**相同的大小**。两个相邻的small bin中的chunk大小相差8bytes。

small bins后面的bin被称作large bins。large bins中的每一个bin分别包含了**一个给定范围内的chunk**，其中的chunk按大小序排列，大小相同的则按照最近使用时间排列，这样要搜索一个可用的内存时，就在bins里按大小搜索，返回一个最小可用的chunk。相邻large bin的每个bin相差64字节。

##### fast bin

malloc除了有unsorted bin，small bin,large bin三个bin之外，还有一个fast bin。

一般的情况是,程序在运行时会经常需要申请和释放一些较小的内存空间。当分配器合并了相邻的几个小的 chunk 之后,也许马上就会有另一个小块内存的请求,这样分配器又需要从大的空闲内存中切分出一块,这样无疑是比较低效的。

因此，malloc 中在分配过程中引入了 fast bins,不大于 max_fast(默认值为 64B)的 chunk被释放后,首先会被放到 fast bins中,fast bins 中的 chunk 并不改变它的使用标志 P。这样也就无法将它们合并,当需要给用户分配的 chunk 小于或等于 max_fast 时,malloc 首先会在 fast bins 中查找相应的空闲块,然后才会去查找 bins 中的空闲 chunk。

在某个特定的时候,malloc 会遍历 fast bins 中的 chunk,将相邻的空闲 chunk 进行合并,并将合并后的 chunk 加入 unsorted bin 中,然后再将 usorted bin 里的 chunk 加入 bins 中。

#### top chunk

当fast bin和bins都不能满足内存需求时，malloc会设法在top chunk中分配一块内存给用户；top chunk为在**mmap区域分配一块较大的空闲内存**模拟sub-heap

#### mmap

当chunk足够大，fast bin和bins都不能满足要求，甚至top chunk都不能满足时，malloc会从mmap来直接使用内存映射来将页映射到进程空间，这样的chunk释放时，直接解除映射，归还给操作系统。

####策略

##### 小内存

采用chunk bins进行小内存管理。

在free的时候，会检查附近的chunk，如果标志位为P，即空闲中，会尝试把连续空闲的chunk合并成一个大的chunk，放到unstored bin里。

但是当很小的chunk释放的时候，ptmalloc会把它并入fast bin中。同样，某些时候，fast bin里的连续内存块会被合并并加入到一个unsorted bin里，然后再才进入普通bin里。

所以malloc小内存的时候，是**先查找fast bin，再查找unsorted bin，最后查找普通的bin，如果unsorted bin里的chunk不合适，则会把它扔到bin里**。 

##### 大内存

分配的内存顶部还有一个top chunk（heap中分配的大块内存），如果前面的bin里的空闲chunk都不足以满足需要，就是尝试从top chunk里分配内存。如果top chunk里也不够，就要从操作系统里拿了，这样会造成heap增大。
还有就是特别大的内存，会直接从系统mmap出来，不受chunk管理，这样的内存在回收的时候也会munmap还给操作系统。

### 过程

1. 一开始时，brk和start_brk是相等的，这时实际heap大小为0；如果第一次用户请求的内存大小小于mmap分配阈值，则malloc会申请(chunk_size+128kb) align 4kb大小的空间作为初始的heap。
2. 初始化heap之后，第二次申请的内存如果还是小于mmap分配阈值时，malloc会先查找fast bins,如果不能找到匹配的chunk，则查找small bins。若还是不行，合并fast bins,把chunk 加入到unsorted bin，在unsorted bin中查找，若还是不行，把unsorted bin中的chunk全加入large bins中，并查找large bins。
3. 若以上都失败了，malloc则会考虑使用top chunk。若top chunk也不能满足分配，且所需的chunk大小大于mmap分配阈值，则使用mmap进行分配。否则增加heap，增加top chunk，以满足分配要求。

### 总结

1. 小内存，大内存分别处理
2. 小内存：**先查找fast bin，再查找unsorted bin，最后查找普通的bin，如果unsorted bin里的chunk不合适，则会把它扔到bin里**
3. 大内存：小于128KB，优先从top chunk里分配，不够从heap中分配；超过128KB，从mmap中分配，并直接归还
4. fast bin和unsorted bin类似于一级二级缓存，unsorted bin已经起到了缓存作用，为何还需要fast bin。那是因为，在频繁申请小内存的场景下，会导致频繁的合并拆分，因此有了fast bin，释放到fast bins 中的空闲chunk并不会马上合并，而是定期合并。

## 参考

[malloc实现原理](http://luodw.cc/2016/02/17/malloc/)

[几种malloc实现原理 ptmalloc(glibc) && tcmalloc(google) && jemalloc(facebook)](https://blog.csdn.net/huangynn/article/details/50700093)