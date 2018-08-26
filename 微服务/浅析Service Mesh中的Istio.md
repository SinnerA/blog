---
title: 浅析Service Mesh中的Istio
date: 2018-08-18
tags: 
    - service mesh
    - 微服务
---

[TOC]

## 介绍

Istio是由Google， IBM和Lyft共同开发的一个Service Mesh框架，为微服务开发和维护提供可靠的基础。Istio是2017年5月才发布的0.1.0 版本的新鲜出炉的开源项目，2018年8月1日发布了1.0.0版本，很多地方还不完善。

> Istio提供一种简单的方式来建立已部署的服务的网络，具备负载均衡，服务到服务认证，监控等等功能，而**不需要改动任何服务代码。**

简单的说，有了Istio，你的服务就不再需要任何微服务开发框架（典型如Spring Cloud，Dubbo），也不再需要自己动手实现各种复杂的服务治理的功能（很多是Spring Cloud和Dubbo也不能提供的，需要自己动手）。只要服务的客户端和服务器可以进行简单的直接网络访问，就可以通过将网络层委托给Istio，从而获得一系列的完备功能。

Istio取代了传统的微服务开发框架和服务治理，开发人员只需要专注于业务开发。

### 功能

Istio首先是一个Service Mesh，但是Istio又不仅仅是Service Mesh：在Linkerd，Envoy这样的典型Service Mesh之上，Istio提供了一个完整的解决方案，为整个Service Mesh提供行为洞察和操作控制，以满足微服务应用程序的多样化需求。

Istio在服务网络中统一提供了许多关键功能(以下内容来自官方文档)：

- 流量管理：控制服务之间的流量和API调用的流向，使得调用更可靠，并使网络在恶劣情况下更加健壮。
- 可观察性：了解服务之间的依赖关系，以及它们之间流量的本质和流向，从而提供快速识别问题的能力。
- 策略执行：将组织策略应用于服务之间的互动，确保访问策略得以执行，资源在消费者之间良好分配。策略的更改是通过配置网格而不是修改应用程序代码。
- 服务身份和安全：为网格中的服务提供可验证身份，并提供保护服务流量的能力，使其可以在不同可信度的网络上流转。

除此之外，Istio针对可扩展性进行了设计，以满足不同的部署需要：

- 平台支持：Istio旨在在各种环境中运行，包括跨云， 预置，Kubernetes，Mesos等。最初专注于Kubernetes，但很快将支持其他环境。
- 集成和定制：策略执行组件可以扩展和定制，以便与现有的ACL，日志，监控，配额，审核等解决方案集成。

这些功能极大的减少了应用程序代码，底层平台和策略之间的耦合，使微服务更容易实现。

## 架构

Istio逻辑上分为数据面板和控制面板。

- 数据面板：由一组智能代理（默认为Envoy）组成，代理部署为SideCar，调解和控制微服务之间所有的网络通信。
- 控制面板：负责管理和配置代理来路由流量，以及在运行时执行策略。包括Pilot、Mixer和Istio-Auth。

在Istio的架构中，这两个部分的分工非常的清晰，控制面板： Mixer，Pilot和Auth这三个模块都是Go语言开发，代码托管在Github上，三个仓库分别是Istio/mixer，Istio/pilot，Istio/auth。而Envoy来自Lyft，编程语言是C++ 11，代码也托管在Github但不是Istio下。从团队分工看，Google和IBM关注于控制面板中的Mixer，Pilot和Auth，而Lyft继续专注于Envoy。

Istio的这个架构设计，将底层Service Mesh的具体实现，和Istio核心的控制面板拆分开。从而使得Istio可以借助成熟的Envoy快速推出产品，未来如果有更好的Service Mesh方案也方便集成。

基于Scala的Linkerd的功能和定位和Envoy非常相似，而且就在今年上半年成功进入CNCF。Nginx也推出了自己的服务网格产品Nginmesh。它们都支持和Istio集成，意味着都能取代Envoy作为Istio的数据面板。

下图为Istio的架构详细分解图：

![img](https://skyao.io/publication/service-mesh-next-generation-microservice/images/ppt-31.JPG)

这是宏观视图，可以更形象的展示Istio两个面板的功能和合作：

![Markdown](https://segmentfault.com/img/remote/1460000011316757)

### Envoy

Istio默认采用Envoy（读音[ˈenvɔɪ] ）作为数据面板，另外数据面板也可以采用Linkerd和nginxmesh等。以下介绍内容来自Istio官方文档：

> Istio 使用Envoy代理的扩展版本，Envoy是以C++开发的高性能代理，用于调解服务网格中所有服务的所有入站和出站流量。
>
> Istio利用了Envoy的许多内置功能，例如动态服务发现，负载均衡，TLS termination，HTTP/2&gRPC代理，熔断器，健康检查，基于百分比流量拆分的分段推出，故障注入和丰富的metrics。
>
> Envoy实现了过滤和路由、服务发现、健康检查，提供了具有弹性的负载均衡。它在安全上支持TLS，在通信方面支持gRPC。

简单来说，Envoy提供的是服务间网络通讯的能力，它掌控了service的入口流量和出口流量，它提供了很多内置功能，如动态负载服务发现、负载均衡、TLS终止、HTTP/2 & gRPC流量代理、熔断、健康检查等功能。如下：

- 网络通信：TCP、HTTP/1.1、HTTP/2、gRPC

- 通讯相关：

  - 服务发现：从Pilot得到服务发现信息

  - 负载均衡
  - 健康检查
  - 执行路由规则(Rule)：规则来自Polit，包括路由和目的地策略
  - 加密和认证：TLS certs来自Istio-Auth

此外, Envoy也上报数据给Mixer：

- Metrics
- Logging
- Distribution Trace：目前支持Zipkin

数据面板中的Envoy负责数据的流动和传递，其他需要控制，决策，管理的功能都是控制面板来负责。

### Pilot

#### 功能

官方文档中对Pilot（读音['paɪlət]）的功能描述：

> Pilot负责**收集和验证配置**并将其传播到各种Istio组件。它从Mixer和Envoy中抽取环境特定的实现细节，为他们提供独立于底层平台的用户服务的抽象表示。此外，流量管理规则（即通用4层规则和7层HTTP/gRPC路由规则）可以在运行时通过Pilot进行编程。
>
> 每个Envoy实例根据其从Pilot获得的信息以及其负载均衡池中的其他实例的定期健康检查来维护负载均衡信息，从而允许其在目标实例之间智能分配流量，同时遵循其指定的路由规则。
>
> Pilot负责在Istio服务网格中部署的Envoy实例的生命周期。

Pilot提供以下重要功能：

- 对Envoy的生命周期进行管理
- 智能路由（如A/B测试、金丝雀部署）
- 流量管理（超时、重试、熔断）

Pliot接收用户指定的高级路由规则配置，转换成Envoy的配置，使这些规则生效。

#### 架构

![Markdown](https://segmentfault.com/img/remote/1460000011316760)

1. Envoy API负责和Envoy的通讯，主要是发送服务发现信息和流量控制规则给Envoy
2. Envoy提供服务发现，负载均衡池和路由表的动态更新的API。这些API将Istio和Envoy的实现解耦。(另外，也使得Linkerd之类的其他服务网络实现得以平滑接管Envoy)
3. Polit定了一个抽象模型Abstract Model，以从特定平台细节中解耦，为跨平台提供基础。
4. Platform Adapter则是这个抽象模型的现实实现版本, 用于对接外部的不同平台。目前主要支持K8S。
5. 最后是 Rules API，提供接口给外部调用以管理Pilot，包括命令行工具Istioctl以及未来可能出现的第三方管理界面

Pilot架构中, 最重要的是Abstract Model和Platform Adapter。

- Abstract Model：为了实现istio对不同服务注册中心的支持，如Kubernetes，consul、Cloud Foundry等，pilot-discovery需要对以上两个输入来源的数据有一个统一的存储格式，也就是图中的Abstract Model。。
- Platform Adapter：这里有各种平台的实现，目前主要是Kubernetes，另外最新的0.2版本的代码中出现了Consul和Eureka。

#### 原理剖析

pilot分为pilot-agent和pilot-discovery。

##### pilot-agent

envoy不直接和k8s，Consul，Eureka等这些平台交互，所以需要其他服务与它们对接，管理配置，pilot-agent就是这样的服务，envoy只负责流量的收发。

pilot-agent负责的工作包括：

1. 生成envoy的配置
2. 管理envoy的生命周期：
   - 启动envoy
   - 监控并管理envoy的运行状况，比如envoy出错时pilot-agent负责重启envoy，或者envoy配置变更后reload envoy

##### pilot-discovery

pilot-discovery扮演服务注册中心、istio控制平面到Envoy之间的桥梁作用。其实上面的pilot的架构图就是pilot-discovery架构图，从图中可以看出，pilot-discovery的主要功能包括：

1. 监控服务注册中心（如Kubernetes）的服务注册情况。在Kubernetes环境下，会监控`service`、`endpoint`、`pod`、`node`等资源信息
2. 监控istio控制面信息变化，在Kubernetes环境下，会监控包括`RouteRule`、`VirtualService`、`Gateway`、`EgressRule`、`ServiceEntry`等以Kubernetes CRD形式存在的istio控制面配置信息。
3. 将上述两类信息合并组合为Envoy可以理解的（即遵循Envoy data plane api的）配置信息，并将这些信息以gRPC协议提供给Envoy

**服务注册：**当前istio能够对接的服务注册中心类型包括k8s和Consul等。以k8s为例，pilot-discovery通过监控k8s的服务发现，把集群信息变更信息提供给envoy。

> k8s的服务发现的两种实现方式：环境变量和DNS

**istio控制面板：**istio的用户可以通过istioctl创建`route rule`、`virtualservice`等实现对服务网络中的流量管理等配置建。

**envoy控制面：**pilot-discovery创建Envoy xds server对外提供gRPC协议discovery服务。所谓的`xds`代表Envoy v2 data plane api中的`eds`、 `cds`、 `rds`、 `lds`、 `hds`、 `ads`、 `kds`等api。这里的x是一个代词，类似云计算里的XaaS可以指代IaaS、PaaS、SaaS等。

pilot-discovery在初始化discovery service（`xds`服务）的过程中（`initDiscoveryService`方法），创建了discovery server对象，由它负责启动了两个gRPC服务：`eds`（endpoint discovery service）和`ads`（aggregated discovery service）。

discovery server的主要逻辑，就是在与每一个Envoy建立一个双向streaming的gRPC连接（Bidirectional streaming RPC）之后：

1. 启动一个协程从gRPC连接中读取来自Envoy的请求
2. 在原来的协程中处理来自各gRPC连接的请求。

### Mixer

Mixer（读音['mɪksə(r)]）负责在服务网格上执行访问控制和使用策略，并收集Envoy代理和其他服务的监控数据。通过对envoy上报的attributes进行处理，结合内部的adapters实现日志记录、监控指标采集、配额管理等功能。

Mixer 提供三个核心功能：

- 前提条件检查（Precondition Checking）：允许服务在响应来自服务消费者的传入请求之前验证一些前提条件。前提条件包括认证，黑白名单，ACL检查等等。
- 配额管理（Quota Management）：当多个请求发生资源竞争时，通过配额管理机制可以实现对资源的有效管理。典型例子如**限速**。
- 遥测报告上报（Telemetry Reporting）：该服务处理完请求后，通过Envoy向Mixer上报日志、监控等数据。

Mixer是高度模块化和可扩展的组件。其中一个关键功能是抽象出不同策略和遥测后端系统的细节，允许Envoy和基于Istio的服务与这些后端无关，从而保持他们的可移植。

#### 工作方式

##### Attribute（属性）

Istio用[attributes](https://link.zhihu.com/?target=https%3A//preliminary.istio.io/docs/concepts/policy-and-control/attributes.html)来控制服务在Service Mesh中运行时行为。attributes是有名称和类型的元数据，用来描述入口和出口流量和流量产生时的环境。attributes携带了一些具体信息，比如：API请求状态码、请求响应时间、TCP连接的原始地址等。例如：

```json
request.path: xyz/abc
request.size: 234
request.time: 12:34:56.789 04/17/2017
source.ip: 192.168.0.1
destination.service: example
```

请求处理过程中，attributes由Envoy收集并发送给Mixer，Mixer中根据运维人员设置的配置来处理attributes。基于这些attributes，Mixer会产生对各种基础设施后端的调用。 

![Markdown](https://segmentfault.com/img/remote/1460000011316769)

##### RefrencedAttributes（被引用的属性）

refrencedAttributes是Mixer Check时进行条件匹配后被使用的属性的集合。Envoy向Mixer发送的Check请求中传递的是属性的全集，refrencedAttributes只是该全集中被应用的一个子集。

为防止每次请求时Envoy都向Mixer中发送Check请求，Mixer中建立了一套复杂的缓存机制，使得大部分请求不需要向Mixer发送Check请求。

##### Adapter（适配器）

Mixer是一个高度模块化、可扩展组件，内部提供了多个适配器([adapter](https://link.zhihu.com/?target=https%3A//link.jianshu.com/%3Ft%3Dhttps%253A%252F%252Fgithub.com%252Fistio%252Fistio%252Ftree%252Fmaster%252Fmixer%252Fadapter))。
Envoy提供request级别的属性（[attributes](https://link.zhihu.com/?target=https%3A//link.jianshu.com/%3Ft%3Dhttps%253A%252F%252Fistio.io%252Fdocs%252Fconcepts%252Fpolicy-and-control%252Fattributes.html)）数据。

adapters基于这些attributes来实现日志记录、监控指标采集展示、配额管理、ACL检查等功能。Istio内置的部分adapters举例如下：

- [circonus](https://link.zhihu.com/?target=https%3A//github.com/istio/istio/tree/master/mixer/adapter/circonus)：一个微服务监控分析平台。
- [cloudwatch](https://link.zhihu.com/?target=https%3A//github.com/istio/istio/tree/master/mixer/adapter/cloudwatch)：一个针对AWS云资源监控的工具。
- [fluentd](https://link.zhihu.com/?target=https%3A//github.com/istio/istio/tree/master/mixer/adapter/fluentd)：一款开源的日志采集工具。
- [prometheus](https://link.zhihu.com/?target=https%3A//github.com/istio/istio/tree/master/mixer/adapter/prometheus)：一款开源的时序数据库，非常适合用来存储监控指标数据。
- [statsd](https://link.zhihu.com/?target=https%3A//github.com/istio/istio/tree/master/mixer/adapter/statsd)：一款采集汇总应用指标的工具。
- [stdio](https://link.zhihu.com/?target=https%3A//github.com/istio/istio/tree/master/mixer/adapter/stdio)：stdio适配器使Istio能将日志和metrics输出到本地，结合内置的ES、Grafana就可以查看相应的日志或指标。

##### Check和Report

对于一个网络请求，Mixer通常会调用两个rpc：Check和Report。

```go
service Mixer {
  // Check 基于活动配置和Envoy提供的attributes，执行前置条件检查和配额管理。
  rpc Check(CheckRequest) returns (CheckResponse) {}

  // Reports 基于活动配置和Envoy提供的attribues上报遥测数据（如logs和metrics）。
  rpc Report(ReportRequest) returns (ReportResponse) {}
}
```

![img](https://pic1.zhimg.com/80/v2-f04699dfbc3160225d1395277d325c5c_hd.jpg)

#### 工作流程

- Mixer server启动。

- - 初始化adapter worker线程池（进比出快可能会导致阻塞，因为协程处理不过来，可以多开协程）
  - 初始化Mixer模板仓库。
  - 初始化adapter builder表。
  - 初始化runtime实例。
  - 注册并启动gRPC server。

- 某一服务外部请求被envoy拦截，envoy根据请求生成指定的attributes，attributes作为参数之一向Mixer发起Check rpc请求。

- Mixer进行前置条件检查和配额检查，调用相应的adapter做处理，并返回相应结果。

- Envoy分析结果，决定是否执行请求或拒绝请求。若可以执行请求则执行请求。请求完成后再向Mixer gRPC服务发起Report rpc请求，上报遥测数据。

- Mixer后端的adapter基于遥测数据做进一步处理。

### Istio-Auth

Istio-Auth提供强大的服务到服务和终端用户认证，使用交互TLS，内置身份和凭据管理。它可在不修改业务代码的情况下，对Service Mesh中的未加密流量进行加密。

Istio的未来版本将增加细粒度的访问控制和审计，以使用各种访问控制机制（包括基于属性和角色的访问控制以及授权钩子，如RBAC）来控制和监视访问您的服务，API或资源的人员。

#### 架构

Istio-Auth架构，其中包括三个组件：身份，密钥管理和通信安全。

在这个例子中, 服务A以服务帐户“foo”运行, 服务B以服务帐户“bar”运行, 他们之间的通讯原来是没有加密的. 但是Istio在不修改代码的情况, 依托Envoy形成的服务网格, 直接在客户端Envoy和服务器端Envoy之间进行通讯加密。

目前在Kubernetes上运行的 Istio，使用Kubernetes service account服务帐户来识别运行该服务的人员。

![Markdown](https://segmentfault.com/img/remote/1460000011316773)

#### 未来

Istio-Auth在目前的Istio版本(0.1.6和即将发布的0.2)中，功能还不是很全，未来则规划有非常多的特性：

- 细粒度授权和审核
- 可插拔密钥管理组件

## 展望

Istio到目前为止（2018年8月）刚刚发布1.0.0版本，因此很多地方还不太完善。还存在一些问题：

- 只支持k8s，而且要求k8s 1.7.4+，因为使用到k8s的CustomResourceDefinitions
- 很多功能尚未完成
- 性能较低

## 参考

[万字解读:Service Mesh服务网格新生代--Istio](https://segmentfault.com/a/1190000011316744)

[从架构到组件，深挖istio如何连接、管理和保护微服务2.0？](https://www.kubernetes.org.cn/3575.html)

[Service Mesh深度学习系列（一）| istio源码分析之pilot-agent模块分析](https://github.com/SecretQY/Service-Mesh/blob/master/istio%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%B9%8Bpilot-agent%E5%88%86%E6%9E%90%20.md)

[Service Mesh深度学习系列（二）| istio源码分析之pilot-discovery模块分析](https://github.com/SecretQY/Service-Mesh/blob/master/istio%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%B9%8Bpilot-discovery%E5%88%86%E6%9E%902.md)

[Service Mesh深度学习系列（三）| istio源码分析之pilot-discovery模块分析（中）](https://github.com/SecretQY/Service-Mesh/blob/master/istio%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%B9%8Bpilot-discovery%E5%88%86%E6%9E%90(%E4%B8%AD).md)

[istio源码分析——pilot-agent如何管理envoy生命周期](https://segmentfault.com/a/1190000015171622)

[istio源码解析系列(三)-Mixer工作流程浅析](https://zhuanlan.zhihu.com/p/36966524)