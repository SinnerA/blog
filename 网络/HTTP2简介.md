[TOC]

gRPC 是基于 HTTP/2 协议的，要深刻理解 gRPC，理解下 HTTP/2 是必要的，这里先简单介绍一下 HTTP/2 相关的知识，然后在介绍下 gRPC 是如何基于 HTTP/2 构建的。

## HTTP/1.x

HTTP 协议可以算是现阶段 Web 上面最通用的协议了，在之前很长一段时间，很多应用都是基于 HTTP/1.x 协议，HTTP/1.x 协议是一个文本协议，可读性非常好，但其实并不高效，笔者主要碰到过几个问题：

### Parser

如果要解析一个完整的 HTTP 请求，首先我们需要能正确的读出 HTTP header。HTTP header 各个 fields 使用 `\r\n` 分隔，然后跟 body 之间使用 `\r\n\r\n` 分隔。解析完 header 之后，我们才能从 header 里面的 `content-length` 拿到 body 的 size，从而读取 body。

这套流程其实并不高效，因为我们需要读取多次，才能将一个完整的 HTTP 请求给解析出来，虽然在代码实现上面，有很多优化方式，譬如：

- 一次将一大块数据读取到 buffer 里面避免多次 IO read
- 读取的时候直接匹配 `\r\n` 的方式流式解析

但上面的方式对于高性能服务来说，终归还是会有开销。其实**最主要的问题在于，HTTP/1.x 的协议是 文本协议，是给人看的，对机器不友好，如果要对机器友好，二进制协议才是更好的选择。**

如果大家对解析 HTTP/1.x 很感兴趣，可以研究下 [http-parser](https://link.jianshu.com?t=https://github.com/nodejs/http-parser)，一个非常高效小巧的 C library，见过不少框架都是集成了这个库来处理 HTTP/1.x 的。

### Request/Response

HTTP/1.x 另一个问题就在于它的交互模式，一个连接每次只能一问一答，也就是client 发送了 request 之后，必须等到 response，才能继续发送下一次请求。

这套机制是非常简单，但会造成网络连接利用率不高。如果需要同时进行大量的交互，client 需要跟 server 建立多条连接，但连接的建立也是有开销的，所以为了性能，通常这些连接都是长连接一直保活的，虽然对于 server 来说同时处理百万连接也没啥太大的挑战，但终归效率不高。

### Push

用 HTTP/1.x 做过推送的同学，大概就知道有多么的痛苦，因为 HTTP/1.x 并没有推送机制。所以通常两种做法：

- Long polling 方式，也就是直接给 server 挂一个连接，等待一段时间（譬如 1 分钟），如果 server 有返回或者超时，则再次重新 poll。
- Web-socket，通过 upgrade 机制显式的将这条 HTTP 连接变成裸的 TCP，进行双向交互。

相比 Long polling，笔者还是更喜欢 web-socket 一点，毕竟更加高效，只是 web-socket 后面的交互并不是传统意义上面的 HTTP 了。

### 总结

1. HTTP1.X是文本协议，不是二进制协议，解析费劲，需要根据 `\r\n` 区匹配解析
2. 一个连接只能一问一答，网络连接利用率不高，大量的交互需要多条连接

## Hello HTTP/2

虽然 HTTP/1.x 协议可能仍然是当今互联网运用最广泛的协议，但随着 Web 服务规模的不断扩大，HTTP/1.x 越发显得捉紧见拙，我们急需另一套更好的协议来构建我们的服务,于是就有了 HTTP/2。

HTTP/2 是一个二进制协议，这也就意味着它的可读性几乎为 0，但幸运的是，我们还是有很多工具，譬如 Wireshark， 能够将其解析出来。

在了解 HTTP/2 之前，需要知道一些通用术语：

- Stream： 一个双向流，一条连接可以有多个 streams。
- Message： 也就是逻辑上面的 request，response。
- Frame:：数据传输的最小单位。每个 Frame 都属于一个特定的 stream 或者整个连接。一个 message 可能有多个 frame 组成。

### Frame Format

Frame 是 HTTP/2 里面最小的数据传输单位，一个 Frame 定义如下（[直接从官网 copy 的](https://link.jianshu.com?t=http://httpwg.org/specs/rfc7540.html#rfc.section.4.1)）：

```
+-----------------------------------------------+
|                 Length (24)                   |
+---------------+---------------+---------------+
|   Type (8)    |   Flags (8)   |
+-+-------------+---------------+-------------------------------+
|R|                 Stream Identifier (31)                      |
+=+=============================================================+
|                   Frame Payload (0...)                      ...
+---------------------------------------------------------------+
```

Length：也就是 Frame 的长度，默认最大长度是 16KB，如果要发送更大的 Frame，需要显式的设置 max frame size。
Type：Frame 的类型，譬如有 DATA，HEADERS，PRIORITY 等。
Flag 和 R：保留位，可以先不管。
Stream Identifier：标识所属的 stream，如果为 0，则表示这个 frame 属于整条连接。
Frame Payload：根据不同 Type 有不同的格式。

可以看到，Frame 的格式定义还是非常的简单，按照官方协议，可以非常方便的写一个出来。

### Multiplexing

HTTP/2 通过 stream 支持了连接的多路复用，提高了连接的利用率。Stream 有很多重要特性：

- 一条连接可以包含多个 streams，多个 streams 发送的数据互相不影响。
- Stream 可以被 client 和 server 单方面或者共享使用。
- Stream 可以被任意一段关闭。
- Stream 会确定好发送 frame 的顺序，另一端会按照接受到的顺序来处理。
- Stream 用一个唯一 ID 来标识。

这里在说一下 Stream ID，如果是 client 创建的 stream，ID 就是奇数，如果是 server 创建的，ID 就是偶数。ID 0x00 和 0x01 都有特定的使用场景。

Stream ID 不可能被重复使用，如果一条连接上面 ID 分配完了，client 会新建一条连接。而 server 则会给 client 发送一个 GOAWAY frame 强制让 client 新建一条连接。

为了更大的提高一条连接上面的 stream 并发，可以考虑调大 `SETTINGS_MAX_CONCURRENT_STREAMS`，要不然整体吞吐上不去。

这里还需要注意，虽然一条连接上面能够处理更多的请求了，但一条连接远远是不够的。一条连接通常只有一个线程来处理，所以并不能充分利用服务器多核的优势。同时，每个请求编解码还是有开销的，所以用一条连接还是会出现瓶颈。

### Priority

因为一条连接允许多个 streams 在上面发送 frame，那么在一些场景下面，我们还是希望 stream 有优先级，方便对端为不同的请求分配不同的资源。譬如对于一个 Web 站点来说，优先加载重要的资源，而对于一些不那么重要的图片啥的，则使用低的优先级。

我们还可以设置 Stream Dependencies，形成一棵 streams priority tree。假设 Stream A 是 parent，Stream B 和 C 都是它的孩子，B 的 weight 是 4，C 的 weight 是 12，假设现在 A 能分配到所有的资源，那么后面 B 能分配到的资源只有 C 的 1/3。

### Flow Control

HTTP/2 也支持流控，如果 sender 端发送数据太快，receiver 端可能因为太忙，或者压力太大，或者只想给特定的 stream 分配资源，receiver 端就可能不想处理这些数据。譬如，如果 client 给 server 请求了一个视频，但这时候用户暂停观看了，client 就可能告诉 server 别在发送数据了。

虽然 TCP 也有 flow control，但它仅仅只对一个连接有效果。HTTP/2 在一条连接上面会有多个 streams，有时候，我们仅仅只想对一些 stream 进行控制，所以 HTTP/2 单独提供了流控机制。Flow control 有如下特性：

- Flow control 是单向的。Receiver 可以选择给 stream 或者整个连接设置 window size。
- Flow control 是基于信任的。Receiver 只是会给 sender 建议它的初始连接和 stream 的 flow control window size。
- Flow control 不可能被禁止掉。当 HTTP/2 连接建立起来之后，client 和 server 会交换 SETTINGS frames，用来设置 flow control window size。
- Flow control 是 hop-by-hop，并不是 end-to-end 的，也就是我们可以用一个中间人来进行 flow control。

这里需要注意，HTTP/2 默认的 window size 是 64 KB，实际这个值太小了，在 TiKV 里面我们直接设置成 1 GB。

### HPACK

在一个 HTTP 请求里面，我们通常在 header 上面携带很多该请求的元信息，用来描述要传输的资源以及它的相关属性。在 HTTP/1.x 时代，我们采用纯文本协议，并且使用 `\r\n` 来分隔，如果我们要传输的元数据很多，就会导致 header 非常的庞大。另外，多数时候，在一条连接上面的多数请求，其实 header 差不了多少，譬如我们第一个请求可能 `GET /a.txt`，后面紧接着是 `GET /b.txt`，两个请求唯一的区别就是 URL path 不一样，但我们仍然要将其他所有的 fields 完全发一遍。

HTTP/2 为了结果这个问题，使用了 HPACK。虽然 HPACK 的 [RFC 文档](https://link.jianshu.com?t=http://httpwg.org/specs/rfc7541.html) 看起来比较恐怖，但其实原理非常的简单易懂。

HPACK 提供了一个静态和动态的 table，静态 table 定义了通用的 HTTP header fields，譬如 method，path 等。发送请求的时候，只要指定 field 在静态 table 里面的索引，双方就知道要发送的 field 是什么了。

对于动态 table，初始化为空，如果两边交互之后，发现有新的 field，就添加到动态 table 上面，这样后面的请求就可以跟静态 table 一样，只需要带上相关的 index 就可以了。

同时，为了减少数据传输的大小，使用 Huffman 进行编码。这里就不再详细说明 HPACK 和 Huffman 如何编码了。

### 总结

上面只是大概列举了一些 HTTP/2 的特性，还有一些，譬如 push，以及不同的 frame 定义等都没有提及，大家感兴趣，可以自行参考 HTTP/2 [RFC 文档](https://link.jianshu.com?t=httpwg.org/specs/rfc7540.html)。

1. 每个连接有多个stream（双向流），基本传输单元为frame，多个frame组成message（request和response）
2. frame由length、type、stream id和playload组成
3. stream用ID标识，支持了多路复用，一条连接上的stream数量可以配置，超过之后会新建新的连接
4. 同一条连接上的stream可以设置优先级
5. Receiver可以选择给 stream 或者整个连接设置 window size，默认64KB
6. header压缩原理很简单，HPACK 提供了一个静态和动态的 table，分别代表通用的和自定义的HTTP header fields。发送请求的时候，只要指定 field 在 table 里面的索引，双方就知道要发送的 field 是什么了。

## 参考

[深入了解 gRPC：协议](https://www.jianshu.com/p/48ad37e8b4ed)

