---
title: golang context详解和底层实现
date: 2018-05-16
tags: 
  - golang
---

[TOC]

## 为什么需要context

在server中，当某个上游请求退出或者超时，所有下游服务也需要退出，因为继续执行下去已经没有意义。举个例子，RPC 1里面，调用了RPC 2，RPC 3，RPC 4，其中RPC 2调用失败了，传统情况下这里有两种做法：

1. 等待其他两个RPC执行完，并返回错误。其实，这里其他两个RPC的执行结果已经没有意义了，因为我们依旧需要给用户返回错误。
2. 直接返回错误。但是其他两个RPC依旧在没意义的运行，浪费了资源。

理想情况下，应该在RPC 2出错后，通知其他两个RPC停止运行，并返回错误，这样就不会造成资源浪费。

![img](https://github.com/SinnerA/blog/blob/master/illustrations/v2-eff89f011b2f456f0f11fba2823da3bf_hd.png)

由于多个RPC可能在不同的goroutine里处理，因此问题变成如何通知不同goroutine，这里自然会想到使用channel来通知结束。其实，**context的核心就是用一个done chan实现的**。

## 介绍

context包原本是Google开发并开源的，从go1.7开始，golang.org/x/net/context包正式作为context包进入了标准库。通过context，我们可以方便地对同一个请求所产生地goroutine进行约束管理，可以设定超时、deadline，甚至是取消这个请求相关的所有goroutine。

### 数据结构

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool) //超时时间
    Done() <-chan struct{} //ctx被cancel时，通知结束的channel
    Err() error            //ctx被cancel原因
    Value(key interface{}) interface{} //在goroutie之间传递的共享变量
}
```

调用链类似于一棵树，节点之间相连，并往下衍生。Context的结构也是像一棵树，存在根节点，子节点继承父节点，衍生下去。Context中的根节点就是emptyCtx：

```go
type emptyCtx int
```

`emptyCtx`不能被取消，没有值，也没有deadline，因为它虽然实现了Context，但是每个函数都是直接return。Context提供了两个函数用于直接返回根节点emptyCtx：

```go
var (
    background = new(emptyCtx)
    todo       = new(emptyCtx)
)

func Background() Context { //用在请求的顶层
    return background
}

func TODO() Context {
    return todo
}
```

context.Background()一般用在请求的顶层，返回的emptyCtx作为根节点；调用某个带ctx参数的函数时，如果不知道是否要用context的话，用`context.TODO()`来替代，千万不要传入一个nil的context。

### 函数

有了根节点，如何创建子节点？context提供了以下函数：

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
func WithValue(parent Context, key interface{}, val interface{}) Context
```

#### WithCancel

WithCancel它返回了一个Context接口，其实是返回了cancelCtx结构体，该结构体实现了Context接口：

```go
type cancelCtx struct {
    Context

    done chan struct{} // closed by the first cancel call.

    mu       sync.Mutex
    children map[canceler]struct{} // set to nil by the first cancel call
    err      error                 // set to non-nil by the first cancel call
}
```

上面可以看出，cancelCtx是通过匿名成员变量实现了Context接口的，通过层层包含，也实现了节点的继承关系。它实现的方法：

```go
func (c *cancelCtx) Done() <-chan struct{} {
    return c.done
}

func (c *cancelCtx) Err() error {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.err
}

func (c *cancelCtx) String() string {
    return fmt.Sprintf("%v.WithCancel", c.Context)
}

// cancel closes c.done, cancels each of c's children, and, if
// removeFromParent is true, removes c from its parent's children.
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
    if err == nil {
        panic("context: internal error: missing cancel error")
    }
    c.mu.Lock()
    if c.err != nil {
        c.mu.Unlock()
        return // already canceled
    }
    c.err = err
    // 关闭c的done channel，所有监听c.Done()的goroutine都会收到消息
    close(c.done)
    // 取消child，由于是map结构，所以取消的顺序是不固定的
    for child := range c.children {
        // NOTE: acquiring the child's lock while holding parent's lock.
        child.cancel(false, err)
    }
    c.children = nil
    c.mu.Unlock()
    // 从c.children中移除取消过的child
    if removeFromParent {
        removeChild(c.Context, c)
    }
}
```

当我们在调用`WithCancel`的时候，实际上返回的就是一个`cancelCtx`指针和`cancel()`方法：

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
    c := newCancelCtx(parent)
    propagateCancel(parent, &c)
    return &c, func() { c.cancel(true, Canceled) }
}

// newCancelCtx returns an initialized cancelCtx.
func newCancelCtx(parent Context) cancelCtx {
    return cancelCtx{
        Context: parent, //包含父节点，通过层层包含，实现了树
        done:    make(chan struct{}),
    }
}
```

那么，`propagateCancel`函数又是干什么的呢？

```go
// 向上找到最近的可以被取消的父context，将子context放入parent.Children中
// propagateCancel arranges for child to be canceled when parent is.
func propagateCancel(parent Context, child canceler) {
    if parent.Done() == nil {
        return // parent is never canceled
    }
    // 判断返回的parent是否是cancelCtx
    if p, ok := parentCancelCtx(parent); ok {
        p.mu.Lock()
        if p.err != nil {
            // parent has already been canceled
            child.cancel(false, p.err)
        } else {
            if p.children == nil {
                p.children = make(map[canceler]struct{})
            }
            p.children[child] = struct{}{} //以便于parent cancel时，就是cancel掉children
        }
        p.mu.Unlock()
    } else {
        //parent不是cancelCtx的话，起一个goroutine等待parent结束，然后cancel掉自己
        go func() {
            select {
            case <-parent.Done():
                child.cancel(false, parent.Err())
            case <-child.Done():
            }
        }()
    }
}

// parentCancelCtx follows a chain of parent references until it finds a
// *cancelCtx. This function understands how each of the concrete types in this
// package represents its parent.
// 不停地向上寻找最近的可取消的父context
func parentCancelCtx(parent Context) (*cancelCtx, bool) {
    for {
        switch c := parent.(type) {
        case *cancelCtx:
            return c, true
        case *timerCtx:
            return &c.cancelCtx, true
        case *valueCtx:
            parent = c.Context
        default:
            return nil, false //注意，如果是自定义的ctx，return false
        }
    }
}
```

#### WithTimeout和WithDeadline

`WithTimeout`和`WithDeadline`其实是差不多的，都是用于超时，返回的都是timerCtx，只不过WithTimeout是传入相对时间，而WithDeadline是传入绝对时间，：

```go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
    return WithDeadline(parent, time.Now().Add(timeout))
}

func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc) {
    // 当前的deadline比新的deadline还要早，直接返回
    if cur, ok := parent.Deadline(); ok && cur.Before(deadline) {
        // The current deadline is already sooner than the new one.
        return WithCancel(parent)
    }
    c := &timerCtx{
        cancelCtx: newCancelCtx(parent),
        deadline:  deadline,
    }
    propagateCancel(parent, c)
    d := time.Until(deadline)
    // deadline已经过了，不再设置定时器
    if d <= 0 {
        c.cancel(true, DeadlineExceeded) // deadline has already passed
        return c, func() { c.cancel(true, Canceled) }
    }
    c.mu.Lock()
    defer c.mu.Unlock()
    if c.err == nil {
        // 设定d时间后执行取消方法
        c.timer = time.AfterFunc(d, func() {
            c.cancel(true, DeadlineExceeded)
        })
    }
    return c, func() { c.cancel(true, Canceled) }
}
```

上面实现可以看出，如果子节点设置的超时时间，比父节点的超时时间还晚，会设置失败，还是使用父节点的超时时间。`timerCtx`的代码也实现地比较简洁：

```go
type timerCtx struct {
    cancelCtx
    timer *time.Timer // Under cancelCtx.mu.

    deadline time.Time
}

func (c *timerCtx) Deadline() (deadline time.Time, ok bool) {
    return c.deadline, true
}

func (c *timerCtx) String() string {
    return fmt.Sprintf("%v.WithDeadline(%s [%s])", c.cancelCtx.Context, c.deadline, time.Until(c.deadline))
}

func (c *timerCtx) cancel(removeFromParent bool, err error) {
    c.cancelCtx.cancel(false, err)
    if removeFromParent {
        // Remove this timerCtx from its parent cancelCtx's children.
        removeChild(c.cancelCtx.Context, c)
    }
    c.mu.Lock()
    if c.timer != nil {
        c.timer.Stop()
        c.timer = nil
    }
    c.mu.Unlock()
}
```

这里需要注意的是，`timerCtx`同样使用了匿名成员变量cancelCtx，这样timerCtx也实现了cancelCtx的方法，而这里按需只实现了`Deadline`方法。`timerCtx`的`cancel`方法先调用了`cancelCtx`的`cancel`方法，然后再去停止定时器。

 #### WithValue

这个方法是用来传递在这次的请求处理中相关goroutine的**共享变量**，这与全局变量是有所区别的，因为它只在这次的请求范围内有效。常用于传递requestId。

```go
func WithValue(parent Context, key, val interface{}) Context {
    if key == nil {
        panic("nil key")
    }
    if !reflect.TypeOf(key).Comparable() {
        panic("key is not comparable")
    }
    return &valueCtx{parent, key, val}
}
```

key，value变量来存储值。看看`valueCtx`的定义：

```go
type valueCtx struct {
    Context
    key, val interface{}
}

func (c *valueCtx) String() string {
    return fmt.Sprintf("%v.WithValue(%#v, %#v)", c.Context, c.key, c.val)
}

func (c *valueCtx) Value(key interface{}) interface{} {
    // 找到了直接返回
    if c.key == key {
        return c.val
    }
    // 向上继续查找
    return c.Context.Value(key)
}
```

 可以看出，WithValue只返回了一个Context，并没有像上面三个函数一样，返回cancel方法，因此WithValue返回的ctx不能自己主动cancel。 

注意在同一个context中设置key/value，若key相同，值会被覆盖。`Value()`查询key对应的value数据，会从当前context中查询，如果查不到，会递归查询父context中的数据。可以看出，**context中的上下文数据并不是全局的，它只查询本节点及父节点们的数据，不能查询兄弟节点的数据。**

总结：把KV和canceling混合在一个结构体里，确实是实用主义的体现。context是goroutine safe的，想想如果没有他，还得自己封装一个map + sync.Mutex的结构体。

## 注意

### context的锁争用

context是一层一层往下传的，如果全局都是使用同一个传递下来的context，会出现一个问题：锁争用。

```go
select {
    case <-context.Done():
}
```

大家都在同一个对象上面调用的Done函数，channel操作最终会加锁。在起goroutine的时候，一般不要用原来的context了，而是新建一个context，原始的context作为父context。这样不同goroutine就不会抢同一个锁。

一般是用的`context.WitCancel()`这个函数：

```go
go func() {
    ctx, cancel = context.WithCancel(ctx)
    doSomething(ctx)
    cancel()
}
```

调用WithCancel的时候，会得到一个新的子context，以及一个cancel函数。子ctx会在父context的Done函数收到信号，或者cancel被调用的情况下收到Done信号。

### 自定义context

如果用户自定义了一个context：

```go
type Example struct {
    context.Context
    ...
}
```

拿它当 context 使用时，每次WithCancel时，在propagateCancel函数中，后台会新起一个goroutine用于监控parent是否cancel。

结论就是，WithCancel 对标准库的几个 context 实现做了特殊优化，不会开启 goroutine，然而对用户实现的 context 非常不友好，会额外开启 goroutine。

### 不被cancel的context

如果某个服务需要稳定性保障，穿ctx进去的时候，最好传入一个不被cancel的ctx，否则上游服务发生错误时，ctx被cancel，则稳定性得不到保障。

## 参考

[Go context源码解析](https://www.jianshu.com/p/4d199f181024)

[Go 语言坑爹的 WithCancel](http://www.zenlife.tk/with-cancel.md)

[Go的context的问题](http://www.zenlife.tk/go-context.md)
