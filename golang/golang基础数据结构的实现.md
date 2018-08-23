---
title: golang基础数据结构的实现
date: 2018-05-05
tags: 
  - golang
---

[TOC]

## byte和rune

Go语言中byte和rune实质上就是uint8和int32类型。byte用来强调数据是raw data，而不是数字；而rune用来表示Unicode的code point。

```go
uint8       the set of all unsigned  8-bit integers (0 to 255)
int32       the set of all signed 32-bit integers (-2147483648 to 2147483647)

byte        alias for uint8
rune        alias for int32
```

go的源码使用的UTF-8，string中也是保存的UTF-8编码的序列，range循环取string时，每次取出的是一个UTF-8单位。

```go
const nihongo = "日本語"
for index, runeValue := range nihongo {
	fmt.Printf("%#U starts at byte position %d\n", runeValue, index)
}

//output
//U+65E5 '日' starts at byte position 0
//U+672C '本' starts at byte position 3
//U+8A9E '語' starts at byte position 6
```

关于Unicode和UTF-8等编码的知识，可以参考[Unicode 和 UTF-8 有何区别？](https://www.zhihu.com/question/23374078/answer/69732605)

**在go语言中，没有字符类型，rune就是字符类型**。Go 语言中的字符串是以 UTF-8 格式编码并存储的，通常我们认为，在电脑中存储一个 ASCII 字符需要一个字节，存储一个非 ASCII 字符需要两个字节，这种认为仅仅是针对 Windows 系统中常用的 ANSI 编码而言的。而在 Go 语言中，使用的是 UTF-8 编码，用 UTF-8 编码来存放一个 ASCII 字符依然只需要一个字节，而存放一个非 ASCII 字符，则需要 2个、3个、4个字节，它是不固定的。

遍历字符串中的字节（使用下标访问）：

```go
func main() {
	s := "Hello 世界！"
	for i, l := 0, len(s); i < l; i++ {
		fmt.Printf("%2v = %v\n", i, s[i]) // 输出单个字节值
	}
}
```

遍历字符串中的字符（使用 for range 语句访问）：

```go
func main() {
	s := "Hello 世界！"
	for i, v := range s { // i 是字符的字节位置，v 是字符的拷贝
		fmt.Printf("%2v = %c\n", i, v) // 输出单个字符
	}
}
```

## struct

先来定义一个简单的struct类型，名为Point，表示内存中两个相邻的整数。

![img](/Users/sinnera/sinnera.github.io/source/illustrations/godata1a.png)

`Point{10,20}`表示一个已初始化的Point类型。对它进行取地址表示一个指向刚刚分配和初始化的Point类型的指针。前者在内存中是两个词，而后者是一个指向两个词的指针。

结构体的域在内存中是紧挨着排列的。

```
type Rect1 struct { Min, Max Point }
type Rect2 struct { Min, Max *Point }
```

![img](/Users/sinnera/sinnera.github.io/source/illustrations/godata1b.png)

Rect1是一个具有两个Point类型属性的结构体，由在一行的两个Point--四个int代表。Rect2是一个具有两个`*Point`类型属性的结构体，由两个*Point表示。

## string

![img](/Users/sinnera/sinnera.github.io/source/illustrations/godata2.png)

（灰色的箭头表示已经实现的但不能直接可见的指针）

字符串在Go语言内存模型中用一个2字长的数据结构表示。它包含一个指向字符串存储数据的指针和一个长度数据。因为string类型是不可变的，对于多字符串共享同一个存储数据是安全的。切分操作`str[i:j]`会得到一个新的2字长结构，一个可能不同的但仍指向同一个字节序列(即上文说的存储数据)的指针和长度数据。这意味着**字符串切分不涉及内存分配或复制操作**。这使得字符串切分的效率等同于传递下标。

（说句题外话，在Java和其他语言里有一个有名的“疑难杂症”：在你分割字符串并保存时，对于源字符串的引用在内存中仍然保存着完整的原始字符串--即使只有一小部分仍被需要，Go也有这个“毛病”。另一方面，我们努力但又失败了的是，让字符串分割操作变得昂贵--包含一次分配和一次复制。在大多数程序中都避免了这么做。）

在 Go 语言中，字符串的内容是不能修改的，也就是说，你不能用 s[0] 这种方式修改字符串中的 UTF-8 编码，如果你一定要修改，那么你可以将字符串的内容复制到一个可写的缓冲区中，然后再进行修改。这样的缓冲区一般是 []byte或[]rune，取决于对字节进行修改，还是对字符进行修改。

**字符串拼接：**

- 如果是少量小文本拼接，用 “+” 就好

- 如果是大量小文本拼接，用strings.Join

  ```go
  s := "123"
  strings.Join([]string{s,"abc"},"")
  ```

- 如果是大量大文本拼接，用bytes.Buffer

  ```go
  var buf bytes.Buffer
  buf.WriteString("123")
  buf.WriteString("abc")
  buf.String()
  ```

- 还可以利用fmt.Sprintf

  ```go
  var s string = "123"
  s = fmt.Sprintf("%s %s",s,"abc")
  ```

性能：

总体来说，bytes.buffer是性能最好的，fmt.Sprintf性能最差，一般情况下，使用“+”足够了，如果要拼接大量字符，可以考虑使用strings.Join

## slice

一个slice是一个数组某个部分的引用。在内存中，它是一个包含3个域的结构体：指向slice中第一个元素的指针，slice的长度，以及slice的容量。长度是下标操作的上界，如x[i]中i必须小于长度。**容量是分割操作的上界，注意不是长度**，如x[i:j]中j不能大于容量。

![img](/Users/sinnera/sinnera.github.io/source/illustrations/godata3.png)

数组的slice并不会实际复制一份数据，它只是创建一个新的数据结构，包含了另外的一个指针，一个长度和一个容量数据。 如同分割一个字符串，分割数组也不涉及复制操作：它只是新建了一个结构来放置一个不同的指针，长度和容量。在例子中，对`[]int{2,3,5,7,11}`求值操作会创建一个包含五个值的数组，并设置x的属性来描述这个数组。分割表达式`x[1:3]`并不分配更多的数据：它只是写了一个新的slice结构的属性来引用相同的存储数据。在例子中，长度为2--只有y[0]和y[1]是有效的索引，但是容量为4--y[0:4]是一个有效的分割表达式。

### 扩容

```go
struct    Slice
{    // must not move anything
    byte*    array;        // actual data
    uintgo    len;        // number of elements
    uintgo    cap;        // allocated number of elements
};
```

在对slice进行append等操作时，可能会造成slice的自动扩容。其扩容时的大小增长规则是：

- 如果新的大小是当前大小2倍以上，则大小增长为新大小
- 否则循环以下操作：如果当前大小小于1024，按每次2倍增长，否则每次按当前大小1/4增长。直到增长的大小超过或等于新大小。

### make和new

Go有两个数据结构创建函数：new和make。两者的区别在学习Go语言的初期是一个常见的混淆点。基本的区别是`new(T)`返回一个`*T`，返回的这个指针可以被隐式地消除引用（图中的黑色箭头）。而`make(T, args)`返回一个普通的T。通常情况下，T内部有一些隐式的指针（图中的灰色箭头）。一句话，new返回一个指向已清零内存的指针，而make返回一个复杂的结构，并且只能是slice、map或channel。

换种说法，`new` 的作用是初始化一个指向类型的指针 (*T)， make 的作用是为 `slice`, `map` 或者 `channel` 初始化，并且返回引用 T。

![img](/Users/sinnera/sinnera.github.io/source/illustrations/godata4.png)

## map

### 数据结构

Go中的map在底层是用哈希表实现的，runtime/hashmap.go定义了map的基本结构和方法，runtime/hashmap_fast.go提供了一些快速操作map的函数。

map的底层结构是hmap（即hashmap的缩写），核心元素是一个由若干个桶（bucket，结构为bmap）组成的数组，每个bucket可以存放若干元素（通常是8个），key通过哈希算法被归入不同的bucket中。当超过8个元素需要存入某个bucket时，hmap会使用extra中的overflow来拓展该bucket。下面是hmap的结构体。

```go
type hmap struct {
	count     int // # 元素个数
	flags     uint8
	B         uint8  // 说明包含2^B个bucket
	noverflow uint16 // 溢出的bucket的个数
	hash0     uint32 // hash种子
 
	buckets    unsafe.Pointer // buckets的数组指针
	oldbuckets unsafe.Pointer // 结构扩容的时候用于复制的buckets数组
	nevacuate  uintptr        // 搬迁进度（已经搬迁的buckets数量）
 
	extra *mapextra
}
```

在extra中不仅有overflow，还有oldoverflow（用于扩容）和nextoverflow（prealloc的地址）。

bucket（bmap）的结构如下

> 注意：后面几行注释，hmap并非只有一个tophash，而是后面紧跟8组kv对和一个overflow的指针，这样才能使overflow成为一个链表的结构。但是这两个结构体并不是显示定义的，而是直接通过指针运算进行访问的。

```go
type bmap struct {
	// tophash generally contains the top byte of the hash value
	// for each key in this bucket. If tophash[0] < minTopHash,
	// tophash[0] is a bucket evacuation state instead.
	tophash [bucketCnt]uint8 //hash值的高8位....低位从bucket的array定位到bucket
	// Followed by bucketCnt keys and then bucketCnt values.
	// NOTE: packing all the keys together and then all the values together makes the
	// code a bit more complicated than alternating key/value/key/value/... but it allows
	// us to eliminate padding which would be needed for, e.g., map[int64]int8.
	// Followed by an overflow pointer.
    
    //下面是根据上面的注释自己加上的
    Bucket *overflow;           // 溢出桶链表，如果有
    byte   data[1];             // BUCKETSIZE keys followed by BUCKETSIZE values
}
```

![img](/Users/sinnera/sinnera.github.io/source/illustrations/hashmap.png)

hashmap 通过一个 bucket 数组实现，所有元素将被 hash 到数组中的 bucket 中，bucket 填满后，将通过一个 overflow 指针来扩展一个 bucket 出来形成链表，也就是解决冲突问题。这也就是一个基本的 hash 表结构，没什么新奇的东西，下面总结一些细节吧。

1. 注意一个 bucket 并不是只能存储一个 key/value 对，而是可以存储8个 key/value 对。每个 bucket 由 header 和 data 两部分组成，data 部分的内存大小为：(sizeof(key) + sizeof(value)) * 8，也就是要存储8对 key/value，这8对 key/value 在 data 内存中的存储顺序是：key0key1…key7value0value1…value7，是按照顺序先依次存储8个 key 值，然后存储对应的8个 value。 为什么不是存储为 key0value0…key7value7 呢？如果那么做，存储结构将变成key1/value1/key2/value2… 设想如果是这样的一个map[int64]int8，考虑到字节对齐，会浪费很多存储空间。（为了提高寻址速度）
2. 如果key, value 的类型大小超过了128字节，将不会直接存储值，而是存储其指针。
3. bucket 的 header 部分有一个 `uint8 tophash[8]` 数组，这个数组将用来存储8个 key 的 hash 值的高8位值。比如：tophash[0] 存储的值就是 hash(key0) » (64 - 8)。保存了一个 key 的 hash 高8位部分，在查找/删除/插入一个 key 的时候，可以先判断两个 key hash 的高8位是否相等，如果不等，那就根本不用去比较 key 的内容。所以这里保存一下 hash 值的高8位可以作为第一步的粗略过滤，不少时候可以省掉比较两个 key 的内容，因为比较两个 key 是否相等的代价远比两个 uint8 的代价高。当然，这里如果存储整个 hash 值，而不仅仅是高8位的话，判断效果将更好，但内存的占用就会多很多了。
4. bucket 的8个 key/value 空间如果都填满后，就会分配新的 bucket，通过 overflow 指针串联起来。注意这个链表指针被命名为 overflow，代表的正是 bucket 溢出了，这个命名感觉很好，hash 表实现的时候我们应该努力避免 bucket overflow。
5. hashmap 是会自增长的，也就说随着插入的 kv 对越来越多，初始的 bucket 数组就可以需要增长，进行rehash，性能才会好。bucket 数组增长的时机就是插入的元素个数大于了 `bucket数组大小 * 6.5`，为什么是6.5（装载因子），这个在代码注释里有说明，主要是测试出来的经验值。
6. hashmap 每次增长，都是重新分配一个新的 bucket 数组，新 bucket 数组是之前 bucket 数组的2倍大小。
7. hashmap 增长后，需要将老 bucket 数组中的元素拷贝到新的 bucket 数组，这个拷贝过程不是一口气立马完成的，而是采用了增量式的拷贝，也就是说分配了新的 bucket 数组后，并没有立刻拷贝元素，而是等接下来每次插入一个元素的时候，才拷贝一点，随着插入的动作增多，逐渐就将全部元素拷贝到了新的 bucket 数组中。
8. 在 make 一个 map 对象的时候，如果不指定大小的话，bucket 数组默认就是1了，随着插入的元素增多，就会增长成2，4，8，16等。可以看出不指定初始化大小的map，很可能要经历很多次的增长、元素拷贝。我们应该给 map 指定一个合适的大小值。

### 优缺点

1. HMap中是Bucket的数组，而不是Bucket指针的数组。可以一次分配较大内存，减少了分配次数，避免多次调用mallocgc。
2. key/val的存储顺序，减少了字节对齐浪费的空间，提高了寻址速度
3. 如果key, value 的类型大小超过了128字节，将不会直接存储值，而是存储其指针
4. tophash[8]用于加快定位，类似于缓存的作用
5. rehash的时机是装载因子为6.5，采用渐进式拷贝
6. map 不会收缩 “不再使用” 的空间。就算把所有键值删除，它依然保留内存空间以待后用
7. 改进方向：empty的bucket合并；table中元素很少时，考虑收缩

### 常见问题

Q：删除掉map中的元素是否会释放内存？

A：不会，删除操作仅仅将对应的tophash[i]设置为empty，并非释放内存。若要释放内存只能等待指针无引用后被系统gc 

Q：如何并发地使用map？

A：map不是goroutine安全的，所以在有多个gorountine对map进行写操作是会panic。多gorountine读写map是应加锁（RWMutex），或使用sync.Map（1.9新增，在下篇文章中会介绍这个东西，总之是不太推荐使用）。

Q：map的iterator是否安全？

A：map的delete并非真的delete，所以对迭代器是没有影响的，是安全的。

Q：针对map的实现，应该如何正确地使用map，才能达到最优性能？

A：

1. 预设容量，避免rehash

2. 对于小对象，直接将数据交由 map 保存，远比用指针高效。这不但减少了堆内存分配，关键还在于垃圾回收器不会扫描非指针类型 key/value 对象
3. map不会自己收缩，长期使用map对象（比如用作cache容器），偶尔换成 “新的” 或许会更好

## container

### heap

heap包为实现了heap.Interface的类型提供了堆方法：Init/Push/Pop/Remove/Fix。（注：Fix用于元素的值发生变化之后， 重新修复堆的有序性）

```go
type Interface interface {
	sort.Interface
	Push(x interface{}) // add x as element Len()
	Pop() interface{}   // remove and return element Len() - 1.
}
```

使用时，需要实现如下方法：Len/Less/Swap, Push/Pop。（注：默认是小根堆，如果Less方法里面反着写就是大根堆）

```go
package main

import (
	"fmt"
	"container/heap"
)

// IntHeap 是一个由整数组成的最小堆。
type IntHeap []int

func (h IntHeap) Len() int           { return len(h) }
func (h IntHeap) Less(i, j int) bool { return h[i] > h[j] }
func (h IntHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }

func (h *IntHeap) Push(x interface{}) {
	// Push 和 Pop 使用 pointer receiver 作为参数，
	// 因为它们不仅会对切片的内容进行调整，还会修改切片的长度。
	*h = append(*h, x.(int))
}

func (h *IntHeap) Pop() interface{} {
	old := *h
	n := len(old)
	x := old[n-1]
	*h = old[0 : n-1]
	return x
}

// 这个示例会将一些整数插入到堆里面，接着检查堆中的最小值，
// 之后按顺序从堆里面移除各个整数。
func main() {
	h := &IntHeap{2, 1, 5}
	heap.Init(h)
	heap.Push(h, 3)
	fmt.Printf("minimum: %d\n", (*h)[0])
	for h.Len() > 0 {
		fmt.Printf("%d ", heap.Pop(h))
	}
}
```

还常用于构造优先级队列。

### list

list是双向链表，可以开箱即用。

- 对list的操作是串行的，则需要注意检查元素指针是否为nil，避免程序崩溃
- 并发处理list时，建议对list进行加写锁（全局锁），然后再操作。注意，读写锁无法保证并行处理list时程序的安全性。

```go
type Element struct {
    next, prev *Element  // 上一个元素和下一个元素
    list *List  // 元素所在链表
    Value interface{}  // 元素
}

type List struct {
    root Element  // 链表的根元素
    len  int      // 链表的长度
}
```

可以用list来实现stack

```go
 package stack
 
 import "container/list"
 
 type Stack struct {
     list *list.List
 }
 
 func NewStack() *Stack {
     list := list.New()
     return &Stack{list}
 }
 
 func (stack *Stack) Push(value interface{}) {
     stack.list.PushBack(value)
 }
 
 func (stack *Stack) Pop() interface{} {
     e := stack.list.Back()
     if e != nil {
         stack.list.Remove(e)
         return e.Value
     }
     return nil
 }
 
 func (stack *Stack) Peak() interface{} {
     e := stack.list.Back()
     if e != nil {
         return e.Value
     }
 
     return nil
 }
 
 func (stack *Stack) Len() int {
     return stack.list.Len()
 }
 
 func (stack *Stack) Empty() bool {
     return stack.list.Len() == 0
 }
```

### ring

ring是一个环形链表。

```go
type Ring struct {
    next, prev *Ring
    Value      interface{}
}
```

ring提供一个Do方法，能在遍历的时候，对每个元素执行一个function。 

```go
// This example demonstrates an integer heap built using the heap interface.
package main

import (
    "container/ring"
    "fmt"
)

func main() {
    ring := ring.New(3)

    for i := 1; i <= 3; i++ {
        ring.Value = i
        ring = ring.Next()
    }

    // 计算1+2+3
    s := 0
    ring.Do(func(p interface{}){
        s += p.(int)
    })
    fmt.Println("sum is", s)
}

output:
sum is 6
```

ring的使用场景，目前只了解到可以用于实现约瑟夫环。

## 参考

[深入解析Go-基本数据结构](https://tiancaiamao.gitbooks.io/go-internals/content/zh/02.0.html)

[Go 性能优化技巧 3/10](https://www.jianshu.com/p/c34e3a787de4)

[Unicode 和 UTF-8 有何区别？](https://www.zhihu.com/question/23374078/answer/69732605)