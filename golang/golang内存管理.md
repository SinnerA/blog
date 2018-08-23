---
title: golang内存管理
date: 2018-08-04
tags: 
  - golang
  - 内存管理
---

[TOC]

## 概述

#### 几个关键数据结构

- mspan 
  由mheap管理的页面，记录了所分配的块大小和起始地址等
- mcache 
  与P(可看做cpu)绑定的线程级别的本地缓存
- mcenter 
  全局空间的缓存，收集了各种大小(67种)的span列表
- mheap 
  分配内存的堆分配器，以8kb进行页管理
- fixalloc 
  固定尺寸的堆外对象空闲列表分配器，用来管理分配器的存储

#### 内存分配逻辑

- 如果object size>32KB, 则直接使用mheap来分配空间；
- 如果object size<16Byte, 则通过mcache的tiny分配器来分配(tiny可看作是一个指针offset)；
- 如果object size在上面两者之间，首先尝试通过sizeclass对应的分配器分配;
- 如果mcache没有空闲的span， 则向mcentral申请空闲块；
- 如果mcentral也没空闲块，则向mheap申请并进行切分；
- 如果mheap也没合适的span，则向系统申请新的内存空间。

#### 内存回收逻辑

- 如果object size>32KB, 直接将span返还给mheap的自由链；
- 如果object size<32KB, 查找object对应sizeclass， 归还到mcache自由链；
- 如果mcache自由链过长或内存过大，将部分span归还到mcentral；
- 如果某个范围的mspan都已经归还到mcentral，则将这部分mspan归还到mheap页堆；
- 而mheap不会定时将内存归还到系统，但会归还虚拟地址到物理内存的映射关系，当系统需要的时候可以回收这部分内存，否则暂时先留着给Go使用。

## 总结

1. 采用TCMalloc的思路，但是有所不同：

2. - 局部缓存不针对进程或线程，而是分配给P（goroutine）

   - 内存回收是STW，而不是针对线程进行回收
   - 小内存变量直接分配在栈，若内存不够则在堆分配，也有可能逃逸到堆上

3. 类似TCMalloc，分3层：mchache、mcentral、mheap

4. TCMalloc中有object和span两种内存单元；golang统一使用了mspan结构，只有一种可能，就是mspan中记录了需要分配的块大小，mspan中分割成了相同size-class的块，也就是说mspan其实相当于object的链表。

5. mchache中维护了不同size-class的mspan数组，也就是说每种size对应一个mspan，mspan中包含了多个该size的object

6. mheap中维护了不同size-class的mcentral数组，每个mcentral维护了一个mspan链表

7. 分配：<16B，mcache的tiny分配器分配，>16B && <32KB则从mcache的size-class中分配，否则直接从mheap分配

8. 回收：mchache将free的span回收给mcentral，mcentral把span回收给mheap，mheap 并不会定时向操作系统归还，但是会对 span 做一些操作，比如合并相邻的 span。

## 参考

[Go语言内存分配器的实现](http://skoo.me/go/2013/10/13/go-memory-manage-system-alloc)

[golang 内存管理 + 垃圾回收](https://blog.csdn.net/qq_17612199/article/details/80278632)

[探索Go内存管理(分配)](https://www.jianshu.com/p/47691d870756)