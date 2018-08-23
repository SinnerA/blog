---
title: 
date: 2018-05-19
tags: Linux
---

[TOC]

Linux中的进程由task_struct表示，Linux中线程也是由进程实现的，也就是说线程实际上也是一个task_struct。task_struct也被称为**进程描述符**，它包含一个进程运行所需的所有信息，比如进程的id、进程的属性以及构建进程的资源。 

![task_struct](https://github.com/SinnerA/blog/blob/master/illustrations/772759-20170127115856159-530461958.png)

## thread_info

进程在内核态运行时需要自己的堆栈信息，因此linux内核为每个进程都提供了一个**内核栈**

```c
struct task_struct
{
    // ...
    void *thread_info;    //指向内核栈的指针(新的内核是stack)
    //void *stack
    // ...
};
```

内核态的进程访问处于内核数据段的栈，这个栈不同于用户态的进程所用的栈。

注意：

- 内核态的栈在task_struct->thread_info（task_struct->stack）里面描述，其底部是thread_info对象，thread_info可以用来快速获取task_struct对象。整个stack区域一般只有一个内存页(可配置)，32位机器也就是4KB。

- 用户空间的堆、栈，在task_struct->mm->vm_area里面描述，都是属于进程虚拟地址空间的一个区域。

- C语言书里面讲的堆、栈大部分都是用户态的概念，用户态的堆、栈对应用户进程虚拟地址空间里的一个区域，栈向下增长，堆用malloc分配，向上增长。

### 为什么需要thread_info

linux内核是支持不同体系的的，但是不同的体系结构可能进程需要存储的信息不尽相同，这就需要我们实现一种通用的方式，我们将体系结构相关的部分和无关的部门进行分离。task_struct是一种通用的描述进程的结构，而thread_info就保存了特定体系结构的汇编代码段需要访问的那部分进程的数据。

通过thread_info能方便直接找到进(线)程的task_struct指针，由于thread_info结构体恰好位于内核栈的低地址开始处，所以只要知道内核栈的起始地址，就可以通过其得到thread_info，进而得到task_struct。

### thread_union

对每个进程，Linux内核都把两个不同的数据结构紧凑的存放在一个单独为进程分配的内存区域中：

- 一个是内核态的进程栈stack
- 另一个是紧挨着进程描述符的小数据结构thread_info，叫做线程描述符

这两个结构被紧凑的放在一个联合体中thread_union中,

```c
union thread_union
{
    struct thread_info thread_info;
    unsigned long stack[THREAD_SIZE/sizeof(long)];
};
```

这块区域32位上通常是8K=8192（占两个页框），64位上通常是16K，其实地址必须是8192的整数倍。

进程task_struct中的thread_info（或stack）指针指向了进程的thread_union的地址，在早期的内核中这个指针用`struct thread_info *thread_info`来表示, 但是新的内核中用了一个更浅显的名字`void *stack`，即内核栈。

![thread_info](https://github.com/SinnerA/blog/blob/master/illustrations/thread_info1.png)

上面我们知道通过thread_info可以轻松找到task_struct，那一开始怎么找到thread_info呢？答案是通过esp寄存器+偏移量。esp寄存器用来存放内核栈栈顶单元的地址。在80x86系统中，栈起始于顶端，并朝着这个内存区开始的方向增长。从用户态刚切换到内核态以后，进程的内核栈总是空的。因此，esp寄存器指向这个栈的顶端。一旦数据写入堆栈，esp的值就递减。

由于放在栈底，栈溢出时`thread_info`结构首先受影响

### current

所有的体系结构都必须实现两个current和current_thread_info的符号定义宏或者函数

- current_thread_info可获得当前执行进程的thread_info实例指针, 其地址可以根据内核指针来确定, 因为thread_info总是位于起始位置

  因为每个进程都有自己的内核栈, 因此进程到内核栈的映射是唯一的, 那么指向内核栈的指针通常保存在一个特别保留的寄存器中(多数情况下是esp)

- current给出了当前进程进程描述符task_struct的地址，该地址往往通过current_thread_info来确定 current = current_thread_info()->task

因此我们的关键就是current_thread_info的实现了，即如何通过**esp栈指针**来获取当前在CPU上正在运行进程的thread_info结构。各个版本的实现不太一样，早期的版本中，不需要对64位处理器的支持，所以，内核通过简单的屏蔽掉esp的低13位有效位就可以获得thread_info结构的基地址了。

### 总结

1. task_struct中的thread_info（stack）指向了thread_union（内核栈+thread_info），thread_union中的thread_info也指向了task_struct，并且它里面保存了一些跟体系结构相关的信息，current宏就是通过它实现的，能快速找到当前正在执行的进程，也就是task_struct。
2. slab动态生成`task_struct`时，只需在栈底创建一个新的`thread_info`结构。每个task的`thread_info`结构在内核栈尾端分配，其中task域指向任务实际的`task_struct`，`task_struct`的stack域指向`thread_info`结构
3. 使用`thread_info`结构的目的是，便于在汇编中计算地址，对`task_struct`进行访问。如在x86中，栈大小为8K，则栈指针直接右移13位得到`thread_info`的偏移，从而能够方便的访问到`thread_info`结构，进一步访问`task_struct`，这个就是内核通过`current`宏访问当前进程描述符`task_struct`的过程 

## mm

## tty

## fs

## files

## 参考

[Linux内核初探 之 进程(一) —— 进程基础](https://dupengair.github.io/2016/10/24/linux%E5%86%85%E6%A0%B8%E5%AD%A6%E4%B9%A0-%E5%9F%BA%E7%A1%80%E7%AF%87-Linux%E5%86%85%E6%A0%B8%E5%88%9D%E6%8E%A2-%E4%B9%8B-%E8%BF%9B%E7%A8%8B-%E4%B8%80-%E2%80%94%E2%80%94-%E8%BF%9B%E7%A8%8B%E5%9F%BA%E7%A1%80/)

[Linux进程内核栈与thread_info结构详解--Linux进程的管理与调度（九）](https://blog.csdn.net/gatieme/article/details/51577479)

