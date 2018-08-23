---
title: golang channel的实现
date: 2018-05-06
tags: 
  - golang
---

[TOC]

interface是Go语言中最成功的设计之一，空的interface可以被当作“鸭子”类型使用，它使得Go这样的静态语言拥有了一定的动态性，但却又不损失静态语言在类型安全方面拥有的编译时检查的优势。

> 弱类型：字符串和数值可以自动转化
>
> 强类型：反之
>
> 动态类型：变量没有类型，但值有类型，变量进行绑定不同类型的值。运行时符号的类型可变。 
>
> 静态类型：变量有类型，编译时确定变量的类型，做类型检查。编译结束，变量的类型信息和存储布局都是确定的。

依赖于接口而不是实现，优先使用组合而不是继承，这是程序抽象的基本原则

## 底层实现

### Eface和Iface

interface在底层是一个结构体，包含了两个成员，分别表示类型信息和具体的数据。空接口（interface{}）和带方法的接口分别由Eface(empty interface)和IFace表示。

注： Eface和Iface是数据类型（built-in 和 type-define）转换成 interface 之后的实体的 struct 结构

```go
struct Eface
{
    type  *Type
    data  unsafe.Pointer
}

struct Iface
{
    tab   Itab*    
    data  unsafe.Pointer
}
```

#### Eface

Eface是interface{}底层使用的数据结构。Go中的任何对象都可以表示为interface{}，与C语言中的void*类型类似，不过interface{}中有类型信息，可以实现反射。

```go
struct Eface
{
    type  *_type
    data  unsafe.Pointer
}

struct _type
{
    size uintptr
    hash uint32
    _unused uint8 
    align uint8 
    fieldAlign uint8 
    kind uint8 
    alg *typeAlg
    gcdata *byte
    _string *string
    x *uncommonType
    ptrto *_ype
}

type uncommonType struct {
	pkgPath nameOff // import path; empty for built-in types like int, string
	mcount  uint16  // number of methods
	_       uint16  // unused
	moff    uint32  // offset from this uncommontype to [mcount]method
	_       uint32  // unused
}
```

在reflect包中有个KindOf函数，返回一个interface{}的Type，其实该函数就是简单的取Eface中的Type域

Type的UncommonType中有一个方法表，某个具体类型实现的所有方法都会被收集到这张表中。这里可能有疑问，interface{}为什么会有方法？上面也提到过，Eface其实是具体数据类型转换为interface之后的struct，这里的方法表就是具体类型实现的所有方法。`reflect`包中的`Method`和`MethodByName`方法都是通过查询这张表实现的。表中的每一项是一个`Method`

#### Iface

Iface和Eface略有不同，它是带方法的interface底层使用的数据结构。

```go
struct Iface
{
    tab   Itab*    
    data  unsafe.Pointer
}

struct Itab
{
    inter  *interfacetype //保存该接口的方法签名
    _type  *_type //保存动态类型的type类型信息
    link   *itab  //可能有嵌套的itab
    bad    int32
    inhash int32
    fun    [1]uintptr //保存动态类型对应的实现
}

type interfacetype struct {
	typ     _type
	pkgpath name
	mhdr    []imethod
}

struct _type
{
    size uintptr
    hash uint32
    _unused uint8 
    align uint8 
    fieldAlign uint8 
    kind uint8 
    alg *typeAlg
    gcdata *byte
    _string *string
    x *uncommonType
    ptrto *_ype
}

type uncommonType struct {
	pkgPath nameOff // import path; empty for built-in types like int, string
	mcount  uint16  // number of methods
	_       uint16  // unused
	moff    uint32  // offset from this uncommontype to [mcount]method
	_       uint32  // unused
}
```

Itab中不仅存储了Type信息，而且还多了一个方法签名inter和方法表fun。inter中保存了该接口的方法签名，具体类型中实现的方法会被拷贝到Itab的fun数组中。

注：func数组长度为1，因为内部把每个方法是按字典序排序存放的，fun中只存放了第一个方法，后续的紧挨着存放在后面。

### type assertion

具体T类型转换为接口类型：

1. 转换成空接口，涉及到两部分内容的复制：
   1. data指针指向原型数据
   2. type指针指向T类型的type
2. 转换为带方法的接口时，涉及了一道检测，该类型必须要实现了接口中声明的所有方法才可以进行转换。
   1. data指针指向原型数据
   2. Itab中的type指针指向T类型的type
   3. 查看T类型的_type方法表uncommentType是否包含了Itab字段inter中所有的imethod，同时将T类型对方法的实现拷贝到Itab的func中

**注：将对象赋值给接口时，编译期检查对象是否实现了接口所有的方法。运行时将对象的数据、类型、实现拷贝到iface接口当中**

接口类型转换为T类型，也是类似的：

```go
    v, ok := i.(T)
```

上面是判断一个接口i的具体类型是否为类型T，如果是则将其值返回给v。这跟上面的类型转换一样，也会检测转换是否合法。不过这里的检测是在**运行时**执行的。这个实现起来比较简单，只需要比较Iface中的Itab的type是否与给定T类型的type为同一个

上面涉及到三个方法表：

type的uncommonType中有一个方法表，某个具体类型实现的**所有方法**都会被收集到这张表中，**包括声明和实现**。reflect包中的Method和MethodByName方法都是通过查询这张表实现的。表中每一项都是method。其数据结构如下：

```go
struct Method
{
    String *name;  
    String *pkgPath;
    Type    *mtyp;
    Type *typ; //前面几个字段加起来是函数声明
    void (*ifn)(void); //函数实现
    void (*tfn)(void);
}
```

Iface的Itab的InterfaceType中也有一张方法表，这张方法表中是**接口所声明的方法**。其中每一项是一个IMethod，数据结构如下：

```go
struct IMethod
{
    String *name;
    String *pkgPath;
    Type *type;
}
```

跟上面的Method结构体对比可以发现，这里是只有声明没有实现的。

Itab的func域也是一张方法表，表中每一项是**一个函数指针，也就是只有实现没有声明**。即赋值的时候只是把具体类型的实现，即函数指针拷贝给了itab的func域。

注：type的uncommonType中的方法表，相当于Itad的inter和func的并集（声明+实现）

### reflect

虽然Go是静态语言，然而还是提供了reflect机制，以弥补静态语言在动态行为上的不足。

go的reflect库有两个重要的类型：reflect.Type和reflect.Value，Type,Value分别对应对象的**类型和值数据**，还有两个重要的函数：reflect.TypeOf(i interface{}) Type，reflect.ValueOf(i interface{}) Value

看下具体例子：

```go
type_ := reflect.TypeOf(obj)
field, _ := type_.FieldByName("hello")
```

这里取出来的 field 对象是 reflect.StructField 类型，但是它没有办法用来取得对应对象上的值。如果要取值，得用另外一套对object，而不是type的反射

```go
type_ := reflect.ValueOf(obj)
fieldValue := type_.FieldByName("hello")
```

这里取出来的 fieldValue 类型是 reflect.Value，它是一个具体的值，而**不是一个可复用的反射对象**。对比下java：

```java
Field field = clazz.getField("hello");
field.get(obj1);
field.get(obj2);
```

这个取得的反射对象类型是 java.lang.reflect.Field。它是可以复用的。

每次反射都需要做重复工作，所以导致了golang的reflect存在性能问题。reflect慢还有两个原因：一是涉及到内存分配以后GC；二是reflect实现里面有大量的枚举，也就是for循环，比如类型之类的。因此，一般情况下不建议使用反射。

对反射进行性能优化：

如果是 reflect.Type，可将其缓存，避免重复操作耗时。但 Value 显然不行，因为它和具体对象绑定，内部存储实例指针。

#### 总结

1. reflection比assert type更慢，其实两种做法都是O(n)的循环，原因是assert type会缓存在map中，而reflection没有缓存。参考[Why is reflect Type.Implements() so much slower than a type assertion?](https://stackoverflow.com/questions/46837485/why-is-reflect-type-implements-so-much-slower-than-a-type-assertion)
2. reflection之所以慢，因为存在循环，以及没有缓存，没法复用。这会增加runtime的开销，因为go为了保持编译器简单高效，所以很多工作都是在runtime完成的。
3. 缓存reflect.Type可以被缓存，reflect.Value没法缓存。

## 总结

why interface：

- writing generic algorithm （泛型编程）
- hiding implementation detail （隐藏具体实现）
- 非侵入式

### 泛型编程

go不支持泛型，官方的说法是：

> 尽管泛型很好，但是它会让我们的语言设计复杂度提升，所以我们现在暂时不打算支持，以后可能会支持。另外，虽然我们现在很弱，但是使用Interface也是可以实现泛型了

因此，使用interface可以实现泛型编程，是 duck-type programming 的一种体现。所谓duck-type，就是我们判断一个对象**不是通过它的类型定义来判断**，而是判断它是否**满足某些特定的方法和属性定义**。

在我们设计函数的时候，下面是一个比较好的准则。

> Be **conservative** in what you send, be **liberal** in what you accept. — Robustness Principle

对应到 Golang 就是：

> Return **concrete types**, receive **interfaces** as parameter. — Robustness Principle applied to Go

**就是传入具体类型，返回interface。**话说这么说，但是当我们翻阅 Golang 源码的时候，有些函数的返回值也是 interface。

#### interface{}与泛型的区别

范型，也就是任何类型，也就是不依赖于具体的数据类型，根据你传入的具体类型表现出不同行为。

go可以使用interface{}实现泛型，但是进行操作前，需要借助type assert或者reflection得到interface{}具体的类型。

然而java这种带泛型的语言，是语言天然支持的，type assert是在编译时完成的。

interface{}与泛型最主要区别：interface{}是runtime做type assert，而泛型是编译时做type assert。

接着来看几个在go中实现泛型的方式：

1. Copy & paste 

尽管这是一个看起来很笨的方法，但是结合实际应用情况，也许大多数情况下你只需要一两个类型就足够了，太早想到『优化』，带来的可能和你预期的有所出入。

- 好处 无需三方库，利用一些 IDE 或者编辑器插件，完成功能迅速。
- 缺点 代码有些臃肿，不符合 [Dry](https://link.zhihu.com/?target=https%3A//en.wikipedia.org/wiki/Don%2527t_repeat_yourself) 编程原则。

2. Interface （with method）

- 好处 无需三方库，代码干净而且通用。
- 缺点 需要一些额外的代码量，以及也许没那么夸张的运行时开销。

3. Use type assertions

- 好处 无需三方库，代码干净。
- 缺点 需要执行类型断言，接口转换的运行时开销，没有编译时类型检查。

4. Reflection

- 好处 干净
- 缺点  相当大的运行时开销，没有编译时类型检查。

5. Code generation

- 好处 非常干净的代码(取决工具)，编译时类型检查（有些工具甚至允许编写针对通用代码模板的测试），没有运行时开销。
- 缺点 构建需要第三方工具，如果一个模板为不同的目标类型多次实例化，编译后二进制文件较大。

#### 总结

1. go中的interface是鸭子类型：不通过类型定义来判断对象，而是通过它实现了哪些方法来判断
2. interface可以实现泛型，但是需要type assert和reflection，需要付出一些运行时开销

### 隐藏具体实现

比如我设计一个函数给你返回一个 interface，那么你只能通过 interface 里面的方法来做一些操作，但是内部的具体实现是完全不知道的。

比如context的三个方法：

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)    //返回 cancelCtx
func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc) //返回 timerCtx
func WithValue(parent Context, key, val interface{}) Context    //返回 valueCtx
```

表面上各个函数返回的是一个 Context interface，但是这个interface的具体实现是都是不同的struct。

### 非侵入式

首先你需要知道什么叫侵入式接口。以java为例，你需要**显式地创建一个类去实现一个接口**，这就是侵入式接口。

```java
public class MyWriter implements io.Writer {}
public class MyReader implements io.Reader {}
public class MyIO implements io.ReadWriter {}
```

而golang的例子中，我们并没有在代码的任何地方告诉MyIO需要去实现下面定义三个接口中的哪一个接口，如果它只实现了Read那就是Reader，如果两个都实现了就是ReadWriter，非常灵活。

不用为了实现一个接口而导入一个包了。想实现一个接口，直接实现它包含的方法就好了。

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

type ReadWriter interface {
    Reader
    Writer
}

type MyIO struct {}

func (io *MyIO) Read(p []byte) (n int, err error) {...}
func (io *MyIO) Write(p []byte) (n int, err error) {...}
```

传统的oo设计中，接口方是强势的，而非入侵式的接口设计中反而是更灵活的，更注重实用的。

这种非侵入式实现也带来了缺点：

1. 性能下降。使用 interface 作为函数参数，runtime 的时候会动态的确定行为。而使用 struct 作为参数，编译期间就可以确定了。
2. 不知道 struct 实现哪些 interface。这个问题可以使用 guru 工具来解决。

## 参考

[深入解析Go-高级数据结构](https://tiancaiamao.gitbooks.io/go-internals/content/zh/07.0.html)

[提高 golang 的反射性能](https://zhuanlan.zhihu.com/p/25474088)

[Go 性能优化技巧 8/10](https://segmentfault.com/a/1190000005052121)
