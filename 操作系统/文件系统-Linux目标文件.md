---
title: 文件系统-Linux目标文件
date: 2018-01-14 00:05:13
tags: OS Linux
---

目标文件是指:编译器编译源代码后生成的文件，那么目标文件里面到底存放的是什么呢？或者说我们的源代码在经过编译以后是怎么样存储的呢？

目标文件从结构上将，它是已经编译后的可执行文件格式，只是好没有经过链接的过程，其中可能有些符号或有些地址还没有被调整。其实，目标文件本身就是按照可执行文件格式存储的，只是跟真正的可执行文件在结构上稍有不同。

可执行文件格式涵盖了程序的编译、链接、装载和执行的各个方面。了解可执行文件的格式对认识系统，了解集成编译器背后的运行机理，还是很有好处的。

### 目标文件到底是什么样的？

猜也可以猜到，目标文件中的内容**至少有编译后的机器指令代码、数据**。没错，除了这些内容以外，目标文件中**还包含了链接时所需要的一些信息，比如符号表、调试信息、字符串等**。一般目标文件将这些信息按不同的属性，以“节”（section）的形式存储，有时候也叫“段”（segment），在一般情况下，他们都表示一个一定长度的区域，基本上不加以区别，唯一的区别是在ELF的链接视图和装载视图的时候，需要特别注意。

**程序源代码编译后的机器指令经常放在代码段（Code Section）里**，代码段常见的字有“.code”和“.text”；**全局变量和局部变量数据经常放在数据段（Data Section）**，数据段的一段名字都叫“.data”。下面是一个简单的程序被编译成目标后的结构，如下图所示：

![img](http://img.blog.csdn.net/20160925105624431)

假设上图的可执行文件格式是ELF，从图中可以看到，ELF文件的开头是一个**“文件头”，他描述了整个文件的文件属性，包括文件是否可执行、是静态链接还是动态链接以及入口地址（如果是可执行文件）、目标硬件、目标操作系统等信息**。头文件包含一个段表（Section Table），段表事实是一个描述文件中各个段的数组。段表描述了文件中各个段在文件中的偏移位置及段的属性，从段表里面可以得到每个段的所有信息。文件头后面就是各个段的内容，比如代码段保存的就是程序的指令，数据段里面保存的就是程序的静态变量等。

**注**：

从上图我们也能看出，

一般C语言的编译后执行语句都编译成机器代码，保存在.text段；

**已初始化**的全局变量和局部静态变量都保存在.data段；

**未初始化**的全局变量和局部静态变量一般放在一个叫*.bss*的段里。

我们知道未初始化的全局变量和局部静态变量默认值都为零，本来他们也可以被放在.data段里的，但是因为他们都是0，所以为他们在.data段里分配空间并且存放数据0是没有必要的。程序运行的时候他们的确是要占用内存空间的，并且可执行文件必须记录所有未初始化的全局变量和局部静态变量的大小总和，记为“.bss”段。所以，**.bss段只是为未初始化的全局变量和局部静态变量预留位置而已。他并没有内容，所以他在文件中也不占据空间**。

### 程序的指令和数据为什么要分开存放？

总体来说，程序源代码被编译以后主要分成两种段：程序指令和程序数据。代码段属于程序指令，而数据段和.bss段属于程序数据。

这样，我们可能就会产生疑问：为什么要这样麻烦，把程序的指令和数据的存储分开？混杂的放在一个段里面岂不是会更简单？其实数据和指令分段的好处有很多，主要如下：

**#1**：当程序被装载后，数据和指令分别被映射到两个虚存区域。由于数据区域对于进程来说是可读写的，而指令区域对于进程来说是只读的，所以这两个虚存区域的权限可以被分别设置成可读写或只读。这样可以防止程序的指令被有意或无意地改写。

**#2**：对现在的CPU来说，他们有着极为强大的缓存(Cache)体系。由于缓存在现代的计算机中地位非常重要，所以程序必须尽量提高缓存的命中率。指令区和数据区的分离有利于提高程序的局限性。现在CPU的缓存一般都被设计成数据缓存和指令缓存分离，所以程序的指令和数据被分开存放对CPU的缓存命中率提高有好处。

**#3**：这是最重要的原因！就是当系统中运行着多个改程序的副本时，他们的指令都是一样的，所以内存中只需保存一份该程序的指令部分。对于指令这种制度的区域来说是这样的，对于其他的只读数据也是这样的。比如，很多程序里面带有的图标、图片、文本等资源也是属于可以共享的。当然每个副本进程的数据区域是不一样的，他们是进程私有的。在动态链接的系统中，该种方式可以节省大量内存。比如我们常用的Windows Internet Explorer 7.0 运行起来以后，他的总虚存空间为112844KB，他的私有数据是15944KB，既有96900KB的空间是共享部分！！！

### 为什么要分堆和栈



### 引用

http://blog.csdn.net/shenziheng1/article/details/52658431


