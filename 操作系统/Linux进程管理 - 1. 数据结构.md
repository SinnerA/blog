---
title: 
date: 2018-05-19
tags: 
---

[TOC]

进程是程序的一次执行过程，进程由PCB来描述，在Linux中就是task_struct。

PCB应该包含：

1. 进程标识符
2. 进程调度信息（状态，优先级，调度算法等）
3. 进程控制信息（进程栈，文件资源）
4. 上下文信息（通用寄存器，PC，PSW，用户栈指针）

## 数据结构

![task_struct](https://github.com/SinnerA/blog/blob/master/illustrations/task_struct.png)

### 进程状态

state：进程运行状态

![img](https://github.com/SinnerA/blog/blob/master/illustrations/process_state.png)

### 进程标识符

- pid：进程id（线程id）
- tgid：这是为了支持线程引入的，一个线程组所有的线程都有相同的tgid，就是该线程组第一个线程的pid，方便对该线程组做统一处理

注意：getpid()返回的是tgid（进程id），gettid()返回的是pid（线程就是线程id，进程就是进程id）

### 内核态进程栈

thread_info（stack）：指向的内核态的进程栈：thread_info，thread_info也指向task_struct，利用它可以轻易找到task_struct，current宏（找到当前执行进程）就是利用它实现的

![thread_info](https://github.com/SinnerA/blog/blob/master/illustrations/thread_info1.png)

**注：在进程切换时，内核栈保存了寄存器信息，用于恢复上下文**

### 亲属关系进程

- real_parent：父进程（亲爹），调用fork的进程，如果创建它的父进程不再存在，则指向PID为1的init进程
- parent：父进程（干爹），real_parent被kill之后，接管它的子进程，比如1号进程
- children：表示链表的头部，链表中的所有元素都是它的子进程

### 进程调度

- 优先级：

  | 字段        | 描述                                               |
  | ----------- | -------------------------------------------------- |
  | static_prio | 用于保存静态优先级，可以通过nice系统调用来进行修改 |
  | rt_priority | 用于保存实时优先级                                 |
  | normal_prio | 的值取决于静态优先级和调度策略                     |
  | prio        | 用于保存动态优先级                                 |

- 调度策略

  | 字段         | 描述                                           |
  | ------------ | ---------------------------------------------- |
  | policy       | 调度策略                                       |
  | sched_class  | 调度类                                         |
  | se           | 普通进程的调用实体，每个进程都有其中之一的实体 |
  | rt           | 实时进程的调用实体，每个进程都有其中之一的实体 |
  | cpus_allowed | 用于控制进程可以在哪里处理器上运行             |

  policy：SCHED_NORMAL，SCHED_BATCH，SCHED_IDLE，SCHED_FIFO，SCHED_RR，SCHED_DEADLINE

### 用户态进程栈

- mm：进程所拥有的用户空间内存描述符，内核线程无的mm为NULL
- active_mm：active_mm指向进程运行时所使用的内存描述符， 对于普通进程而言，这两个指针变量的值相同。但是内核线程kernel thread是没有进程地址空间的，所以内核线程的task_struct->mm域是空（NULL）。但是内核必须知道用户空间包含了什么，因此它的active_mm成员被初始化为前一个运行进程的active_mm值。

### 信号

- signal：指向进程的信号描述符 
- pending：存放私有挂起信号的数据结构

### 文件

- fs：用来表示进程与文件系统的联系，包括当前目录和根目录
- files：表示进程当前打开的文件

### 其他

- usgae：有几个进程正在使用该结构

- flags：反应进程状态的信息，但不是运行状态，用于内核识别状态

- run_list：可运行状态进程队列，这是一个双向链表的结构

- tasks：所有进程队列，是一个双向链表，把所有进程串起来了

  ![task_struct.tasks](https://github.com/SinnerA/blog/blob/master/illustrations/task_struct.tasks.png)

## 参考

[Linux进程描述符task_struct结构体详解--Linux进程的管理与调度（一）](https://blog.csdn.net/gatieme/article/details/51383272)
