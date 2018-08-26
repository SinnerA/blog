---
title: Envoy学习笔记
date: 2018-08-19
tags: 
    - service mesh
    - 微服务
---

[TOC]

Istio默认采用Envoy（读音[ˈenvɔɪ] ）作为数据面板，另外数据面板也可以采用Linkerd、nginxmesh和Conduit等。以下介绍内容来自Istio官方文档：

> Istio 使用Envoy代理的扩展版本，Envoy是以C++开发的高性能代理，用于调解服务网格中所有服务的所有入站和出站流量。
>
> Istio利用了Envoy的许多内置功能，例如动态服务发现，负载均衡，TLS termination，HTTP/2&gRPC代理，熔断器，健康检查，基于百分比流量拆分的分段推出，故障注入和丰富的metrics。
>
> Envoy实现了过滤和路由、服务发现、健康检查，提供了具有弹性的负载均衡。它在安全上支持TLS，在通信方面支持gRPC。

简单来说，Envoy提供的是服务间网络通讯的能力，它掌控了service的入口流量和出口流量，它提供了很多内置功能，如动态负载服务发现、负载均衡、TLS终止、HTTP/2 & gRPC流量代理、熔断、健康检查等功能。如下：

- 网络通信：TCP、HTTP/1.1、HTTP/2、gRPC
- 通讯相关：
  - 服务发现：从Pilot得到服务发现信息
  - 负载均衡：根据配置，一般有round robin等
  - 健康检查
  - 执行路由规则(Rule)：规则来自Polit，包括路由和目的地策略
  - 加密和认证：TLS certs来自Istio-Auth

此外, Envoy也上报数据给Mixer：

- Metrics
- Logging
- Distribution Trace：目前支持Zipkin

数据面板中的Envoy负责数据的流动和传递，其他需要控制，决策，管理的功能都是控制面板来负责。

## 基本流程

Envoy作为网络代理（SideCar），其职责就是完成请求的转发，在转发过程中会做一些额外处理。Envoy作为一个高度可定制化的进程，其定制化的载体必然是配置信息，那么我们下面就试着从Envoy的一份配置来解读其架构设计与进程流程。

关键字段说明：

Listener: 服务(进程)监听者。就是真正干活的。Envoy 会暴露一个或者多个listener监听downstream的请求。

Filter: 过滤器。在Envoy 中指的是一些“可插拔”和可组合的逻辑处理层。是**Envoy 核心逻辑处理单元**。

Route_config: 路由规则配置，即请求路由到后端那个集群(cluster)。

Cluster: 服务提供方集群。Envoy 通过服务发现定位集群成员并获取服务。具体请求到哪个集群成员是由负载均衡策略决定。通过健康检查服务来对集群成员服务状态进行检查。

![Envoy](https://img0.tuicool.com/zA7jiye.png)

基本流程：

1. 监听端口，准备获取请求
2. 拿到请求数据后对其做些微处理，抽象为Filter，例如对请求的读写是ReadFilter、WriteFilter，对TCP的处理是TcpProxyFilter，其继承自ReadFilter，各个Filter最终会组织成一个FilterChain
3. 路由到指定集群并做负载均衡获取一个目标地址，然后转发到该目标地址

## 架构

![Envoy架构](https://img0.tuicool.com/RRJj63j.png)

Envoy里面Worker的工作方式和Nginx类似：多线程 + 非阻塞 + 异步IO（Libevent），性能得到了保证。

Envoy的另一特点是支持配置信息的热更新，其功能由XDS模块完成，XDS是个统称，具体包括ADS（Aggregated Discovery Service）、SDS（[Service Discovery Service](https://hk.saowen.com/rd/aHR0cHM6Ly93d3cuZW52b3lwcm94eS5jbi9JbnRyb2R1Y3Rpb24vQXJjaGl0ZWN0dXJlb3ZlcnZpZXcvRHluYW1pY2NvbmZpZ3VyYXRpb24uaHRtbA==)）、EDS（[Endpoint Discovery Service](https://hk.saowen.com/rd/aHR0cHM6Ly93d3cuZW52b3lwcm94eS5jbi9JbnRyb2R1Y3Rpb24vQXJjaGl0ZWN0dXJlb3ZlcnZpZXcvRHluYW1pY2NvbmZpZ3VyYXRpb24uaHRtbA==)）、CDS（[Cluster Discovery Service](https://hk.saowen.com/rd/aHR0cHM6Ly93d3cuZW52b3lwcm94eS5jbi9JbnRyb2R1Y3Rpb24vQXJjaGl0ZWN0dXJlb3ZlcnZpZXcvRHluYW1pY2NvbmZpZ3VyYXRpb24uaHRtbA==)）、RDS（[Route Discovery Service](https://hk.saowen.com/rd/aHR0cHM6Ly93d3cuZW52b3lwcm94eS5jbi9JbnRyb2R1Y3Rpb24vQXJjaGl0ZWN0dXJlb3ZlcnZpZXcvRHluYW1pY2NvbmZpZ3VyYXRpb24uaHRtbA==)）、LDS（[Listener Discovery Service](https://hk.saowen.com/rd/aHR0cHM6Ly93d3cuZW52b3lwcm94eS5jbi9JbnRyb2R1Y3Rpb24vQXJjaGl0ZWN0dXJlb3ZlcnZpZXcvRHluYW1pY2NvbmZpZ3VyYXRpb24uaHRtbA==)）。XDS模块功能是向Istio的Pilot获取动态配置信息，拉取配置方式分为V1与V2版本，V1采用HTTP，V2采用gRPC。

为了保证高可用，Envoy还支持热重启，即重启时可以做到无缝衔接，启动新进程之后，通知老进程关闭。

Envoy同样也支持Lua编写的Filter，不过与Nginx一样，都是工作在HTTP层，具体实现原理都一样，不做赘述了。

## 性能

虽然ServiceMesh优势很明显，被称为服务间的通讯层，但不可否认的是ServiceMesh的到来确实对应用的性能带来了损耗，可以从两个方面看待此问题：

1. Sidecar的加入增加了业务请求的链路长度，必然会带来性能的损耗，由此延伸可知请求转发性能的高低必然会成为各个Sidecar能否最终胜出的关键点之一。

2. 控制面板采用的是集中式管理，统一负责请求的合法性校验、流控、遥测数据的收集与统计，而这要求Sidecar每转发一个请求，都需要与控制面板通讯。

   例如对应到Istio的架构中，这部分工作是由Mixer组件负责，那么可想而知这里必然会成为性能瓶颈之一，针对这个问题Istio官方给出了解决方案，即**将Mixer的大部分工作下放到Sidecar中**，对应到Envoy中就是添加一个MixerFilter来承担请求校验、流控、数据收集与统计工作，MixerFilter需要定时与Istio通讯以批量上报数据与拉取最新配置数据。这种方式在Istio之前微博的Motan、华为Mesher、唯品会的OSP中已经这么做了。

Envoy作为Sidecar其提供的核心功能可以简单总结为以下三点：

1. 对业务透明的请求拦截。
2. 对拦截请求基于一定规则做校验、认证、统计、流量调度、路由等。
3. 将请求转发出去，在ServiceMesh中所有的流量出入都要经过Sidecar，即由Sidecar承担起所有的网络通讯职责，由此可知请求转出后的下一个接收方也必然是Sidecar，那么Sidecar之间通讯协议的高效与否对ServiceMesh整体性能也会产生较大影响。

从上述三点中我们试着分析下性能优化的关键点，其中第1、3点是与业务基本无关的，属于通用型功能，而第2点的性能是与业务复杂度呈现相关性的，比如请求校验规则的多与少、遥测数据的采集精细度、数据统计的维度多样性等，因此最有可能提升Sidecar性能的点就是**对请求的拦截**与**Sidecar之间通讯协议**的高效性。

### 请求拦截

针对请求的拦截，目前常规的做法是使用iptables，在部署Sidecar时配置好iptables的拦截规则，当请求来临后iptables会从规则表中从上至下顺序查找匹配规则，如果没遇到匹配的规则，就一条一条往下执行，如果遇到匹配的规则，那就执行本规则并根据本规则的动作(accept, reject, log等)，决定下一步执行的情况。

![iptables](https://img1.tuicool.com/MF7Rv2Z.png)

了解iptables的基本流程后，不难发现其性能瓶颈主要是两点：

1. 在规则配置较多时，由于其本身顺序执行的特性，性能会下滑严重。
2. 每个request的处理都要经过内核态—>用户态—>内核态的过程，这其中会带来数据从内核态拷贝到用户态的，再拷贝到内核态的性能消耗。

既然知道了iptables的缺陷，那么优化手段不外乎从这两点下手。

Envoy社区目前正在推动官方重构其架构，目的是为了支持自定义的network socket实现，目的是引入VPP(Vector Packet Processing)，可以实现数据包在纯用户态或者内核态的处理，避免内存的来回拷贝、上下文切换。

#### DPDK

DPDK全称Intel Data Plane Development Kit，是Intel提供的数据平面开发工具集。DPDK专注于网络应用中数据包的高性能处理，它将数据包处理、内存管理、处理器调度等任务**转移到用户空间完成**，而内核仅仅负责部分控制指令的处理。

虽然Linux设计初衷是以通用性为目的的，但随着Linux在服务器市场的广泛应用，其原有的网络数据包处理方式已很难跟上人们对高性能网络数据处理能力的诉求。在这种背景下DPDK应运而生，其利用UIO技术，在Driver层直接将数据包导入到用户态进程，绕过了Linux协议栈，接下来由用户进程完成所有后续处理，再通过Driver将数据发送出去。原有内核态与用户态之间的内存拷贝采用mmap将用户内存映射到内核，如此就规避了内存拷贝、上下文切换、系统调用等问题，然后再利用大页内存、CPU亲和性、无锁队列、基于轮询的驱动模式、多核调度充分压榨机器性能，从而实现高效率的数据包处理。

总结：DPDK利用UIO技术绕过Linux协议栈，采用mmap避免内存拷贝；利用无锁队列等进一步提高性能。

VPP是the vector packet processor的简称，是一套**基于DPDK的网络帧处理解决方案**，是一个可扩展框架，提供开箱即用的交换机/路由器功能。

换句话说，Envoy可以利用DPDK的网络高性能代替现有的iptables，从而提高请求拦截的性能。

### 通信协议

Sidecar之间如何高效的通讯呢？需要一个高效的通信协议，使用最普遍的两大基本通讯协议TCP与UDP都有各自的优缺点，TCP的可靠与安全性，UDP的速度与效率。QUIC集合了两者的优点，旨在创建几乎等同于TCP的独立连接，但有着低延迟，并对类似SPDY的多路复用流协议有更好的支持。

#### QUIC

QUIC协议本身就内置TLS栈，实现自己的[传输加密层](https://hk.saowen.com/rd/aHR0cHM6Ly9kb2NzLmdvb2dsZS5jb20vZG9jdW1lbnQvZC8xZzVuSVhBSWtOX1ktN1hKVzVLNDVJYmxIZF9MMmY1TFRhRFVEd3ZaNUw2Zy9lZGl0)，而没有使用现有的TLS 1.2。同时QUIC还包含了部分HTTP/2的实现，因此QUIC的地位看起来是这样的：

![浅谈Service Mesh体系中的Envoy](https://img2.tuicool.com/2mIBBnI.png)

 QUIC协议的诞生就是为了**降低网络延迟**，开创性的使用了**UDP协议作为底层传输协议**，通过多种方式减少了网络延迟。因此带来了性能的极大提升，且具体的提升效果在Google旗下的YouTube已经验证。

Envoy社区正在推动官方重构其架构的目的之一就是为了QUIC，最终目的是希望使用QUIC作为Sidecar之间的通讯协议。

Quic 相比现在广泛应用的 http2+tcp+tls 协议有如下优势：

1. 减少了 TCP 三次握手及 TLS 握手时间。
2. 改进的拥塞控制。
3. 避免队头阻塞的多路复用。
4. 连接迁移。
5. 前向冗余纠错。

QUIC建立在UDP基础上，通过packet number严格单调递增 + offset来保证数据的顺序性和可靠性。QUIC 一个连接上的多个 stream 之间没有依赖，因此解决了HTTP2的多路复用带来的队头阻塞的问题。

## 参考

[浅谈Service Mesh体系中的Envoy](http://jm.taobao.org/2018/07/05/Mesh%E4%BD%93%E7%B3%BB%E4%B8%AD%E7%9A%84Envoy/)

[科普：QUIC 协议原理分析](https://cloud.tencent.com/developer/article/1017235)