---
title: go同步方式
date: 2017-12-03 16:39:08
tags: golang 编程语言
---

### channel

channel是goroutine之间同步的主要方法。

produce和consume：

- 当channel已满（无缓冲channel或者有缓冲channel已满），produce会阻塞，先有consume才能produce
- 当channel未满且无数据（有缓冲chennel中无数据），consume会阻塞，先有produce才能consume
- 当channel未满且有数据，produce和consume都不会阻塞

### 锁

sync包实现了2种锁数据结构，sync.Mutex和sync.RWMutex。

### Once

sync包提供了在多个goroutine环境下初始化机制，即类型Once。

对于特定的函数f，多个线程都可以执行once.Do(f)，但是只有一个会执行f，其它线程对f的调用都会被阻塞，直到f()返回。

### 不正确的同步方式

