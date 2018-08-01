---
title: Linux内存管理 - 2. 虚拟内存管理
date: 2018-05-26
tags: Linux 内存管理
---

[TOC]

## 概览

Linux虚拟内存现在采用的是段页式分配机制，采用的是4级页表。物理寻址的好处是简单，坏处也有很多，比如：

### 为什么需要虚拟内存

CPU 对内存的寻址最简单的方式就是直接使用物理内存地址，这种方式一般叫做物理寻址。

1. **不安全：**操作系统的地址直接暴露给用户程序，用户程序可以破坏操作系统。
2. **同时运行多个程序比较困难：**多个用户程序如果都直接引用物理地址，很容易互相干扰。可以通过不断交换物理内存和磁盘来保证物理内存某一时间自由一个程序在运行，但是这引入很多不必要和复杂的工作。 
3. **用户程序大小受限**：受制于物理内存大小。

### 思想

大小一般是和 CPU 字长相关，比如 32 位对应的虚拟地址空间大小为：0 ~ 2^31。

每个程序拥有独立的地址空间（也就是虚拟内存地址，或者称作虚拟地址），互不干扰。地址空间被分割成多个块，每一块称作一页（page），每一页有连续的地址范围。

虚拟地址的页被映射到物理内存（通过 MMU，Memory Management Unit），但是并不是所有的页都必须在内存中才能运行程序。当程序引用到一部分在物理内存中的地址空间时，由硬件立刻执行必要的映射。当程序引用到一部分不在物理内存中的地址空间时，由操作系统负责将确实的部分装入物理内存。

虚拟地址寻址（也叫做虚拟寻址）的示意图如下。

![img](/Users/sinnera/sinnera.github.io/source/illustrations/vm_address_02.png)

### MMU

CPU 将虚拟地址发送给 MMU，然后 MMU 将虚拟地址翻译成物理地址，再寻址物理内存。那么虚拟地址和物理地址具体是怎么映射的呢？

完成映射还需要另一个重要的数据结构的参与：**页表**（page table）。页表完成虚拟地址和物理地址的映射，MMU每次翻译的时候都需要拿着页号读取页表，找到具体页地址，加上偏移量就得到了物理地址。

页表的一种简单表示如下。

![img](/Users/sinnera/sinnera.github.io/source/illustrations/page_table.png)

### 页表

页表的大小为虚拟地址位数/页的大小。比如32位机器，页大小一般为4K ，则页表项有 2^32 / 2^12 = 2^20 条目。如果机器字长64位，页表项就更多了。页表项就占用了大量内存。

可以采用多级页表来解决。使用二级页表来索引物理地址，一级页表用来索引二级页表，这样一级页表的页表项大大减少了。比如，32位系统中，需要的二级页表总条目还是2^32 / 2^12 = 2^20个，但是一级页表只需要2^20 / 2^12 = 512个页表项。

多级页表减少内存占用的关键在于：

1. 如果一级页表中的一个页表项为空，那么相应的二级页表就根本不会存在。这是一种巨大的潜在节约。
2. **只有一级页表才需要常驻内存**。虚拟内存系统可以在需要时创建、页面调入或者调出二级页表，从而减轻内存的压力。

> Linux在没有启用物理地址扩展的32位系统，两级页表够用了，启用了物理地址扩展的32 位系统使用了三级页表，64位系统采用的是三级或者四级分页，这取决于硬件对线性地址的位的划分

### TLB

页表是在内存中，而 MMU 位于 CPU 芯片中，这样每次地址翻译可能都需要先访问一次内存中的页表，效率非常低下。对应的解决方案是引入页表的高速缓存：TLB（Translation Lookaside Buffer）。

加入 TLB，整个虚拟地址翻译的过程如下两图所示。

![img](/Users/sinnera/sinnera.github.io/source/illustrations/tlb_hit.png)

### Page Fault（缺页错误）

当一个逻辑地址，经过MMU映射后发现，对应的页表项还没有映射到物理内存，就会触发缺页错误(page fault)：CPU需要陷入kernel，找到一个可用的物理内存页面，从页表项映射过去。

page fault分为：**硬缺页错误**（Hard Page Fault）和**软缺页错误**（Soft Page Fault），**无效页错误**（Invalid）

#### Soft Page Fault

Soft Page Fault也称为minor page fault，指需要访问的内存不在虚拟地址空间，但是在物理内存中，只需要MMU建立物理内存和虚拟地址空间的映射关系即可。 

发生这种情况的可能性之一，是一块物理内存被两个或多个程序共享，操作系统已经为其中的一个装载并注册了相应的页，但是没有为另一个程序注册。

可能性之二，是该页已被从CPU的工作集中移除，但是尚未被交换到磁盘上

#### Hard Page Fault

Hard Page Fault也称为major page fault，指需要访问的内存不在虚拟地址空间，也不在物理内存中，需要从慢速设备载入。从swap回到物理内存也是hard page fault。

这时操作系统需要：

1. 寻找到一个空闲的页。或者把另外一个使用中的页写到磁盘上（如果其在最后一次写入后发生了变化的话），并注销在MMU内的记录
2. 将数据读入被选定的页
3. 向MMU注册该页，建立映射关系

#### Invalid Page Fault

当程序访问的虚拟地址是不存在于虚拟地址空间内的时候，属于越界访问，则发生**无效页缺失**。一般来说这是个软件问题，但是也不排除硬件可能，比如因为内存故障而损坏了一个正确的指针。

### 分页/分段/段页式

分页优缺点：

- 优点：
  1. 页长固定，容易管理，且不存在外部碎片
- 缺点：
  1. 不利于共享，进程直接映射到页面，页面中可能包含不能共享的内容
  2. 不利于动态增长，其中某一部分的空间扩张起来都会影响到相邻的空间

分段优缺点：

- 优点：
  1. 按逻辑关系划分，因此共享起来十分方便
  2. 段长可以动态变化，其他段不受影响
- 缺点：
  1. 段之间容易留下碎片
  2. 相比分页需要更多的硬件支持

分段的思想, 说穿了就是把内存分成若干段, 每个段是一个单独的地址空间, 有自己的起始的基地址, 根据 基地址+偏移量 来做寻址.

分段的好处是带来了比较大的灵活性, 也更安全. 每个段都构成了自己的独立地址空间, 增大或者减小而不会影响其他段. 还可以对每个段设置不同的保护级别.

Linux下采用的是段页式内存管理，先分段，再分页。但是因为Linux中所有的段基址都设置成了0，段偏移量相当于就是线性地址，只用了一个地址空间，效果上就是正常的分页。其实Linux的进程处理机制很大程度依赖于分页，使用分页会更加高效，但是为了兼容各种硬件体系必须要支持分段，所以分段在Linux中只为了兼容。

虽然Linux下, "分段"只是一个摆设, 但是在进程的内存管理中, 还是应用了分段的思想的: 每一个进程在运行时, 它的逻辑地址空间都会被分为代码段, 数据段, 堆, 栈等, 当访问段之外的内存地址时, kernel能监测到并给出段错误(segment fault)

采用分段和分页结合的方式管理内存，一个地址由两个部分组成：段和段内地址。段内地址又进一步分为页号和页偏移。在进行内存访问时，过程如下：

1. 根据段号找到段描述符（存放段基址）。
2. 检查该段的页表是否在内存中。如果在，则找到它的位置，如果不在，则产生段错误。
3. 检查所请求的虚拟页面的页表项，如果该页面不在内存中则产生缺页中断，如果在内存中就从页表项中取出这个页面在内存中的起始地址。
4. 将页面起始地址和偏移量进行拼接得到物理地址，然后完成读写。

## mm_struct

用户进程地址空间逻辑上分为：

1. 代码段：已编译的机器代码，运行过程中不能被修改
2. 数据段：保存全局变量，静态变量和一些常量字符串等
3. 堆 ：就是平时所说的动态内存， malloc/new 大部分都来源于此。其中堆顶的位置可通过函数brk和sbrk进行动态调整。
4. 栈：用于维护函数调用的上下文空间，一般为 8M ，可通过 ulimit –s 查看。
5. BSS：未初始化的全局和静态变量
6. 文件映射区域 ：如动态库、共享内存等映射物理空间的内存，一般是mmap函数所分配的虚拟地址空间。
7. 内核虚拟空间（不属于用户态）：用户代码不可见的内存区域，由内核管理。（陷入内核态时使用）

> 注：%esp 执行栈顶，往低地址方向变化；brk/sbrk 函数控制堆顶往高地址方向变化

下图是 32 位系统典型的虚拟地址空间分布，这里逻辑上是连续的，其实物理上并不连续：

![img](/Users/sinnera/sinnera.github.io/source/illustrations/linux_memory_04.png)

### 结构

task_struct中有个字段是mm_struct，抽象并描述了Linux视角下管理进程地址空间的所有信息，mm_struct定义在include/linux/mm_types.h中：

```c
struct mm_struct
{
    unsigned long mmap_base;                /* base of mmap area */
    pgd_t * pgd;                            //指向页全局目录
    
    struct vm_area_struct *mmap;            /* list of VMAs */
    struct rb_root mm_rb;
    struct vm_area_struct *mmap_cache;      /* last find_vma result */
    ...
    unsigned long start_code, end_code, start_data, end_data;
    unsigned long start_brk, brk, start_stack;
    ...
    atomic_t mm_users;                      //次使用计数器，使用这块空间的个数    
    atomic_t mm_count;                      //主使用计数器
    ...
};
```

其中的域抽象了进程的地址空间，如下图所示：

![img](/Users/sinnera/sinnera.github.io/source/illustrations/linux_memory_11.png)

每个进程都有自己独立的mm_struct，使得每个进程都有一个抽象的平坦的独立的32或64位地址空间，各个进程都在各自的地址空间中相同的地址内存存放不同的数据而且互不干扰。

代码段：[start_code, end_code)

数据段：[start_data, end_start)

堆：[start_brk, brk)分别表示heap段的起始空间和当前的heap指针

栈：[start_stack, end_stack)

BSS：只有一个占位符，并不占用空间

映射区：mmap_base表示memory mapping段的起始地址，结束地址？

除此之外，mm_struct还定义了几个重要的域：

```c
atomic_t mm_users;                      /* How many users with user space? */
atomic_t mm_count;                      /* How many references to "struct mm_struct" (users count as 1) */
```

### vm_area_struct

![task_struct，mm_struct，vm_area_struct](/Users/sinnera/sinnera.github.io/source/illustrations/vma.png)

tsk->mmap指向了由多个vm_area_struct组成的集合。vm_area_struct是虚存管理的最基本的管理单元，分别表示不同类型的虚拟内存区域。vm_area_struct表示一段连续的虚拟空间，大小为页大小的倍数。

VMA（vm_area_struct）的种类：

- 和文件相关的VMA：代码/库, 数据文件, 共享内存, 设备; 这些 VMA 的内容都是来至于文件的.
- 匿名VMA:：Stack, Heap, CoW pages; 这些 VMA 的内容都是用户程序管理的.

#### 结构

vm_area_struct 的关键 fields:

- vm_mm: 指向 VMA 对应的 mm_struct;
- vm_start: vma 的起始位置 (低位)
- vm_end: vma 的结束位置 (高位). vm_end - vm_start 就是这个 VMA 的 size.
- 一组函数指针; 这些函数实现了在这个VMA 上的各种操作 (page fault, open, close ...)

#### 存储

vm_area_struct集合存储在mm_struct中的一个单向链表和红黑树中。当输出/proc/pid/maps文件时，只需要遍历这个链表即可。红黑树主要是为了通过给定的虚拟地址能够快速定位到某一个内存块，红黑树的根存储在mm_rb域。

![task_struct，mm_struct，vm_area_struct](/Users/sinnera/sinnera.github.io/source/illustrations/vma _02.png)

#### 总结

1. linux进程的内存布局的每个段都是有一个vm_area_struct，而这个实例是由连续的虚拟内存地址组成；
2. 当请求内存时，先是扩展vm_area_struct或者新分配一个vm_area_struct，但是并不映射物理内存，只有等到访问这块内存时，产生缺页异常，内核才分配物理内存。

### mm_users

无论我们在调用fork,vfork,clone的时候最终会调用do_fork函数，里面又会调用copy_mm函数，copy_mm函数中，如果创建线程中有CLONE_VM标识，则表示父子进程共享地址空间和同一个内存描述符，并且只需要将mm_users值+1，也就是说mm_users表示正在引用该地址空间的thread数目，是一个**thread level的counter**。

### mm_count

对Linux来说，用户进程和内核线程（kernel thread)都是task_struct的实例，唯一的区别是kernel thread是没有进程地址空间的，内核线程也没有mm描述符的，所以内核线程的tsk->mm域是空（NULL）。内核scheduler在进程context switching的时候，会根据tsk->mm判断即将调度的进程是用户进程还是内核线程。但是虽然thread thread不用访问用户进程地址空间，但是仍然需要page table来访问kernel自己的空间。

但是幸运的是，**对于任何用户进程来说，他们的内核空间都是100%相同的**，所以内核可以’borrow'**上一个被调用的用户进程**的mm中的页表来访问内核地址，这个mm就记录在**active_mm**。

简而言之就是，对于kernel thread,tsk->mm == NULL表示自己内核线程的身份，而tsk->active_mm是借用上一个用户进程的mm，用它的page table来访问内核空间。对于用户进程，tsk->mm == tsk->active_mm。

为了支持这个特别，mm_struct里面引入了另外一个counter，mm_count。刚才说过mm_users表示这个进程地址空间被多少线程共享或者引用，而mm_count则表示这个地址空间被内核线程引用的次数+1。

内核不会因为mm_users==0而销毁这个mm_struct，内核**只会当mm_count==0的时候才会释放mm_struct**，因为这个时候既没有用户进程使用这个地址空间，也没有内核线程引用这个地址空间。

## mmap

mmap是一种**内存映射文件**的方法，即将一个**文件或者其它对象映射到进程的地址空间**，实现文件磁盘地址和进程虚拟地址空间中一段虚拟地址的一一对映关系。

实现这样的映射关系后，进程就可以采用指针的方式读写操作这一段内存，而系统会自动回写脏页面到对应的文件磁盘上，即完成了对文件的操作而不必再调用read，write等系统调用函数。

相应的，内核空间对这段区域的修改也直接反映用户空间，从而可以实现不同进程间的文件**共享**。

> mmap()并不仅仅是说把硬盘空间直接映射为一段内存，而是把某个文件的连续一段映射为一段连续内存。“文件”这个概念可以是设备，可以是某个驱动假造出来的文件，也可以是磁盘文件。

![img](/Users/sinnera/sinnera.github.io/source/illustrations/mmap.png)

tsk->mmap区域是一堆vma的集合，mmap函数就是要创建一个新的vm_area_struct结构，并将其与文件的物理磁盘地址相连。

### 过程

1. 进程在用户空间调用库函数mmap，寻找一段空闲的满足要求的连续的**虚拟地址**，为此虚拟区分配一个vm_area_struct结构

2. 调用内核函数mmap，它将通过虚拟文件系统inode模块定位到文件磁盘物理地址，建立起**页表映射关系**

   > 此时，并没有将任何文件数据的拷贝至主存，真正的文件读取是当进程发起读或写操作时

3. 进程的读或写操作访问虚拟地址空间这一段映射地址，通过查询页表，发现这一段地址并不在物理页面上，真正的硬盘数据还没有拷贝到内存中，因此引发**缺页异常**。

4. 调页过程先在交换缓存空间（swap cache）中寻找需要访问的内存页，如果没有则把所缺的页从**磁盘装入**到主存中

5. 之后进程即可对这片主存进行读或者写的操作，如果写操作改变了其内容，一定时间后系统会自动（pdflush）**回写脏页面**到对应磁盘地址，也即完成了写入到文件的过程。

### 优缺点

1. 对文件的读写操作，减少了数据的拷贝次数，用内存读写取代I/O读写，提高了文件读取效率
2. 实现了用户空间和内核空间的高效交互方式。两空间的各自修改操作可以直接反映在映射的区域内
3. 提供进程间共享内存及相互通信的方式。不管是父子进程还是无亲缘关系的进程，都可以将自身用户空间映射到同一个文件或匿名映射到同一片区域

### 总结

1. **mmap之所以快，是因为建立了页到用户进程的虚地址空间映射**，以读取文件为例，避免了页从内核态拷贝到用户态（零拷贝）。
2. mmap映射的页和其它的页并没有本质的不同。得益于主要的3种数据结构的高效，其页映射过程也很高效：
   - radix tree，用于查找某页是否已在缓存
   - red black tree ，用于查找和更新vma结构
   - 双向链表，用于维护active和inactive链表，支持LRU类算法进行内存回收
3. mmap不是银弹
   - 对变长文件不适合
   - 如果更新文件的操作很多，mmap避免两态拷贝的优势就被摊还，最终还是落在了大量的脏页回写及由此引发的随机IO上（大量swap）
4. **在随机写很多的情况下，mmap方式在效率上不一定会比带缓冲区的一般写快**

## malloc

进程调用malloc分配内存，陷入内核态分别由brk和mmap完成，但这两种分配还没有分配真正的物理内存。

**brk：堆**上分配内存，数据段的最高地址指针_edata往高地址推

- 当malloc需要分配的内存<M_MMAP_THRESHOLD（默认**128k**）时，采用brk
- brk分配的内存需高地址内存全部释放之后才会释放。(由于是通过推动指针方式)
- 当最高地址空间的空闲内存大于M_TRIM_THRESHOLD时(默认128k)，执行内存紧缩操作；

**do_mmap：在文件映射区域**找空闲的虚拟内存

- 当malloc需要分配的内存>M_MMAP_THRESHOLD（默认128k）时，采用do_map();
- mmap分配的内存可以单独释放

## 内存分配场景

- page 管理
- slab（kmalloc、内存池）
- 用户态内存使用（malloc、relloc 文件映射、共享内存）
- 程序的内存 map（栈、堆、code、data）
- 内核和用户态的数据传递（copy_from_user、copy_to_user）
- 内存映射（硬件寄存器、保留内存）
- DMA 内存

## 总体架构图

![img](/Users/sinnera/sinnera.github.io/source/illustrations/memory_manager.jpg)

## 参考

[malloc 背后的系统知识](http://legendtkl.com/2017/03/21/malloc-os-knowledge/)

[Linux进程地址管理之mm_struct](https://www.cnblogs.com/Rofael/archive/2013/04/13/3019153.html)

[linux内存管理](http://luodw.cc/2016/02/17/linux-memory/)