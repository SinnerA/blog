---
title: golang函数调用协议
date: 2018-05-06
tags: 
  - golang
---

[TOC]

## 多值返回

让我们先看一看C语言是如果返回多个值的。在C中如果想返回多个值，通常会在调用函数中分配返回值的空间，并将返回值的指针传给被调函数。

```
int ret1, ret2;
f(a, b, &ret1, &ret2)
```

被调函数被定义为下面形式，在函数中会修改ret1和ret2。对指针参数所指向的内容的修改会被返回到调用函数，用这种方式实现多值返回。

假设我们定义一个Go函数如下：

```
func f(arg1, arg2 int) (ret1, ret2 int)
```

Go的做法是在传入的参数之上留了两个空位，被调者直接将返回值放在这两空位，函数f调用前其内存布局是这样的：

```
为ret2保留空位
为ret1保留空位
参数3
参数2
参数1  <-SP 
```

调用之后变为：

```
为ret2保留空位
为ret1保留空位
参数3
参数2
参数1  <-FP
保存PC <-SP
f的栈
...
```

![img](https://github.com/SinnerA/blog/blob/master/illustrations/3.2.funcall.png)

这就是Go和C函数调用协议中很重要的一个区别：为了实现多值返回，Go是使用**栈空间**来返回值的。而常见的C语言是通过**寄存器**来返回值的。

## go关键字

先看看正常的函数调用，下面是调用f(1, 2, 3)时的汇编代码：

```assembly
    MOVL    $1, 0(SP)
    MOVL    $2, 4(SP)
    MOVL    $3, 8(SP)
    CALL    f(SB)
```

首先将参数1、2、3进栈，然后调用函数f。

下面是go f(1, 2, 3)生成的代码：

```assembly
    MOVL    $1, 0(SP)
    MOVL    $2, 4(SP)
    MOVL    $3, 8(SP)
    PUSHQ   $f(SB)
    PUSHQ   $12
    CALL    runtime.newproc(SB)
    POPQ    AX
    POPQ    AX
```

对比一个会发现，前面部分跟普通函数调用是一样的，将参数存储在正常的位置。接下来的两条指令有些不同，将f和12作为参数进栈而不直接调用f，然后调用函数`runtime.newproc`。

12是参数占用的大小。`runtime.newproc`函数接受的参数分别是：参数大小，新的goroutine是要运行的函数，函数的n个参数。

在`runtime.newproc`中，会新建一个栈空间，将栈参数的12个字节拷贝到新栈空间中并让栈指针指向参数。这时的线程状态有点像当被调度器剥夺CPU后一样，寄存器PC、SP会被保存到类似于进程控制块的一个结构体struct G内。f被存放在了struct G的entry域，后面进行调度器恢复goroutine的运行，新线程将从f开始执行。

在函数协议上，go表达式调用就比普通的函数调用多四条指令而已，并且在实际上并没有为go关键字设计一套特殊的东西。

总结一个，go关键字的实现仅仅是一个语法糖衣而已，也就是：

```go
    go f(args)
```

可以看作

```go
    runtime.newproc(size, f, args)
```

## defer关键字

### 坑

defer是在return之前执行的。这个在 [官方文档](http://golang.org/ref/spec#defer_statements)中是明确说明了的。要使用defer时不踩坑，最重要的一点就是要明白，**return xxx这一条语句并不是一条原子指令!**

函数返回的过程是这样的：先给返回值赋值，然后调用defer表达式，最后才是返回到调用函数中。

defer表达式可能会在设置函数返回值之后，在返回到调用函数之前，修改返回值，使最终的函数返回值与你想象的不一致。

其实使用defer时，用一个简单的转换规则改写一下，就不会迷糊了。改写规则是将return语句拆成两句写，return xxx会被改写成:

```
返回值 = xxx
调用defer函数
空的return
```

看个例子

```go
func f() (result int) {
    defer func() {
        result++
    }()
    return 0
}
```

实际上可以改写成

```go
func f() (result int) {
     result = 0  //return语句不是一条原子调用，return xxx其实是赋值＋ret指令
     func() { //defer被插入到return之前执行，也就是赋返回值和ret指令之间
         result++
     }()
     return //return 1，而不是0
}
```

defer确实是在return之前调用的。但表现形式上却可能不像。本质原因是return xxx语句并不是一条原子指令，defer被插入到了赋值与ret之间，因此可能有机会改变最终的返回值。

#### 命名返回值

如果是命名返回值，defer才有可能直接操作返回值。如果非命名返回值，则defer内操作的是局部变量，不会影响返回值。

### 实现 

defer关键字的实现跟go关键字很类似，不同的是它调用的是runtime.deferproc而不是runtime.newproc。

在defer出现的地方，插入了指令call runtime.deferproc，然后在函数返回之前的地方，插入指令call runtime.deferreturn。

普通的函数返回时，汇编代码类似：

```
add xx SP
return
```

如果其中包含了defer语句，则汇编代码是：

```
call runtime.deferreturn，
add xx SP
return
```

goroutine的控制结构中，有一张表记录defer，调用runtime.deferproc时会将需要defer的表达式记录在表中，而在调用runtime.deferreturn的时候，则会依次从defer表中出栈并执行。

## 闭包

闭包是由函数及其相关引用环境组合而成的实体(即：闭包=函数+引用环境)。

这里简单地讲一下在Go语言中闭包是如何实现的。

```go
func f(i int) func() int {
    return func() int {
        i++
        return i
    }
}
```

函数f返回了一个函数，返回的这个函数，返回的这个函数就是一个闭包。这个函数中本身是没有定义变量i的，而是引用了它所在的环境（函数f）中的变量i。

```go
c1 := f(0)
c2 := f(0)
c1()    // reference to i, i = 0, return 1
c2()    // reference to another i, i = 0, return 1
```

c1跟c2引用的是不同的环境，在调用i++时修改的不是同一个i，因此两次的输出都是1。函数f每进入一次，就形成了一个新的环境，对应的闭包中，函数都是同一个函数，环境却是引用不同的环境。

变量i是函数f中的局部变量，假设这个变量是在函数f的栈中分配的，是不可以的。因为函数f返回以后，对应的栈就失效了，f返回的那个函数中变量i就引用一个失效的位置了。所以**闭包的环境中引用的变量不能够在栈上分配**

### 逃逸分析

先看一看Go的一个语言特性：

```go
func f() *Cursor {
    var c Cursor
    c.X = 500
    noinline()
    return &c
}
```

Cursor是一个结构体，这种写法在C语言中是不允许的，因为变量c是在栈上分配的，当函数f返回后c的空间就失效了。但是，在Go语言规范中有说明，这种写法在Go语言中合法的。语言会自动地识别出这种情况并在堆上分配c的内存，而不是函数f的栈上。

为了验证这一点，可以观察函数f生成的汇编代码：

```assembly
MOVQ    $type."".Cursor+0(SB),(SP)    // 取变量c的类型，也就是Cursor
PCDATA    $0,$16
PCDATA    $1,$0
CALL    ,runtime.new(SB)    // 调用new函数，相当于new(Cursor)
PCDATA    $0,$-1
MOVQ    8(SP),AX    // 取c.X的地址放到AX寄存器
MOVQ    $500,(AX)    // 将AX存放的内存地址的值赋为500
MOVQ    AX,"".~r0+24(FP)
ADDQ    $16,SP
```

识别出变量需要在堆上分配，是由编译器的一种叫escape analyze的技术实现的。如果输入命令：

```shell
go build --gcflags=-m main.go
```

可以看到输出：

```shell
./main.go:20: moved to heap: c
./main.go:23: &c escapes to heap
```

表示c逃逸了，被移到堆中。escape analyze可以分析出变量的作用范围，这是对垃圾回收很重要的一项技术。

### go闭包的实现

闭包是函数和它所引用的环境组成的，那么是不是可以表示为一个结构体呢。事实上，Go在底层就是通过结构体表示闭包的，大概长这样：

```go
type Closure struct {
    F func()() 
    i *int
}
```

### 小结

1. Go语言支持闭包
2. Go语言能通过escape analyze识别出变量的作用域，自动将变量在堆上分配。将闭包环境变量在堆上分配是Go实现闭包的基础。
3. 返回闭包时并不是单纯返回一个函数，而是返回了一个结构体，记录下函数返回地址和引用的环境中的变量地址。

## 参考

[深入解析Go-函数调用协议](https://tiancaiamao.gitbooks.io/go-internals/content/zh/03.0.html)
