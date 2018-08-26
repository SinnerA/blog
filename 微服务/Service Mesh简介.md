---
title: Service Mesh简介
date: 2018-08-06
tags: 
    - service mesh
    - 微服务
---

[TOC]

## 什么是Service Mesh

这个词最早使用由开发Linkerd的Buoyant公司提出，并在内部使用。2016年9月29日第一次公开使用这个术语。2017年的时候随着Linkerd的传入，Service Mesh进入国内技术社区的视野。最早翻译为“服务啮合层”，这个词比较拗口。用了几个月之后改成了服务网格。

Buoyant给出的Service Mesh的定义：

> service mesh是一个基础设施层，用于**处理服务间通信**。云原生应用有着复杂的服务拓扑，service mesh负责在这些拓扑中实现请求的可靠传递。在实践中，服务网格通常实现为一组**轻量级网络代理**，它们与应用程序部署在一起，而**对应用程序透明**。

### 部署模型

**单个服务**

Service Mesh的部署模型，先看单个的，对于一个简单请求，作为请求发起者的客户端应用实例，会首先用简单方式将请求发送到本地的Service Mesh实例。这是两个独立进程，他们之间是远程调用。

![img](https://skyao.io/publication/service-mesh-next-generation-microservice/images/ppt-7.JPG)

这种形式也叫做Sidecar（边车或挎斗），就是在原有的客户端和服务端之间加多了一个**代理**。

**多个服务**

多个服务调用的情况，在这个图上我们可以看到Service Mesh在所有的服务的下面，这一层被称之为**服务间通讯专用基础设施层**。Service Mesh会接管整个网络，把所有的请求在服务之间做转发。在这种情况下，我们会看到上面的服务不再负责传递请求的具体逻辑，只负责完成业务处理。服务间通讯的环节就从应用里面剥离出来，呈现出一个抽象层。

![img](https://skyao.io/publication/service-mesh-next-generation-microservice/images/ppt-9.JPG)

**大量服务**

如果有大量的服务，就会表现出来网格。图中左边绿色方格是应用，右边蓝色的方框是Service Mesh，蓝色之间的线条是表示服务之间的调用关系。Sidecar之间的连接就会形成一个网络，这个就是服务网格名字的由来。这个时候代理体现出来的就和前面的sidecar不一样了，形成网状。

![img](https://skyao.io/publication/service-mesh-next-generation-microservice/images/ppt-10.JPG)

**四个关键**

- 抽象：抽象出一个基础设施层，在应用之外。
- 功能：实现请求的可靠传递。
- 部署：体现为轻量级的网络代理。
- 透明：对应用程序透明。

Service Mesh定义当中一个非常重要的关键点，和Sidecar不相同的地方：**不再将代理视为单独的组件，而是强调由这些代理连接而形成的网络**。在Service Mesh里面非常强调代理连接组成的网络，而不像sidecar那样看待个体。

## 演进历程

虽然Service Mesh这个词汇直到2016年9才有，但是它表述的东西很早以前就出现了。

**远古时代**

第一代网络计算机系统，最早的时候开发人员需要在自己的代码里处理网络通讯的细节问题，比如说数据包顺序、流量控制等等，导致网络逻辑和业务逻辑混杂在一起。

接下来出现了TCP/IP技术，解决了流量控制问题，业务代码不需要关心网络通信的细节，全交给TCP/IP就行。

**微服务时代**

我们在做微服务的时候要处理一系列的比较基础的事情，比如说常见的服务注册、服务发现，在得到服务器实例之后做负载均衡，为了保护服务器要熔断/重试等等。本质问题跟“远古时代”一样，应用程序里面加上了大量的非功能性的代码。

为了简化开发，我们开始使用各种类库，这样会带来几个问题：

1. 内容比较多，需要学习成本
2. 功能不够，需要自己定制开发
3. 跨语言，需要针对语言封装类库
4. 兼容性，版本升级需要维护兼容性

**代理（SideCar）**

既然我们可以把网络访问的技术栈向下移为TCP，我们是可以也有类似的，把微服务的技术栈向下移？

有人使用代理的方案，常见的nginx，haproxy，apache等代理。这些代码和微服务关系不大，但是提供了一个思路：在服务器端和客户端之间插入了一个东西完成功能，避免两者直接通讯。

其实这就是Sidecar，但是这个时期的Sidecar是有局限性的，都是**为特定的基础设施而设计**，通常是和当时开发Sidecar的公司自己的基础设施和框架直接绑定的，在原有体系上搭出来的。这里面会有很多限制，一个最大的麻烦是无法通用。

![img](https://res.infoq.com/articles/pattern-service-mesh/zh/resources/158-1509284841154.png)

**通用型Service Mesh**

在这样的模型里，每个服务都会有一个SideCar与之配对，服务间通信都是通过SideCar进行的。

后来演化出了通用型的Service Mesh：Linkerd、Envoy、nginmesh，Linkerd和Envoy都加入了CNCF。

Service Mesh和Sidecar的差异：

1. 首先Service Mesh不再视为单独的组件，而是强调连接形成的网络
2. 第二Service Mesh是一个通用组件
3. Sidecar是可选的，容许直连（原生语言可以选择直连），但是Service Mesh会要求完全掌控所有流量，也就是所有的请求都必须通过service mesh。

**Istio**

后来演化过程中，Service Mesh又分为数据面板和控制面板，传统负责数据收发的是数据面板，而为数据面板提供策略和控制的是控制面板。

**数据面板**：接收系统中每一个包或请求。提供服务发现，健康检查，路由，负载均衡，保证服务认证/授权，以及监控的功能。上面提到的Linkerd、Envoy等都是数据面板。

**控制面板**：为正在网格中运行的数据面板提供策略和配置。不接受系统中的任何包或请求。控制面板将所有的数据面板转变为一个分布式系统。Istio是控制面板的代表。

Istio来自于谷歌、IBM和Lyft，是Service Mesh的集大成者。**可以通过控制面板控制整个系统**，从使用一系列独立运行的代理转向使用集中式的控制面板，这是Istio带来的最大的革新。

Istio的架构，主要分为两大块：

- 数据面板：就是传统的service mesh，目前标配是Envoy，Linkerd和nginmesh也在和istio做集成，也就是说可以替代Envoy做数据面板。
- 控制面板：这是Istio的主要革新，主要分成Pilot、Mixer、Istio-Auth

![img](https://skyao.io/publication/service-mesh-next-generation-microservice/images/ppt-31.JPG)

**总结**

service mesh这是一步一步过来的：从原始的代理，到限制很多的Sidecar，再到通用性的Service Mesh，然后到**加强管理功能**的Istio，在未来成为下一代的微服务。

## 为何需要Service Mesh

Service Mesh解决了微服务时代的几个问题：

1. “内容比较多，需要学习成本”：交给service mesh，业务不关心各种非业务功能
2. “功能不够，需要自己定制开发”：让service mesh去丰富功能
3. “跨语言，需要针对语言封装类库”：一套service mesh满足所有语言
4. “兼容性，版本升级需要维护兼容性”：service mesh可以单独升级，应用程序不需要升级

**带来的变革**：

1. 微服务开发的门槛降低
2. 团队专注业务，提高了效率
3. 加强运维监控能力
4. 多语言使得团队语言选择不受限

## 参考

[Service Mesh：下一代微服务](https://skyao.io/publication/service-mesh-next-generation-microservice/)

[模式之服务网格](http://www.infoq.com/cn/articles/pattern-service-mesh)