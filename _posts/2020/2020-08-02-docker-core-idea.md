---
title: Docker 的核心概念（02）
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2020-08-02 19:33:00 +0800
categories: [虚拟化技术, Docker 落地笔记]
tags: [Docker]
math: true
mermaid: true
image:
  src: https://images.happymaya.cn/assert/docker/docker.jpeg
  width: 800
  height: 500
---

# 核心概念
Docker 的操作围绕镜像、容器、仓库三大核心概念。

## 镜像
-  通俗的讲，镜像是一个只读的文件和文件夹组合 
-  镜像包含容器运行时所需要的所有基础文件和配置信息 
-  镜像是容器启动的基础，是容器启动的先决条件。 

如果想要使用一个镜像，有两种方式：

1. 自己创建镜像。通常情况下，一个镜像是基于一个基础镜像构建的，可以在基础镜像上添加一些用户自定义的内容。例如基于 CentOS 镜像创作自己的业务镜像，首先安装 nginx 服务，然后部署应用程序，最后做一些自定义配置，这样一个业务镜像就好了
2. 从功能镜像仓库拉去别人制作好的镜像

## 容器
- 通俗的讲，容器是镜像的运行实体
- 容器是带有运行时需要的可写文件层（镜像是静态的只读文件）
- 容器中的进程属于运行状态（容器运行着真正的应用进程）
- 容器有**初建**、**运行**、**停止**、**暂停**和**删除**五个种状态。

容器本质上是主机上运行的一个进程.

与直接运行在主机上进程的本质区别是：容器有自己独立的命名空间隔离和资源限制（在容器内部，无法看到主机上的进程、环境变量以及网络等信息）


## 仓库


- 通俗的讲，Docker 的镜像仓库类似于代码仓库，用来存储和分发 Docker 镜像
- 镜像仓库分为**公共镜像仓库**和**私有镜像仓库**


目前，Docker Hub 是 Docker 官方的公开镜像仓库，它不仅有很多应用或者操作系统的官方镜像，还有很多组织或者个人开发的镜像供我们免费存放、下载、研究和使用。除了公开镜像仓库，也可以构建自己的私有镜像仓库。


## 镜像、容器、仓库三者之间的联系


![镜像、容器以及仓库关系图.png](https://images.happymaya.cn/assert/docker/image_container_repo_.png)

从上图 1 来看：

- 镜像是容器的基石，一个镜像可以创建多个容器
- 容器是镜像运行的实体
- 仓库就是用来存放的分发镜像。

# Docker 架构
在了解 Docker 架构之前，先的了解以下容器的发展历史
​

## 容器的发展史


1. 起初容器技术随着 Docker 的出现变得炙手可热
1. 随后所有公司都在积极拥抱容器技术，此时，出现了除 Docker 容器之外的其它容器技术，如  CoreOS 的 rkt、lxc 等。容器技术百花齐放
1. 容器技术的标准到底是什么？容器标准应该由谁来制定？ 诸公司各自为营
1. 此时还伴随着的编排技术之争（编排技术有三大主力 Docker Swarm、Kubernetes 和 Mesos，Swarm 毋庸置疑，肯定愿意把 Docker 作为唯一的容器运行时，但是 Kubernetes 和 Mesos 就不同意了，因为它们不希望调度的形式过度单一。）

在这样的背景下，最终爆发了容器大战。OCI 也正是在这样的背景下应运而生。
> OCI 全称为开放容器标准（Open Container Initiative），
> - 一个轻量级、开放的治理结构
> - OCI 组织在 Linux 基金会的大力支持下，于 2015 年 6 月份正式注册成立。基金会旨在为用户围绕工业化容器的格式和镜像运行时，制定一个开放的容器标准。
> - 目前主要有两个标准文档：容器运行时标准 （runtime spec）和容器镜像标准（image spec）。



正是由于容器的战争，才导致 Docker 不得不在战争中改变一些技术架构。最终形成了下图所示的技术架构。
![Docker 架构图 (2).png](https://images.happymaya.cn/assert/docker/dcoker_architecture.png)

从上图可以看到，Docker 整体架构采用 C/S（客户端 / 服务器）模式，主要由客户端和服务端两大部分组成。

- 客户端负责发送操作指令
- 服务端负责接收和处理指令。

客户端和服务端通信有多种方式，既可以在同一台机器上通过UNIX套接字通信，也可以通过网络连接远程通信。


## Docker 客户端

- Docker 客户端是一种泛称
- 与服务端交互的方式：
   - docker 命令（Docker 用户与 Docker 服务端交互）
   - 直接请求 REST API 的方式与 Docker 服务端交互
   - 使用各种编程语言的 SDK 与 Docker 服务端交互（前社区维护着 Go、Java、Python、PHP 等数十种语言的 SDK，足以满足你的日常需求）



## Docker 服务端

- Docker 服务端是 Docker 所有后台服务的统称。
- 其中 dockerd 是一个非常重要的后台管理进程。负责响应和处理来自 Docker 客户端的请求，然后将客户端的请求转化为 Docker 的具体操作。例如镜像、容器、网络和挂载卷等具体对象的操作和管理。
- Docker 从诞生到现在，服务端经历了多次架构重构。
   - 起初，服务端的组件是全部集成在 docker 二进制里。
   - 从 1.11 版本开始， dockerd 已经成了独立的二进制（此时的容器也不是直接由 dockerd 来启动了，而是集成了 containerd、runC 等多个组件）、
- 虽然 Docker 的架构在不停重构，但是各个模块的基本功能和定位并没有变化。
- 与一般的 C/S 架构系统一样，Docker 服务端模块负责与 Docker 客户端交互，并管理 **Docker 的容器**、**镜像**、**网络**等资源



## Docker 重要组件


以 Docker 的 18.09.2 版本为例，看下 Docker 的工具和组件。
在 Docker 安装路径下执行 ls 命令可以看到以下与 docker 有关的二进制文件。


```bash
-rwxr-xr-x 1 root root 27941976 Dec 12  2019 containerd
-rwxr-xr-x 1 root root  4964704 Dec 12  2019 containerd-shim
-rwxr-xr-x 1 root root 15678392 Dec 12  2019 ctr
-rwxr-xr-x 1 root root 50683148 Dec 12  2019 docker
-rwxr-xr-x 1 root root   764144 Dec 12  2019 docker-init
-rwxr-xr-x 1 root root  2837280 Dec 12  2019 docker-proxy
-rwxr-xr-x 1 root root 54320560 Dec 12  2019 dockerd
-rwxr-xr-x 1 root root  7522464 Dec 12  2019 runc
```
​

这里面由 Docker 两个至关重要的组件：`runc`  和 `containerd`
​


- runc
   - Docker 官方安装 OCI 容器运行时标准的实现
   - 用来运行容器的轻量级工具，是真正用来运行的容器
- containered 
   - Docker 服务端的核心组件
   - 是从 dockerd 中剥离出来的
   - 它的诞生完全遵循 OCI 遍，是容器标准化后的掺入
   - 通过 containerd-shim 启动并管理 runC
   - 真正的管理容器的生命周期

![Docker 生命周期](https://images.happymaya.cn/assert/docker/docker_life.png)

通过上图，可以看到：

- dockerd 通过 gRPC 与 containerd 通信
- 由于 dockerd 与真正的容器运行是 runC 之间有了 containerd 这一 OCI 标准，使得 dockerd 可以确保接口向下兼容
> gRPC 是一种远程服务调用。想了解更多信息可以参考[https://grpc.io](https://grpc.io)
> containerd-shim 的意思是垫片，类似于拧螺丝时夹在螺丝和螺母之间的垫片。containerd-shim 的主要作用是将 containerd 和真正的容器进程解耦，使用 containerd-shim 作为容器进程的父进程，从而实现重启 dockerd 不影响已经启动的容器进程。

# Docker 各组件之间的关系

1. 首先通过以下命令来启动一个 happymaya 容器
```bash
$ docker run -d happymaya sleep 3600
```

2. 容器启动后，通过以下命令查看 dockerd 的 PID
```bash
$ sudo ps aux | grep dockerd
root      4147  0.3  0.2 1447892 83236 ?       Ssl  Jul09 245:59 /usr/bin/dockerd
```

3. 为验上图中 Docker 各组件之间的调用关系，使用 pstree 命令查看进程父子关系,
```bash
$ sudo pstree -l -a -A 4147
dockerd
  |-containerd --config /var/run/docker/containerd/containerd.toml --log-level info
  |   |-containerd-shim -namespace moby -workdir /var/lib/docker/containerd/daemon/io.containerd.runtime.v1.linux/moby/d14d20507073e5743e607efd616571c834f1a914f903db6279b8de4b5ba3a45a -address /var/run/docker/containerd/containerd.sock -containerd-binary /usr/bin/containerd -runtime-root /var/run/docker/runtime-runc
  |   |   |-sleep 3600
```

事实上，dockerd 启动的时候， containerd 就随之启动了，dockerd 与 containerd 一直存在。

当执行 docker run 命令（通过 happymaya 镜像创建并启动容器）时，containerd 会创建 containerd-shim 【containerd-shim 是真正容器的进程的父进程，这么做为了不让真正的容器进程作为 containerd 的子进程，从而可以实现重启 containerd 而不影响已经运行的容器】充当 “垫片”进程，然后启动容器的真正进程 sleep 3600 。这个过程和架构图是完全一致的


> 为什么 Docker 公司要把containerd拆分并捐献给社区吗？containerd 捐赠拆分， 
在一定程度上让开发者更容易的去接触到一些特性，使得‘标准’这个概念也更加深刻吧
{: .prompt-note }

