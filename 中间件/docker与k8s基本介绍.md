---
title: docker与k8s基本介绍
date: 2018-06-26
tags: k8s docker
---

[TOC]

## docker

docker daemon就是宿主机上的一个进程，应用只是docker daemon 的一个子进程, 换句话说, 应用直接运行在宿主机内核上。

docker容器使用namespace、cgroup实现了CPU, 内存, 网络, 文件系统等资源隔离，namespace决定了能看见什么，cgroup决定了能干什么。也就是说，宿主机上有什么资源，docker就有什么资源，只不过进行了一定程度的隔离。

namespace隔离了进程、网络、IPC等资源，每个namespace都是独立的，互不干扰。

cgroup实现了对资源的分配控制，当然这需要宿主机的内核支持，Linux 2.6.24之后才支持。

### docker与虚拟机对比

![lark_01_vm_vs_docker](https://github.com/SinnerA/blog/blob/master/illustrations/docker_vs_vm.png)

1. 虚拟机运行在虚拟硬件上, 应用运行在虚拟机内核上。而 docker daemon 是宿主机上的一个进程, 应用只是 docker daemon 的一个子进程, 换句话说, 应用直接运行在宿主机内核上。
2. 虚拟机需要特殊硬件虚拟化技术支持, 因而只能运行在物理机上。docker没有硬件虚拟化, 因而可以运行在物理机、虚拟机, 甚至docker容器内(嵌套运行)。
3. 因为没有硬件虚拟化及多运行一个Linux内核的开销, 应用运行在 docker 上比虚拟机上更轻、更快。

### 核心概念

#### 镜像（imag）

docker容器的文件系统在哪里呢? 答案就是镜像。

镜像描述了 docker 容器运行的初始文件系统, 包含运行应用所需的所有依赖。即可以是一个完整的操作系统, 也可以仅包含应用所需的最小 bin/lib文件集合。我们通常是基于一个已有的镜像创建应用镜像。

一个问题：假设一个 docker 容器文件系统大小为10GB, 创建10个容器需要多少磁盘空间？ 100GB？错，还是只要 10 GB！因为 docker 镜像和容器采用分层文件系统结构，每个容器包含一层薄薄的可写层，**只读部分是共享的**。

当container运行时，这些叠加在一起的层就构成了container的运行环境（包括相应的文件，运行库等，不包括内核）。运行时，会在镜像最上面加一层writeable层，只有这一层是可写的，下面都是只读共享的。

写时复制？

#### 容器（container）

有了镜像，如何创建容器？docker不加载 kernel (不同于 vm), 也不执行init(不同于 vm 和 lxc)。“你不执行应用, 空跑一个 kernel 或者 init 有什么意义？”，docker 说。docker 容器为运行应用而生, 要创建 docker 容器, 必须指定镜像 tag 和启动应用的命令行 (或者镜像设置了默认命令行)。

容器的定义到底是什么？其实，容器就是应用程序运行的所需要资源的集合，比如镜像，CPU，内存等资源。容器类似于进程，镜像也类似程序代码，进程提供了代码的运行环境。

应用就是docker容器的主进程，其实就是docker deamon的一个子进程。主进程退出后，容器自动退出。

>  如果删除容器后，在镜像上创建的可写层文件系统也将被删除，所以日志、配置、数据等需要持久保存的文件需要挂接到外部储存。

#### 仓库（Repository）

Docker Registry提供了集中存储、分发镜像的服务。

一个 **Docker Registry** 中可以包含多个**仓库**；每个仓库可以包含多个**标签**（`Tag`）；每个标签对应一个镜像。

通常，一个仓库会包含同一个软件不同版本的镜像，而标签就常用于对应该软件的各个版本。

### 基础技术

####namespace

namespace技术能够隔离各个容器的资源，比如文件、进程、网络、IPC等。使得各个容器互相不可见，也不会互相影响。但是，它们共享内核资源，容器内的进程还是运行在内核上面。

####cgroup

Linux提供的cgroups功能，可以限制容器在运行时使用到的宿主机资源，比如内存、CPU、I/O、网络等。

可以使用默认值，也可以通过执行`docker run`命令时设置相应的选项，来具体分配容器的资源。某个容器能分配的资源取决于其他容器已占用资源。

#### AUFS（Advanced UnionFS）

UnionFS是一种为Linux设计的用于把多个目录合并mount到同一个目录下的技术。AUFS即Advanced UnionFS其实就是UnionFS的升级版。

这种分层结构，能够很方便快速地在原镜像上面重新创建一个新的镜像。

![](https://github.com/SinnerA/blog/blob/master/illustrations/docker_imag.png)

> 除了aufs，docker还支持btrfs, devicemapper和vfs

### 举例

下面是一个docker.yaml文件：

```yaml
from: a.b.c.com/library/centos:7.4
name: 
addfiles:
    # 支持使用$GOPATH环境变量，或者直接指定二进制程序的路径
    - $GOPATH/bin/test:/usr/bin
ports:
    - 5270
cmd: test
args:
    - -addr=:6789
    - -etcd-addrs=$(ETCD_ADDR)
    - -log=$(LOG_ADDR):$(LOG_PORT)
    - -namespace=$(NAMESPACE)

```

from：基于一个已有的镜像

addfile：表示将本地编译好的二进制文件 `test` 复制到镜像内`/usr/bin`下

cmd：启动应用的命令

args：启动应用的参数

ports：对外暴露的端口号

执行 `docker build` 就能创建镜像， `docker run`执行：创建容器、运行

### 好处

1. 校准化交付

   docker 将应用及其所有依赖打包到镜像内, 包括二进制文件(包括底层基础库), 静态配置文件, 环境变量等。剥离了应用对操作系统和环境的依赖， **松耦合** 。只需要拉取镜像, 启动容器即可完成应用部署, **方便** 。毫秒级创建销毁容器，从而可以实现 **快速部署、快速迁移、快速扩容缩容，一键快速回滚** (只依赖应用启动时间)。docker镜像制定了应用交付标准, 开发人员对应用及其运行环境完全可控，并**有效避免各种环境问题踩坑**。

2. 微服务编排

   单机部署多应用时，应用之间完全解耦，可以任意部署编排, **完美支持微服务编排的需求**。多个应用混布时, docker 化实现依赖隔离, 避免依赖冲突。

3. 提升资源利用率

   docker是轻量级的解决方案, 不做虚拟化, 不运行多余的 kernel 和 init 进程, 能有效提升资源利用率。

4. …？

## 参考

[Docker 容器概念](http://jm.taobao.org/2016/05/12/introduction-to-docker/)

[只要一小时，零基础入门Docker](https://zhuanlan.zhihu.com/p/23599229)

[Kubernetes的设计理念](https://jimmysong.io/kubernetes-handbook/concepts/concepts.html)

[【技术总结】一起聊聊Kubernetes](https://zhuanlan.zhihu.com/p/20612045)

[Kubernetes入门：Pod、节点、容器和集群都是什么？](https://zhuanlan.zhihu.com/p/32618563)
