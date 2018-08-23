---
title: TCMalloc详解
date: 2018-04-22
tags: 
  - 基础知识
  - 内存管理
---

[TOC]

## 前言

设计一个内存分配器时，需要考虑到很多因素：

- 分配、回收的速度
- 多线程下的行为
- 内存用光时的策略
- 局部缓存
- 额外内存开销
- 小对象和大对象如何分配
- 内部碎片、外部碎片

[TCMalloc](http://goog-perftools.sourceforge.net/doc/tcmalloc.html)是Google 开发的内存分配器，在不少项目中都有使用，例如在 Golang 中就使用了类似的算法进行内存分配。它具有现代化内存分配器的基本特征：分配小对象空间利用率高（减少内存碎片）、减少多线程锁竞争（适合高并发场景）。Google公开的性能测试结果显示，它的内存分配速度是 glibc2.3 中实现的 malloc（ptmalloc2）的数倍。

## 总体架构

如果让你设计一个现代画的内存分配器，你该如何设计？

1. 为了效率肯定要局部缓存
2. 必须考虑到多线程并发的情况，因此为了减少锁的开销，可以考虑线程局部缓存
3. 为了尽量减少内存碎片，需要将内存块分为不同size进行管理
4. 分配策略选择：First-Fit、Next-Fit、Best-Fit
5. 回收策略选择：Last In First Out、Address-Ordered

TCMalloc的策略简单来说就是分层，总体思想是：分配时从里层到外层分别尝试，里层分配失败，从外层补充一批到里层，回收时类似，里层释放一定内存之后，则回收到外层。

内存粒度：

object和span。

多个连续的page(4KB)组成了span，span记录了page的数量和起始编号，也就是说span的大小一定是page的整数倍。span中可以由多个object组成，当然object是地址连续的，而且这些object的总大小也一定是page的整数倍。

object和span都会按照一定规则，分配多种size-class。比如object从小到大有8KB, 16KB, 32KB, 48KB … 256KB，span分为1page, 2page, 3page, 4page … 128page。

分层：

TheadCache：每个线程一份，应对小对象的内存申请，避免了锁竞争，提高了分配速度

CentralCache：维护着多份span list，每个span list维护不同size的span，这些span指向了他们标识的object(s)

PageHeap：管理着span list的list，也就是通过它可以找到某种size的span所在的span list

## 具体策略

### 小对象

#### 定长

这里，为了将对象组成list，需要记录每个对象的位置，很显然需要额外的空间来记录。为了达到内存利用率最大化的目的，如何减少额外空间，成了首要问题。

一般的，可以使用bitmap，实现简单，但是需要花费一定的额外空间。

这里，TCMalloc使用了经典的Freelist。例如，我们要以16字节为单位分配，可以将每页（4KB）划分为多个16字节的单元，其中每个单元前面的8字节作为节点指针，指向下一个单元。指针部分在分配出去之后就是数据区，不占空间，只有待分配的时候才是指针，相当于不需要额外空间。

![image-20180422164552217](https://github.com/SinnerA/blog/blob/master/illustrations/image-20180422164552217.png)

#### 不定长

不定长对象的分配，可以转化为定长对象的分配。

把所有不定长对象取整处理，例如，7字节转化为8字节，13字节转化为16字节，不过带来了内部碎片的问题。为了尽量减少内部碎片，制定了转化规则（下面详述），分别是8, 16, 32, 48, 64, 80 …，并不是简单的2的幂次方。

![image-20180422171310883](https://github.com/SinnerA/blog/blob/master/illustrations/image-20180422171310883.png)

### 大对象

这里，大于256KB的对象被叫做大对象，反之小于256KB的就是小对象。为何这样界定，目前查资料暂时没得到结论。小对象的单位是字节，范围8B ~ 256KB，大对象的单位，按道理至少应该是KB，这里是以页（1page=4KB）为单位。小对象用object来表示，object以字节为单位，这里大对象用span来表示，span以page为单位。

#### span

多个连续的Page会组成一个Span，在 Span 中记录起始Page的编号，以及Page数量。分配对象时，大的对象分配Span，小的对象分配Object。

（注意：span虽然表示多个连续的page，但实际上span由多个object组成，不过这些object是地址连续的，而且总大小为page的整数倍）

![image-20180422183521506](https://github.com/SinnerA/blog/blob/master/illustrations/image-20180422183521506.png)

#### span的分配

span的分配和object的分配类似，不定长的page取整转化为定长page，按照一定规则计算好size-class，不同size的span分配在不同的list中。回收时，需要考虑span的合并策略，前后连续的span要进行合并，避免只剩下很小的span，这样会带来外部碎片的问题。

![image-20180422185021050](https://github.com/SinnerA/blog/blob/master/illustrations/image-20180422185021050.png)

#### PageMap

与object的分配类似，span为了链接组成list，也需要记录每个span的位置信息。

同样简单的，可以使用bitmap，需要花费一定额外空间。

这里，TCMalloc使用了RadixTree，用较少的额外空间以及较快的速度来实现。

>  RadixTree就是压缩过的前缀树（trie），所谓压缩，就是在一条路径上的节点都只有一个子节点时，就把这条路径合并到父节点去，因此内部节点最少会有Radix个字节点。具体的分析可以参考一下 [wikipedia](https://link.zhihu.com/?target=https%3A//en.wikipedia.org/wiki/Radix_tree) 。
>
> 实现时，可以通过一定的空间换来时间，也就是减少层数，比如说3层。每层都是一个数组，用一个地址的前 1/3 的bit 索引数组，剩下的 bit 对下一层进行寻址。实际的寻址也可以非常快。

![image-20180422191708616](https://github.com/SinnerA/blog/blob/master/illustrations/image-20180422191708616.png)

32位系统使用的2层Radix Tree，而64位系统使用的3层Radix Tree。对不同的系统采用不同的数据结构主要是考虑这个映射表占用内存的大小。

下图是32为系统，8K页(kPageShift=13)的pagemap_，8K页总共有2^19页(32-13)，将高5bits作为root[]，而低14位作为Leaf[]。

![image-20180422223105822](https://github.com/SinnerA/blog/blob/master/illustrations/image-20180422223105822.png)

在当前的x86_64处理器中，只用了地址的低48bits用于虚拟和物理地址的转换，高16bits是不用的。所以在8K页的配置下，总共有2^35页(48-13)，TCMalloc将35bits分为12，12，11三级Radix Tree，如下图所示。

![image-20180422223119622](https://github.com/SinnerA/blog/blob/master/illustrations/image-20180422223119622.png)

上面数据结构中的Node和Leaf只有在需要的时候才创建，因此pagemap_实际占用的内存比较少。

#### PageHeap

![image-20180422191938589](https://github.com/SinnerA/blog/blob/master/illustrations/image-20180422191938589.png)

#### CentralCache

现在，整个TCMalloc的结构已经很清晰了，小对象（<=256KB）从ThreadCache中分配Object，大对象（>256KB）从PageHeap中分配Span，其实就是分配Span表示的Object(s)，因此还缺少表示Span到Object的映射的结构，这就是CentralCache。

//TODO：少了一张图，CentralCache的也是跟ThreadCache类似的，有一个size class

![image-20180422192855475](https://github.com/SinnerA/blog/blob/master/illustrations/image-20180422192855475.png)

### 整体回顾

![image-20180422173156149](https://github.com/SinnerA/blog/blob/master/illustrations/image-20180422173156149.png)

上图是TCMalloc的三个层次：ThreadCache、CentralCache、PageHeap。

#### ThreadCache

维护不同size object的FreeList，范围：8B ~ 256KB

#### CentralCache

维护了不同size span的SpanList，范围：1page ~ 128page

回收时，整个span的空间将整体回收到PageHeap。同样的object回收到CentralCache时，也需要找到属于自己的span，这里将会产生一定的耗时。

所以每个SpanList里面还搞了一个cache，回收回来的一批object先往cache里面塞，塞不下了再回收进span的object list。分配object给ThreadCache时也是先尝试在cache里面拿，没了再去span里面分配。

#### PageHeap

维护了两个很重要的东西：page到span的映射关系，和空闲span的伙伴系统。

page到span的映射关系通过radix tree来实现，逻辑上可以把它理解为一个大数组，以page的值作为偏移，就能访问到page所对应的节点。为减少查询radix tree的开销，PageHeap还维护了一个最近最常使用的若干个page到class（span.sizeclass）的对应关系cache。为了保持cache的效率，cache只提供64K个固定坑位，旧的对应关系会被新来的对应关系替换掉。

空闲span的伙伴系统为上层提供span的分配与回收。当需要的span没有空闲时，可以把更大尺寸的span拆小（如果大的span都没有了，则需要重新找kernel分配）；当span回收时，又需要判断相邻的span是否空闲，以便将它们组合。判断相邻span还是要用到radix tree，radix tree就像一个大数组，很容易取到当前span前后相邻的span。

> PageHeap有两个map，pagemap_记录某一内存页对应哪一个span，显然可能多页对应一个span，pagemap_cache_记录某一内存页对应哪一个SizeClass。
>
> 在TCMalloc源码分析（一）中有提到过pagemap_所占内存的问题，假设32位系统4GB可用内存，若pagemap_使用数组实现需要占用4MB的内存（假设一页4KB），仿佛还可以接受，但如果是64位系统呢？所以实际上TCMalloc使用了radix-tree树实现了
>
> pagemap_（64位系统使用三层radix-tree TCMalloc_PageMap2，32位使用两层 TCMalloc_PageMap3）。
>
> radix-tree其实是一棵多叉树，原理是这样：比如三层，会把对应的key的二进制位分成三部分（High，Medium，Low），依次来生成树的三层，最后一层是叶子节点保存key对应的value。
>
> [![graph1(1)](https://images0.cnblogs.com/blog/563450/201311/28230731-bee613f8cd9a4850a946cd0f76d474eb.png)](https://images0.cnblogs.com/blog/563450/201311/28230730-49031cf34b7846da8c53e97abeacc5b3.png)

#### 分配和回收流程

分配流程：

- 根据分配size，判断是小对象还是大对象（256K为界定）
- 小对象：
  1. 通过size计算得到class
  2. 从ThreadCache.list[class]中分配，成功则返回（失败下一步，下同）
  3. 从CentralCache.list[class]的cache中分配batchSize个object，其中一个返回，剩余加入到ThreadCache.list[class]
  4. 从CentralCache.list[class]分配batchSize个object，其中一个返回，剩余加入到ThreadCache.list[class]
  5. 从PageHeap中申请一个span，CentralCache拿到span后，拆分成多个object，其中一个返回，剩余加入到CentralCache.list[class]中
  6. 向kernel申请若干个page的内存，返回所需要的span给PageHeap，其他的span放回到PageHeap的伙伴系统中，之后转5；
- 大对象：
  1. 直接向PageHeap去申请一个刚好大于等于请求size的span。申请过程与小对象走到这一步时的过程一致；

回收流程：

- 通过释放的ptr，得到当前的page，在PageHeap维护的映射关系中，找到page对应span的class（先尝试在cache里面找，没有再查radix tree，然后插入cache。cache里面自动淘汰老的项）。class为0代表ptr指向的是小对象
- 小对象：
  1. 将ptr指向的内存释放到ThreadCache.list[class]里面
  2. 如果ThreadCache.list[class]长度超过阈值（FreeList.length_>=FreeList.max_length），或者ThreadCache的容量超过阈值（ThreadCache.size>=ThreadCache.max_size），则触发回收过程。两种情况分别针对class对应的FreeList，和ThreadCache下面的所有FreeList进行回收
  3. object被回收到CentralCache.list[class]上。先尝试batch_size个object的整块回收，CentralCache.list会试图将其释放到自己的cache里面
  4. 如果cache装满，或者凑不满batch_size个整数的object，则单个回收，回收进其对应的span.objects，这时，object可以通过PageMap直接找到span
  5. 如果span下面的object都已经回收了，则进一步将其释放回PageHeap。在radix tree中找到span之前和这后的span，如果它们空闲且也在normal链上，则进行合并；
- 大对象：
  1. ptr对应的直接就是一个span，直接将其释放回PageHeap即可

### 内存碎片

内存碎片可以分两种，内部碎片和外部碎片，内部碎片是分配器分配的内存大于程序申请的内存，外部碎片是内存块太小，无法分配给程序使用。

![image-20180422230533976](https://github.com/SinnerA/blog/blob/master/illustrations/image-20180422230533976.png)

#### 经典的内存分配算法

内存分配时，通常有 First-Fit、Next-Fit、Best-Fit这几种策略。

回收的时候，也有多种策略，可以直接放回链表头部（Last In First Out），最省事；或者按照地址顺序放回（Address-Ordered），使得链表中的空闲块都按照地址顺序排列。

通常来说，分配时，First-Fit策略会使得链表前面的大块内存被频繁分裂，从而造成较多的内存碎片；Best-Fit的内存碎片较少；放回时，采用Address-Ordered顺序能够增加内存合并的机会，相比于 LIFO 的碎片会更少。

这里有一个很有意思的策略是Address-Ordered。先看一下LIFO的情况：

![image-20180422231145832](https://github.com/SinnerA/blog/blob/master/illustrations/image-20180422231145832.png)

首先这些内存块都是按地址排序的，3和5是空闲的，4是已分配的，3指向5。现在分别申请了3和5的内存，然后又释放了3和5，得到第二幅图的情况，指针变成了5指向3，因为直接把3和5插入到链表头部，LIFO策略。接下来再申请3字节内存，按照 First-Fit策略，就会把5的内存进行分裂。

![image-20180422231231808](https://github.com/SinnerA/blog/blob/master/illustrations/image-20180422231231808.png)

如果采用Address-Ordered策略，放回3和5时，指针顺序还是从3指向5。那么再次申请3字节内存时，直接分配原有的3，而不需要分裂5的内存。

一些研究表明，采用Address-Ordered策略能够显著降低内存碎片，不过其实现较为复杂，释放内存的复杂度较高。

#### Segregated-Freelist

FreeList上面我们详细介绍过，用于分配定长小对象。其实，上面介绍的不定长小对象的分配，采用的就是这里要介绍的Segregated-Freelist。包含多个FreeList，每个 Freelist 存储不同大小的内存块，上面我们也提到，这带来了内部碎片的问题。为了尽量减少内部碎片，需要制定了转化规则，并不是简单的2的幂次方。

![image-20180422232053704](https://github.com/SinnerA/blog/blob/master/illustrations/image-20180422232053704.png)

#### Buddy-System

按照一分为二，二分为四的原则，直到分裂出一个满足大小的内存块；合并的时候看看它的 Buddy 是否也为空闲，如果是就可以合并，可以一直向上合并。

伙伴系统的优势在于地址计算很快，对于一个内存块的地址，可以通过位运算直接算出它的 Buddy，Buddy 的 Buddy，因此速度很不错。

不过考虑内存碎片，它并没有什么优势，像图中的这种经典的 Binary Buddy，全部都是2的幂级数，内部碎片还是会达到 50%。当然也有一些其他的优化，块大小可以是3的倍数之类的。

![image-20180422232126254](https://github.com/SinnerA/blog/blob/master/illustrations/image-20180422232126254.png)

#### 内存分配算法的比较

对于以上的几种算法，实际会产生的内存碎片会有多少呢，有人专门做过测试比较：

![image-20180422232255300](https://github.com/SinnerA/blog/blob/master/illustrations/image-20180422232255300.png)

Frag#3和Frag#4分别是两种不同的度量方法，一个是分配器申请内存最大时，程序分配的内存和分配器申请的内存，另一个是程序最大申请的内存和分配器最大申请的内存。测试时使用了实际的程序进行模拟，例如GCC之类内存开销较大的程序。

对于 best fit AO来说，其内存碎片显然是相当少的，而一些在我们看来比较朴素的算法，first fit，其内存碎片率其实也相当低，反而是 buddy system 和 segregated list 比较尴尬。

因此，说明只要选择了合适的策略，其内存碎片就完全不用担心，只要关心实现的性能就可以了。

#### TCMalloc的内部碎片

TCMalloc采用了 Segregated-Freelist 的算法，提前分配好多种size-class，在64位系统中，通常就是 88 种，那么问题来了，这个怎么计算？

首先看一下结果：8， 16， 32， 48， 64， 80， 96， 112， 128， 144， 160， 176......

TCMalloc 的目标是最多 12.5% 的内存碎片，按照上面的 size-class算一下，例如 [112, 128)，碎片率最大是 (128-112-1)/128 = 11.7%，(1152-1024-1)/1151 = 11.02%。当然结果是满足 12.5% 这一目标的。

直接看下align的代码，这是省略版本，看起来很简单

```go
func align(size int) int {
	aligment := 8
	if size > 256*1024 {
		aligment = PageSize
	} else if size >= 128 {
		aligment = (1 << uint32(lgfloor(size))) / 8
	} else if size >= 16 {
		aligment = 16
	}
	if aligment > PageSize {
		aligment = PageSize
	}
	return aligment
}
```

计算 Alignment 的时候，大于 256 × 1024就按照Page进行对齐；最小对齐是8；在128到256×1024之间的，按照 1<<lgfloor(size) / 8进行对齐。

其实 lgfloor 就是用二分查找，向下对齐到2的幂级数的：

```go
func lgfloor(size int) int {
	n := 0
	for i := 4; i >= 0; i-- {
		shift := uint32(1 << uint32(i))
		if (size >> shift) != 0 {
			size >>= shift
			n += int(shift)
		}
	}
	return n
}
```

先看左边16位，有数字的话就搜左边，否则搜右边

#### TCMalloc的外部碎片

外部碎片是因为 CentralCache 在向 PageHeap 申请内存的时候，以 Page 为单位进行申请。举个例子，对于 size-class 1024，以一个Page（8192）申请，完全没有外部碎片；但是对于 size-class 1152，就有 8192 % 1152 = 128 的碎片。为了保证外部碎片也小于 12.5%，可以一次多申请几个Page，但是又不能太多造成浪费。

```go
func numMoveSize(size int) int {
	if size == 0 {
		return 0
	}
	num := 64 * 1024 / size
	return minInt(32768, maxInt(2, num))
}

func InitPages(classToSize []int) []int {
	classToPage := make([]int, 0, NumClass)
	for _, size := range classToSize {
		blocksToMove := numMoveSize(size) / 4
		psize := 0
		for {
			psize += PageSize
			for (psize % size) > (psize >> 3) {
				psize += PageSize
			}
			if (psize / size) >= blocksToMove {
				break
			}
		}
		classToPage = append(classToPage, psize>>PageShift)
	}
	return classToPage
}
```

这里计算出来的结果，就是每个 size-class 每次申请的 Page 数量，保证12.5%以下的外部碎片。

## 总结

1. 分为三层，ThreadCache、CentralCache、PageHeap，申请时从里层逐步向外层申请
2. ThreadCache很简单，维护了不同size-class对应的freeList
3. CentralCache维护span的链表，每个span下面再挂一些由这个span切分出来的object的链表。这样做便于在span内的object是否都已经free的情况下，将span整体回收给PageHeap。每个回收的object都需要找到自己的span，比较耗时，因此弄了个cache，先放到cache里面，拿也是先从cache里面拿
4. PageHeap维护了两个很重要的东西：page到span的映射关系，和空闲span的伙伴系统。地址值经过地址对齐，很容易知道它属于哪一个page。再通过page到span的映射关系就能知道object应该回收到哪里。当需要的span没有空闲时，伙伴系统可以把更大尺寸的span拆小。
5. 特色：
   - 小对象从ThreadCache分配，减少锁竞争
   - 碎片问题：采用Segregated-Freelis（离散式空闲列表）的算法，按照测试的结果，提前分配好多种size-class，碎片率控制在12.5%以内
   - 适合于线程数不固定，经常频繁创建退出的场景，因为有ThreadCache
   - 周期性的垃圾回收则将内存从各个ThreadCache回收到CentralCache

## 参考

[TCMalloc : Thread-Caching Malloc](http://goog-perftools.sourceforge.net/doc/tcmalloc.html)

[图解 TCMalloc](https://zhuanlan.zhihu.com/p/29216091)

[TCMalloc分析 - 如何减少内存碎片](https://zhuanlan.zhihu.com/p/29415507)

[tcmalloc浅析](https://yq.aliyun.com/articles/6045)

[TCMalloc分析笔记(gperftools-2.4)](https://blog.csdn.net/zwleagle/article/details/45113303)
