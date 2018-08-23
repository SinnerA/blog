[TOC]

## 介绍

1. HTTP2 多路复用
2. protobuf序列化反序列化
3. 多语言

### 多语言

三种语言的实现：C/C++、Java和Go

其中其他语言，比如Object-C、Python、PHP、Ruby等等，都是基于C底层库提供的API实现的

### protobuf

序列化 & 反序列化属于通讯协议的一部分

- 序列化 / 反序列化 属于 `TCP/IP`模型 应用层 和OSI模型 展示层的主要功能：
  1. （序列化）把 应用层的对象 转换成 二进制串
  2. （反序列化）把 二进制串 转换成 应用层的对象
- 所以， `Protocol Buffer`属于 `TCP/IP`模型的应用层 & `OSI`模型的展示层

protobuf相比较XML和json，具有如下的特点：

![img](http://upload-images.jianshu.io/upload_images/944365-a9b3fc2ed16f61e5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 特点总结

1. 性能：
   - 体积：体积小，序列化之后数据大小可缩小3倍左右

   - 速度：比XML和json快20-100倍
   - 传输：体积小传输带宽有优势
2. 使用维护：
   - 跨平台跨语言：共享一套proto文件
   - 加密：http传输内容都是二进制流（不可读）
3. 缺点：
   - 二进制可读性差，不具有自描述特性

#### 原理

1. 体积小：
   - 不同字段采用不同编码（定长，变长），如`Varint`、`Zigzag`编码方式
   - 采用`T - L - V` 的数据存储方式：减少了分隔符的使用 & 数据存储得紧凑

2. 速度快：
   - 编码方式简单（只需要简单的移位补位等） 
   - 采用 Protocol Buffer 自身的框架代码和编译器共同完成


### HTTP2

HTTP2提供了多路复用功能，可以使得gRPC支持stream

## RPC性能核心要素

决定RPC性能的三个核心要素：协议、线程模型和IO模型

### 协议

gRPC 是 Google 基于 HTTP/2 以及 protobuf 的，要了解 gRPC 协议，只需要知道 gRPC 是如何在 HTTP/2 上面传输就可以了。

gRPC 通常有四种模式，unary，client streaming，server streaming 以及 bidirectional streaming，**对于底层 HTTP/2 来说，它们都是 stream，并且仍然是一套 request + response 模型**。

#### Request

gRPC 的 request 通常包含 Request-Headers, 0 或者多个 Length-Prefixed-Message 以及 EOS。

Request-Headers 直接使用的HTTP/2的HEADERS frame ，在 HEADERS 和 CONTINUATION frame 里面派发。定义的 header 主要有 Call-Definition 以及 Custom-Metadata。Call-Definition 里面包括 Method（其实就是用的 HTTP/2 的 POST），Content-Type 等。而 Custom-Metadata 则是应用层自定义的任意 key-value，key 不建议使用 `grpc-` 开头，因为这是为 gRPC 后续自己保留的。

Length-Prefixed-Message 主要在 DATA frame 里面派发，它有一个 Compressed flag 用来表示该 message 是否压缩，如果为 1，表示该 message 采用了压缩，而压缩算啊定义在 header 里面的 Message-Encoding 里面。然后后面跟着四字节的 message length 以及实际的 message。

EOS（end-of-stream） 会在最后的 DATA frame 里面带上了 `END_STREAM` 这个 flag。用来表示 stream 不会在发送任何数据，可以关闭了。

#### Response

Response 主要包含 Response-Headers，0 或者多个 Length-Prefixed-Message 以及 Trailers。如果遇到了错误，也可以直接返回 Trailers-Only。

Response-Headers 主要包括 HTTP-Status，Content-Type 以及 Custom-Metadata 等。Trailers-Only 也有 HTTP-Status ，Content-Type 和 Trailers。Trailers 包括了 Status 以及 0 或者多个 Custom-Metadata。

HTTP-Status 就是我们通常的 HTTP 200，301，400 这些，很通用就不再解释。Status 也就是 gRPC 的 status， 而 Status-Message 则是  gRPC 的 message。Status-Message 采用了 Percent-Encoded 的编码方式，具体参考[这里](https://link.jianshu.com?t=https://tools.ietf.org/html/rfc3986#section-2.1)。

如果在最后收到的 HEADERS frame 里面，带上了 Trailers，并且有 `END_STREAM` 这个 flag，那么就意味着 response 的 EOS。

### 总结

1. request：header + content + endflag，header包含Method，Content-Type和Custom-Metadata，content包含压缩flag + length + content
2. response：header + content + endflag，header包含Status，Content-Type和Custom-Metadata，content包含压缩flag + length + content
3. 对于底层 HTTP/2 来说，它们都是 stream，并且仍然是一套 request + response 模型
4. grpc传输的request和response主要使用了HTTP2的HEADERS和DATA帧，其中Request/Response的header用的是HEADERS，message用的是DATA。

### 线程模型

gRPC 的线程模型主要包括服务端线程模型和客户端线程模型

服务端线程模型：

1. 服务端监听和客户端接入线程（HTTP2 Acceptor）
2. 网络IO读写线程
3. 服务接口调用线程

客户端线程模型：

1. 客户端连接线程（HTTP2  Connector）
2. 网络IO读写线程
3. 接口调用线程
4. 响应回调通知线程

### IO模型

非阻塞IO + 多路复用

go将非阻塞变成阻塞IO

## 详细设计

### 服务抽象

一个gRPC Server可以包含多个Service，每个Service包含多个方法。换句话说，一个RPC Server可以提供多个服务：

```go
// Server is a gRPC server to serve RPC requests.
type Server struct {
	opts options
	lis    map[net.Listener]bool
	conns  map[io.Closer]bool
	m      map[string]*service // service name -> service info
    //......
}

// service consists of the information of the server serving this service and
// the methods in this service.
type service struct {
	server interface{} // the server for service methods
	md     map[string]*MethodDesc
	sd     map[string]*StreamDesc
	mdata  interface{}
}

// MethodDesc represents an RPC service's method specification.
type MethodDesc struct {
	MethodName string
	Handler    methodHandler
}
```

应用开发者需要：

- 不使用protobuf
  - 规定Service对应的ServiceDesc。ServiceDesc描述了Service的服务名、处理此服务的接口类型（Service接口）、单次调用的方法数组、流式方法数组、其他元数据。
  - 定义结构体srv实现此ServiceDesc中描述的的各个方法，也就是实现Service接口
  - 实例化Server，并将srv和ServiceDesc注册到Server。此时Server会记录srv和它实现的所有方法，用于client请求调用
  - 监听并启动Server服务
- 使用protobuf
  - 实现protobuf grpc插件生成的ServiceDesc和Service接口
  - 实例化Server，并注册Service接口的具体实现
  - 监听并启动Server

可见，protobuf的gRPC-Go插件帮助我们生成了Service的接口和ServiceDesc。

### 底层传输

客户端和服务端的连接在应用层由Transport抽象，在客户端是ClientTransport，在服务端是ServerTransport。

客户端的每个http2请求会打开一个新的流。流可以从两边关闭，对于单次请求来说，客户端会主动关闭流，对于流式请求客户端不会主动关闭。Server端接收到一个客户端的http2请求后即打开一个新的Stream，ClientTransport和ServerTransport之间使用这个新打开的Stream以http2帧的形式交换数据。

### 服务端流程

主要流程：

1. 实例化Server
2. 注册Service
3. 监听并接收连接请求
4. 连接与请求处理
5. 连接的处理细节(http2连接的建立)
6. 新请求的处理细节(新流的打开和帧数据的处理)

#### 实例化Server

工厂方法new一个rpc server，server中维护了监听地址列表、客户端连接列表、服务service列表和可选项

```go
type Server struct {
  opts options //可选项
  lis map[net.Listener]bool // 监听地址列表
  conns map[io.Closer]bool // 客户端的连接
  m map[string]*service // service name -> service info
  //......
}
```

#### 注册Service

将ServiceDesc和实现了Service接口的srv注册到rpc server中：

1. 判断srv是否实现了Service接口的所有方法
2. 在server中维护了service的信息：实现了Service接口的srv和它所实现的所有方法，用于client请求调用

#### 监听并接收连接请求

go的典型server处理方式：for循环中使用Accept接收新的请求，每个新的tcp连接使用单独的goroutine处理。

本质还是epoll处理，只不过把非阻塞变成阻塞的方式。

#### 连接与请求处理

每个http2连接在服务端会生成一个ServerTransport

### 客户端流程















