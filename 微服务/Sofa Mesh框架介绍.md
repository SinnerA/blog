---
title: Sofa Mesh框架介绍
date: 2018-08-25
tags: 
    - service mesh
    - 微服务
---

[TOC]

## 技术选型

从现状出发，需要满足以下的落地要求：

- 性能和稳定要求：性能和稳定性都要求很高
- 部署：支持k8s，也要支持非k8s
- 体系：除了Service Mesh之外，还需要跟原来的体系，比如说Sofa，或者社区主流框架如Dubbo，Spring Cloud，相互之间打通和过渡。

数据面板：Envoy是采用C++ 14写的，跟现有的Java技术栈相差较大，长期的话，可能部分会转到Golang上去。

控制面板：Istio是做的最好的，设计理念和产品方向都很认可，但是性能和稳定性是个问题，而且对非k8s支持不理想，最后和侵入式框架互通也是一个问题，比如和采用Dubbo或者Spring Cloud的系统无法互通。

最终的策略是这样的，如下图：

![img](https://skyao.io/publication/service-mesh-explore/images/ppt-12.jpg)

这是Sofa Mesh的技术选型：左边是Istio现有的架构，Envoy/Pilot/Mixer/Auth，右边是Sofa Mesh的架构。

- 最重要的第一点：我们用Golang开发的Sidecar替换Envoy，用**Golang重写整个数据平面**。

- 第二点是我们会合并一部分的Mixer内容进到Sidecar，也就是Mixer的一部分功能会直接做进Sidecar。

  > 其实Envoy也打算Mixer的大部分工作下放到SideCar中，也就是在Envoy中添加一个MixerFilter，因为Sidecar每转发一个请求都需要跟Mixer交互，容易成为性能瓶颈。

- 第三点是我们的Pilot和Auth会做扩展和增强。

这是我们整个的技术选型方案，实际上是Istio的一个增强和扩展版本，我们会在整个Istio的大框架下去做这个事情，但是会做一些调整。

## 架构设计

### 替代Envoy

使用golang重新编写了Sidecar，说白了是做Envoy的替换，在架构上没有什么变化。

首先，实现了XDS API，兼容了兼容Istio。

其次，在协议支持上，支持标准的HTTP/1.1和HTTP/2，也就是大家常见的REST和gRPC协议。此外，增加一些特殊的协议扩展，包括Sofa协议，Dubbo协议，HSF协议。

最后，合并了部分Mixer的公道到SideCar中，这是**最大的变化**。

#### 优化Mixer

Mixer的三大功能：

1. check。也叫precondition，前置条件检查，比如说黑白名单，权限。
2. quota。比如说访问次数之类。
3. report。比如说日志，度量等。

注意：前两个功能是**同步阻塞**的，就是一定要检查通过，或者是说quota验证OK，才能往下走。如果结果没回来只能等。而Report是可以通过**异步和批量**的方式来做的。

#####下放到SideCar

在这里：决定将其中的两个部分(check和quota)合并到SideCar中，原有report部分继续保留在mixer里面。

理由：

- **每一次**请求的check都会存在**同步阻塞**和**远程调用**，这里会产生很大的开销，因此应该下放到Sidecar中完成。
- report可以作为异步和批量提交，因此可以留在Mixer中。

Istio其实也给出了Mixer Cache的方案来优化性能：把这些结果缓存在Envoy的内存里面，如果下次的检查参数是相同的，那我们可以根据这样一个缓冲的设计，拿到已经缓存的结果，就可以避免远程调用。只要缓存能够命中，那就可以避免这一次远程调用。

Mixer Cache是在Envoy这边，因为做在mixer这端是没有用的，还是要远程调用。这就会造成缓存key的量非常大，因为Mixer有多个adapter，每个adapter都有一个取值范围，因此这是一个笛卡尔乘积的量。这时要解决key过多的问题，要么就全放内存，内存爆掉；要不然限制缓存大小，就放1万个，缓存的命中率会非常低，整个缓存相当于失效了。

因此，Mixer Cache的方案并不是很理想。

Istio的初衷是把基础设施抽象出来，隔离到Mixer中，但是代价就是多了一次RPC，其实没有必要，有些所谓的基础设施就可以内置到SideCar中来做，因此进一步支撑了下放部分功能到Sidecar的理由。

##### report网络集中

report这块还留在Mixer中，没有合并到SideCar中，但是这块不是说完全没有问题，存在一个隐患：report这块可能会存在一个叫做**网络集中**的问题。

注意到，应用和Sidecar是一对一部署的，有一万个应用，就有一万个Sidecar。基础设施后端也是多机部署的。而现在的方式，流量会先打到Mixer来，Mixer也是高可用的，也是会部署多台。但是这个数量肯定不是一万这个级别，跟这个肯定会有很大的差异。这样会导致report时流量会集中到Mixer，网络吞吐量瞬间加大。

现在还不是很确定这个问题会不会导致质的影响。所以现在暂时还是把它放在这里，就是说后面会做验证，如果这个方式有问题的话，可能再合进去。但是如果没问题的话，认为分开之后架构确实会更理想一些，所以现在暂时先不合并。

目前Conduit最新版本已经把report的功能合并进来，然后check的功能，会在后续的计划中合并。国内的话，华为新浪微博他们现在通通都是选择在Sidecar里面实现功能，不走mixer。

### 增强版Pilot

需求：

- 支持跨集群，比如说现在有多个注册中心，多个注册中心之间可以相互同步信息，然后可以做跨注册中心的调用
- 支持异构，注册中心可能是不一样的东西。有些是Service Mesh的注册中心，比如Istio的，有些是Spring Cloud的注册中心，比如Consul。
- 然后终极形态，希望在两种场景都可以支持。

 以Istio的Pilot模块为基础去做扩展和增强：

- 增加Sofa Registry的Adapter，Sofa Registry是内部的服务注册中心，提供超大规模的服务注册和服务发现的解决方案。所谓超大规模，服务数以万计。
- 再加一个数据同步的模块，来实现多个服务注册中心之间的数据交换。
- 然后第三点就是希望加一个Open Service Registry API，增加服务注册，因为现在Istio的方案只有服务发现，它的服务注册是走k8s的，用的是k8s的自动服务注册。**如果想脱离k8s环境，就要提供服务注册的方案**。在服务发现和服务模型已经标准化的情况下，希望服务注册的API也能标准化。

### Edge Sidecar

当A服务发出一个请求去调用B服务的时候，由于两个集群是隔离的，网络无法相通，肯定直接调用不到的。这时local sidecar会发现，服务B不在本集群，Local Sidecar就会将请求转发给Edge Sidecar，然后由Edge Sidecar接力完成后续的工作。

简单来说，就是打通跨集群的注册中心。

## 开源策略

Sofa Mesh在非常早的时间点上就选择开源出来，其实都还不是一个完整的产品。希望能吸引整个社区，谋求合作，走开源共建的方式。

因为Golang的类库做的不是很好，没有Java沉淀的那么好。目标是希望在这个产品做完之后，能给整个社区沉淀出一套Golang的微服务基础类库。

## 参考

[大规模微服务架构下的Service Mesh探索之路](https://skyao.io/publication/service-mesh-explore/)

[剖析 | SOFARPC 框架之总体设计与扩展机制](https://mp.weixin.qq.com/s?__biz=MzUzMzU5Mjc1Nw==&mid=2247484070&idx=1&sn=206f697f0059f3ce8a05751a4a616af1&chksm=faa0ed7ccdd7646a844e783bf9e5cb042ef6bd77620f507910e4882be737172f1e2e50801d78&token=566785343&lang=zh_CN&scene=21#wechat_redirect)

[【剖析 | SOFARPC 框架】系列之链路追踪剖析](https://juejin.im/post/5b7d15d6e51d4538bf55b001)

[关于Service Mesh和Kubernets的最前沿思考|十问蚂蚁金服](https://hk.saowen.com/a/aa80b75324751669d8e9cc5890e5d2129ab8b1590a946666a72f404c8602e2f0)