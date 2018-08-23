---
title: kafka知识点索引
date: 2018-08-06
tags: 
    - kafka
---

[TOC]

## 简介

Kafka是一种分布式的，基于**发布/订阅**的消息系统。主要设计目标如下：

- 以时间复杂度为O(1)的方式提供消息持久化能力，即使对TB级以上数据也能保证常数时间复杂度的访问性能
- 高吞吐率。即使在非常廉价的商用机器上也能做到单机支持每秒100K条以上消息的传输
- 支持Kafka Server间的消息分区，及分布式消费，同时保证每个Partition内的消息顺序传输
- 同时支持离线数据处理和实时数据处理
- Scale out：支持在线水平扩展

特点：高吞吐量、超强消息堆积、持久化能力、快速的消息get、put

pull模型：push无法适应多个consumer的消费速度，可能会来不及消费。pull可以通过offset增量pull，按需消费

## 架构

[kafka原理以及设计实现思想](https://kaimingwan.com/post/framworks/kafka/kafkayuan-li-yi-ji-she-ji-shi-xian-si-xiang#toc_0)

[kafka 数据可靠性深度解读](http://www.importnew.com/25247.html)

### 基本架构

broker、topic、partition、producer、consumer、consumer group、zk

**ZK**：选举partition的leader保证高可用，以及在consumer group变化时进行rebalance

**Broker：**Kafka中使用Broker来接受Producer和Consumer的请求，并把Message持久化到本地磁盘。每个Cluster当中会选举出一个Broker来担任Controller，负责处理Partition的Leader选举，协调Partition迁移等工作。

**Consumer group**：组内只有一个consumer可以消费topic，某些情况下触发rebalance。Kafka的设计理念之一就是同时提供离线处理和实时处理，consumer group可以很好的支持。

**Partition**：每个partition都是可复制

**Producer**：根据key决定当前消息将发送给当前topic的哪个分区。如果用户给出key，则对该key进行哈希求余，得到分区号；如果用户没有提供key，则采用round-robin方式决定分区。

注意：以上组件在分布式环境下均可以是多个，支持故障转移。同时ZK仅和broker和consumer相关。值得注意的是broker的设计是无状态的，消费的状态信息依靠消费者自己维护，通过一个offset偏移量。client和server之间通信采用TCP协议。

### 设计原理

#### Topic & Partition

一个topic可以认为一个一类消息，每个topic将被分成多个partition，每个partition在存储层面是append log文件。任何发布到此partition的消息都会被追加到log文件的尾部，每条消息在文件中的位置称为offset(偏移量)，offset为一个long型的数字，它唯一标记一条消息。每条消息都被append到partition中，是顺序写磁盘，因此效率非常高（经验证，顺序写磁盘效率比随机写内存还要高，这是Kafka高吞吐率的一个很重要的保证）。

每一条消息被发送到broker中，会根据partition规则选择被存储到哪一个partition。如果partition规则设置的合理，所有消息可以均匀分布到不同的partition里，这样就实现了水平扩展。（如果一个topic对应一个文件，那这个文件所在的机器I/O将会成为这个topic的性能瓶颈，而partition解决了这个问题）。

**优点：**

1. 负载均衡：分配到多个broker上均衡负载，也可以将同一个key路由到同一个partition上
2. 灵活性：consumer可以通过offset来控制消费
3. 并发性：同一个consumer可以同时消费多个partition
4. 高可用：以partition为单位支持leader-follower
5. 有序消费：partition内有序消费

#### 文件存储机制

partition是实际物理上的概念，而topic是逻辑上的概念。partition还可以细分为segment，一个partition物理上由多个segment组成。

在Kafka文件存储中，同一个topic下有多个不同的partition，每个partiton为一个目录，partition的名称规则为：topic名称+有序序号，第一个序号从0开始计，最大的序号为partition数量减1。

````
drwxr-xr-x 2 root root 4096 Apr 10 16:10 topic_zzh_test-0
drwxr-xr-x 2 root root 4096 Apr 10 16:10 topic_zzh_test-1
drwxr-xr-x 2 root root 4096 Apr 10 16:10 topic_zzh_test-2
drwxr-xr-x 2 root root 4096 Apr 10 16:10 topic_zzh_test-3
````

segment文件由两部分组成，分别为“.index”文件和“.log”文件，分别表示为segment索引文件和数据文件。这两个文件的命令规则为：partition全局的第一个segment从0开始，后续每个segment文件名为上一个segment文件最后一条消息的offset值，数值大小为64位，20位数字字符长度，没有数字用0填充，如下：

```
00000000000000000000.index
00000000000000000000.log
00000000000000170410.index
00000000000000170410.log
00000000000000239430.index
00000000000000239430.log
```

以上面的segment文件为例，展示出segment：00000000000000170410的“.index”文件和“.log”文件的对应的关系，如下图：

![20170506094723439](http://incdn1.b0.upaiyun.com/2017/06/41c1c4d33c59043979e80a9f2c0e35ad.png)

如上图，“.index”索引文件存储大量的元数据，“.log”数据文件存储大量的消息，索引文件中的元数据指向对应数据文件中message的物理偏移地址，index文件中式稀疏存储log文件的信息。

消息都具有固定的物理结构，包括：offset（8 Bytes）、消息体的大小（4 Bytes）、crc32（4 Bytes）、magic（1 Byte）、attributes（1 Byte）、key length（4 Bytes）、key（K Bytes）、payload(N Bytes)等等字段，可以确定一条消息的大小，即读取到哪里截止。

查找文件时，因为有序可以直接二分查找比较高效。

#### 复制同步

![20170506094740547](http://incdn1.b0.upaiyun.com/2017/06/56c3e79ad2cb660168835a1bbe4436f3.png)

上图中有两个新名词：HW和LEO。这里先介绍下LEO，LogEndOffset的缩写，表示每个partition的log最后一条Message的位置。HW是HighWatermark的缩写，是指consumer能够看到的此partition的位置，后面的消息consumer无法消费。

为了提高消息的可靠性，Kafka每个topic的partition有N（>=1）个副本（replicas）。N个replicas中，其中一个replica为leader，其他都为follower, leader处理partition的所有读写请求，与此同时，follower会被动定期地去复制leader上的数据。

如下图所示，Kafka集群中有4个broker, 某topic有3个partition,且复制因子即副本个数也为3：

![20170506094800641](http://incdn1.b0.upaiyun.com/2017/06/042c0f680d328e59f163d0eca9eecab4.png)

如果leader发生故障或挂掉，一个新leader被选举并被接受客户端的消息成功写入。新leader是从ISR中选举出来的，leader负责维护和跟踪ISR。

当producer发送一条消息到broker后，leader写入消息并复制到所有follower。如果follower“落后”太多或者失效，leader将会把它从ISR中删除，因为消息复制延迟受最慢的follower限制。

#### ISR（In-Sync Replicas，同步副本集）

ISR (In-Sync Replicas)，这个是指副本同步队列。副本数对Kafka的吞吐率是有一定的影响，但极大的增强了可用性。

Kafka引入了 **ISR**的概念。ISR是`in-sync replicas`的简写。ISR的副本保持和leader的同步，当然leader本身也在ISR中。初始状态所有的副本都处于ISR中，当一个消息发送给leader的时候，leader会等待ISR中所有的副本告诉它已经接收了这个消息，如果一个副本失败了或延迟超过阈值，那么它会被移除ISR。下一条消息来的时候，leader就会将消息发送给当前的ISR中节点了。

注：ISR中包括：leader和follower。

HW俗称高水位，HighWatermark的缩写，取一个partition对应的ISR中最小的LEO作为HW，consumer最多只能消费到HW所在的位置。另外每个replica都有HW,leader和follower各自负责更新自己的HW的状态。对于leader新写入的消息，consumer不能立刻消费，leader会等待该消息被所有ISR中的replicas同步后更新HW，此时消息才能被consumer消费。这样就保证了如果leader所在的broker失效，该消息仍然可以从新选举的leader中获取。对于来自内部broKer的读取请求，没有HW的限制。

下图详细的说明了当producer生产消息至broker后，ISR以及HW和LEO的流转过程：

![20170506094838798](http://incdn1.b0.upaiyun.com/2017/06/664ca93b9de6d10977e81a31335f30de.png)

Kafka的这种使用ISR的方式则很好的**均衡了确保数据不丢失以及吞吐率**。

kafka使用Zookeeper实现leader选举。如果leader失败，controller会从ISR**随机**选出一个新的leader。leader 选举的时候可能会有数据丢失，但是committed的消息保证不会丢失。

####consumer rebalance

 理想情况下，consumer group中某个consumer固定消费某个partition，如果consumer或者partition数量有变化，需要rebalance，已达到复杂均衡的目的。

目前，每个Consumer被创建时会触发Consumer Group的Rebalance，由Zookeeper watch实现的，这种方式有以下缺点：

- **Herd effect**
  　　任何Broker或者Consumer的增减都会触发所有的Consumer的Rebalance
- **Split Brain**
  　　每个Consumer分别单独通过Zookeeper判断哪些Broker和Consumer 宕机了，那么不同Consumer[在同一时刻从Zookeeper“看”到的View就可能不一样，这是由Zookeeper的特性决定的](http://zookeeper.apache.org/doc/r3.1.2/zookeeperProgrammers.html#ch_zkGuarantees)，这就会造成不正确的Reblance尝试。
- **调整结果不可控**
  　　所有的Consumer都并不知道其它Consumer的Rebalance是否成功，这可能会导致Kafka[工作在一个不正确的状态](https://issues.apache.org/jira/browse/KAFKA-242)。

## 设计

[谈谈kafka的exactly once](https://kaimingwan.com/post/framworks/kafka/tan-tan-kafkade-exactly-once)

### 可靠性

发送（producer）可靠性：

1. acks = 0：无ack（丢消息）
2. acks = 1：producer发送数据到leader，leader写本地日志成功，返回客户端成功；此时ISR中的副本还没有来得及拉取该消息，leader就宕机了，那么此次发送的消息就会丢失。
3. acks = all：等待所有ISR中成员都复制完（只要ISR中有至少一个副本，则不会丢消息）

消费（consumer）可靠性：

1. 至多一次：读取消息->保存offset->处理消息
2. 至少一次：读取消息->处理消息->保存offset
3. 有且仅有一次：读取消息->（处理消息，保存offset）（引入两阶段提交，或者设置UUID进行消息去重）

broker发布可靠性：当发送者由于网络问题导致重发，这时候可能会产生消息重复消费。当然消费者可以自己做处理来避免重复消费，例如全局唯一ID。

### 高吞吐率

#### 顺序IO

只采用顺序IO不仅可以利用RAID技术带来很高的吞吐量，同时可以利用队列来提供常量时间的get和put。这样获取消息的效率也就是O(1)了。**这种设计方法使得消息访问速度和消息堆积的量剥离了联系**。而且操作系统对顺序IO都会进行优化，提升整体顺序IO的性能

#### page cache

当上层有写操作时，操作系统只是将数据写入PageCache，同时标记Page属性为Dirty。当读操作发生时，先从PageCache中查找，如果发生缺页才进行磁盘调度，最终返回需要的数据。实际上PageCache是把尽可能多的空闲内存都当做了磁盘缓存来使用。同时如果有其他进程申请内存，回收PageCache的代价又很小。总结：依赖OS的页缓存能大量减少IO，高效利用内存来作为缓存

#### sendfile

传统网络数据传输时，涉及到4个步骤：

1. 数据从硬盘copy到内核缓存区
2. 从内核缓冲区copy到用户空间
3. 用户空间copy到内核缓冲区（socket 缓冲区）
4. 内核缓冲区copy到网卡

总共4次copy，而使用sendfile能够直接从内核缓冲区拷贝到socket缓冲区，**避免了用户空间**的切换以及省略了2次copy，copy只发生在内核空间。

sendfile原理：使用cpu copy将内核缓冲区copy到socket 缓冲区，而其他两次copy都是DMA copy

![img](https://user-gold-cdn.xitu.io/2018/1/27/161364402193fe15?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

注：DMA除了开始时需要cpu，其他时间全程不需要cpu参与。相比中断时，需要cpu完成copy工作，大大提高了cpu的效率。

#### 批量发送

同步produce：可以指定是否批量

异步produce：总是批量

#### 序列化

Kafka消息的Key和Payload（或者说Value）的类型可自定义，只需同时提供相应的序列化器和反序列化器即可。因此用户可以通过使用快速且紧凑的序列化-反序列化方式（如Protocal Buffer）

#### 消息压缩

Producer支持End-to-End的压缩。数据在本地压缩后放到网络上传输，在Broker一般不解压(除非指定要Deep-Iteration)，直至消息被Consume之后在客户端解压。

当然用户也可以选择自己在应用层上做压缩和解压的工作(毕竟Kafka目前支持的压缩算法有限，只有GZIP和Snappy)，不过这样做反而会意外的降低效率

### 高可用

kafka通过分区的复制，来实现高可用。当leader挂了，可以重新选举新的leader来保证消费的高可用。

一种非常常用的选举leader的方式是“少数服从多数”，Kafka并不是采用这种方式。这种模式下，如果我们有2f+1个副本，那么在commit之前必须保证有f+1个replica复制完消息，同时为了保证能正确选举出新的leader，失败的副本数不能超过f个。也就是说，在生产环境下为了保证较高的容错率，必须要有大量的副本，而大量的副本又会在大数据量下导致性能的急剧下降。

实际上，leader选举的算法非常多，比如Zookeeper的Zab、Raft以及Viewstamped Replication。而Kafka所使用的leader选举算法更像是微软的PacificA算法。

Kafka在Zookeeper中为每一个partition动态的维护了一个ISR（In-sync Replication），这个ISR里的所有replica都跟上了leader，只有ISR里的成员才能有被选为leader的可能。

**在ISR中至少有一个follower时，Kafka可以确保已经commit的数据不丢失**，但如果某一个partition的所有replica都挂了，就无法保证数据不丢失了。

这种情况下有两种可行的方案：

1. 等待ISR中任意一个replica“活”过来，并且选它作为leader
2. 选择第一个“活”过来的replica（并不一定是在ISR中）作为leader

kafaka选择第2中方案。

## 实践与调优

[kafka producer性能调优](https://kaimingwan.com/post/framworks/kafka/kafka-producerxing-neng-diao-you)

### 分区

1. Partition的数量尽量提前预分配，虽然可以在后期动态增加Partition，但是会冒着可能破坏Message Key和Partition之间对应关系的风险。
2. 单机分区数不宜过多，否则会造成发端到端延迟变长。
3. 虽然跨分区不能保证全局有序消费，但是一般只要按照消息有序的KEY散列到不同的分区上，然后由多个不同的消费者并发消费。最后做排序也很简单。因为每个分区的消费都是有序的。如果一定要一开始就做到全局严格有序，可以只用一个分，当然效率会低不少。

### 生产者

1. 数据重排序、MessageSet等手段来使得消息批量顺序写入
2. 数据压缩
3. 异步发送
4. 负载均衡

### 消费者

强烈推荐使用Low level API，虽然繁琐一些，但是目前只有这个API可以对Error数据进行自定义处理，尤其是处理Broker异常或由于Unclean Shutdown导致的Corrupted Data时，否则无法Skip只能等着“坏消息”在Broker上被Rotate掉，在此期间该Replica将会一直处于不可用状态。

### 操作系统

1. 提升文件描述符的数量从而支持大量会话和连接
2. 增大socket buffer保证数据中心之间高性能数据传输

## 压测

[kafka压力测试](https://kaimingwan.com/post/framworks/kafka/kafkaya-li-ce-shi)

1. 客户端的acks策略对发送的TPS有较大的影响，TPS：acks_0 > acks_1 > ack_all;
2. 副本数越高，TPS越低
3. partition的不同会影响TPS，随着partition的个数的增长TPS会有所增长，但并不是一直成正比关系，到达一定临界值时，partition数量的增加反而会使TPS略微降低。
4. Kafka在acks=all,min.insync.replicas>=1时，具有高可靠性，所有成功返回的消息都可以落盘。

## 问题

[kafka问题收集](https://kaimingwan.com/post/framworks/kafka/kafkawen-ti-shou-ji)

[LinkedIn的Kafka分布式消息系统实践（Part 1）](http://www.infoq.com/cn/articles/kafkaesque-days-at-linkedin-part01)

## 参考

[kafka原理以及设计实现思想](https://kaimingwan.com/post/framworks/kafka/kafkayuan-li-yi-ji-she-ji-shi-xian-si-xiang#toc_0)

[kafka 数据可靠性深度解读](http://www.importnew.com/25247.html)
