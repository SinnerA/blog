---
title: kafka基本介绍及原理分析
date: 2018-06-21
tags: kafka
---

[TOC]

## 基本概念

Kafka部分名词解释如下：

- Broker：消息中间件处理结点，一个Kafka节点就是一个broker，多个broker可以组成一个Kafka集群。
- Producer：负责发布消息到Kafka broker
- Consumer：消息消费者，向Kafka broker读取消息的客户端。
- Consumer Group：每个Consumer属于一个特定的Consumer Group（可为每个Consumer指定group name，若不指定group name则属于默认的group）。
- Topic：一类消息，例如page view日志、click日志等都可以以topic的形式存在，Kafka集群能够同时负责多个topic的分发。
- Partition：topic物理上的分组，一个topic可以分为多个partition，**每个partition是一个有序的队列**，partition内保证消息有序。
- Segment：partition物理上由多个segment组成，下面2.2和2.3有详细说明。
- Offset：每个partition都由一系列有序的、不可变的消息组成，这些消息被连续的追加到partition中。partition中的每个消息都有一个连续的序列号叫做offset，用于partition唯一标识一条消息.

### 文件存储机制

#### partition

在Kafka文件存储中，同一个topic下有多个不同partition，**每个partition为一个目录**，partiton命名规则为topic名称+有序序号，第一个partiton序号从0开始，序号最大值为partitions数量减1。

#### segment

每个partion(目录)相当于一个巨型文件被平均分配到多个大小相等segment(段)数据文件中。但每个段segment file**消息数量不一定相等**，这种特性方便old segment file快速被删除。

下图表示一个partition目录下有多个segment file：

![image](https://github.com/SinnerA/blog/blob/master/illustrations/kafka-segment-file-list.png)

- segment file组成：由2大部分组成，分别为index file和data file，此2个文件一一对应，成对出现，后缀".index"和“.log”分别表示为segment索引文件（元数据）、数据文件
- segment文件命名规则：partion全局的第一个segment从0开始，后续每个segment文件名为上一个segment文件最后一条消息的offset值。数值最大为64位long大小，19位数字字符长度，没有数字用0填充。

以上图中一对segment file文件为例，说明segment中index file和data file对应关系物理结构如下：

![image](https://github.com/SinnerA/blog/blob/master/illustrations/kafka-segment-index-correspond-data.png)

上图中索引文件存储大量元数据，数据文件存储大量消息，索引文件中元数据指向对应数据文件中message的物理偏移地址。

> 注意：在index文件当中，并不是存储了每一条消息的的索引信息，而是采用了稀疏索引，隔几个存一个索引。它减少索引文件大小，通过mmap可以直接内存操作。这种做法降低index文件元数据占用空间大小，但是查找起来增加了成本

segment data file物理结构：

![image](https://github.com/SinnerA/blog/blob/master/illustrations/kafka-segment-message-structure.png)

####通过offset查找message

1. 查找segment file

   由于segment file的命名是带有offset，并且是有序的，通过二分查找能很快定位到具体的segment文件

2. segment file中查找message

   在.log文件中顺序查找到对应offset的message

#### 读写message

写message：

- 消息直接转入page cache（物理内存）
- 由异步线程刷盘，消息从page cache刷入磁盘

读message：

- 消息直接从page cache转入socket发送出去。
- 当从page cache没有找到相应数据时，此时会产生磁盘IO，从磁盘Load消息到page cache，再从socket发出

#### 总结

- Kafka把topic中一个parition大文件分成多个小文件段，通过多个小文件段，就容易定期清除或删除已经消费完文件，减少磁盘占用。
- 通过索引信息可以快速定位message和确定response的最大大小。
- 通过index元数据全部映射到memory，可以避免segment file的IO磁盘操作。
- 通过索引文件稀疏存储，可以大幅降低index文件元数据占用空间大小。

### Consumer Group

每一个High Level Consumer实例都属于一个Consumer Group，若不指定则属于默认的Group。

- 为了实现传统MQ消息只被消费一次的语义，Kafka保证每条消息在同一个Consumer Group里只会被某一个Consumer消费。
- 与传统MQ不同的是，Kafka还允许不同Consumer Group同时消费同一条消息，这一特性可以为**消息的多元化处理提供支持**。（Kafka的设计理念之一就是同时提供离线处理和实时处理）

### Consumer Rebalance

Kafka保证同一Consumer Group中只有一个Consumer会消费某条消息，实际上，Kafka保证的是稳定状态下**每一个Consumer实例只会消费某一个或多个特定Partition的数据**，而某个Partition的数据只会被某一个特定的Consumer实例所消费。如果Consumer的数量与Partition数量相同，则正好一个Consumer消费一个Partition的数据。

触发rebalance的时机：

- 新的consumer加入
- consumer主动退出、宕机或下线

- consumer group订阅的topic的partition数量变化
- consumer调用unsubscrible取消对某topic的订阅
- coordinator挂了，集群选举出新的coordinator（0.10 特有的）

Consumer Rebalance的算法如下：

- 将目标Topic下的所有Partirtion排序，存于PTPT
- 对某Consumer Group下所有Consumer排序，存于CG于CG，第ii个Consumer记为CiCi
- N=size(PT)/size(CG)，向上取整
- 解除CiCi对原来分配的Partition的消费权（i从0开始）
- 将第i∗Ni∗N到（i+1）∗N−1（i+1）∗N−1个Partition分配给Ci

简单概括：计算size相除，解除原来的，按照公式分配新的

老版实现有脑裂和羊群问题，新版通过状态机解决了...

## 高性能



## 参考

[Kafka文件存储机制那些事](https://tech.meituan.com/kafka-fs-design-theory.html)

[kafka原理以及设计实现思想](https://kaimingwan.com/post/framworks/kafka/kafkayuan-li-yi-ji-she-ji-shi-xian-si-xiang#toc_0)

[Kafka设计解析（四）- Kafka Consumer设计解析](http://www.jasongj.com/2015/08/09/KafkaColumn4/)

[分布式消息系统Kafka简介](https://blog.csdn.net/caisini_vc/article/details/48007297)

[14个最常见的Kafka面试题及答案](https://blog.csdn.net/yjh314/article/details/77568580)
