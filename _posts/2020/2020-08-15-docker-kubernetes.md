---
title: Docker Kubernetes（11）
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2020-08-15 23:33:00 +0800
categories: [虚拟化技术, Docker 落地笔记]
tags: [Docker,Kubernetes]
math: true
mermaid: true
image:
  src: https://images.happymaya.cn/assert/docker/docker.jpeg
  width: 800
  height: 500
---

Docker 虽然在容器领域有着不可撼动的地位，然而在容器的编排领域，却有着另外一个事实标准，那就是 Kubernetes。

# Kubernetes 的历史

说起 Kubernetes，这一切得从云计算说起。

云计算是 2006 年由 Google 提起的，近些年被提及的频率也越来越高。

云计算从起初的概念，演变为现在的 AWS、阿里云等实实在在的云产品（主要是虚拟机和相关的网络、存储服务），经变得非常成熟和稳定。

正当大家以为云计算领域已经变成了以虚拟机为代表的云平台时，Docker 在 2013 年横空出世。

Docker 提出了镜像、仓库等核心概念，规范了服务的交付标准，使得复杂服务的落地变得更加简单。

之后 Docker 又定义了 OCI 标准，可以说在容器领域 Docker 已经成了事实的标准。

然而 Docker 诞生只是帮助定义了开发和交付标准，如果想要在生产环境中大批量的使用容器，还离不开的容器的编排技术。

在 2014 年 6 月 7 日，Kubernetes（Kubernetes 简称为 K8S，8 代表 ubernete 8个字母） 的第一个 commit（提交）拉开了容器编排标准定义的序幕。

Kubernetes 是舵手的意思，把 Docker 比喻成一个个集装箱，Kubernetes 正是运输这些集装箱的舵手。早期的 

Kubernetes 主要参考 Google 内部的 Borg 系统，Kubernetes 刚刚诞生时，提出了 Pod、Sidecar 等概念，这些都是 Google 内部积累的精华。经过将近一年的沉淀和积累，Kubernetes 于 2015 年 7 月 21 日对外发布了第一个正式版本 v1.0，正式走入了大众的视线。

# Kubernetes 架构

Kubernetes 采用声明式 API 来工作，所有组件的运行过程都是异步的，整个工作过程大致为：

1. 用户声明想要的状态，
2. Kubernetes 各个组件相互配合并且努力达到用户想要的状态。

Kubernetes 采用典型的主从架构，分为 Master 和 Node 两个角色。
- Mater 是 Kubernetes 集群的控制节点，负责整个集群的管理和控制功能。
- Node 为工作节点，负责业务容器的生命周期管理。

整体架构如下图：

![](https://images.happymaya.cn/assert/docker/docker-kubernetes-1.svg)
更多 Kubernetes 的了解去官方阅读 [Kubernetes 的文档](https://kubernetes.io/)

## Master 节点

Master 节点负责对集群中所有容器的调度，各种资源对象的控制，以及响应集群的所有请求。

Master 节点包含三个重要的组件：
- kube-apiserver
- kube-scheduler
- kube-controller-manager

**kube-apiserver**
负责提供 Kubernetes 的 API 服务，所有的组件都需要与 kube-apiserver 交互获取或者更新资源信息，它是 Kubernetes Master 中最前端组件。

kube-apiserver 的所有数据都存储在 etcd 中
- etcd 采用 Go 语言编写的高可用 Key-Value 数据库，由 CoreOS 开发
- etcd 虽然不是 Kubernetes 的组件，但是在 Kubernetes 中却扮演着至关重要的角色，是 Kubernetes 的数据大脑
- etcd 的稳定性直接关系着 Kubernetes 集群的稳定性，因此生产环境中 etcd 一定要部署多个实例以确保集群的高可用。

**kube-scheduler**

用于监听未被调度的 Pod，然后根据一定调度策略将 Pod 调度到合适的 Node 节点上运行。

**kube-controller-manager**

负责维护整个集群的状态和资源的管理。例如：
- 多个副本数量的保证，
- Pod 的滚动更新等。每种资源的控制器都是一个独立


> 为了保证 Kubernetes 集群的高可用，Master 组件需要部署在多个节点上，由于 Kubernetes 所有数据都存在于 etcd 中，Etcd 是基于 Raft 协议实现，因此生产环境中 Master 通常建议至少三个节点（如果你想要更高的可用性，可以使用 5 个或者 7 个节点）。
{: .prompt-warning }

## Node 节点
Node 节点是 Kubernetes 的工作节点，负责运行业务容器。

Node 节点主要包含两个组件：
- kubelet
- kube-proxy

**kubelet**
- 是在每个工作节点运行的代理
- 负责管理容器的生命周期
- 通过监听分配到自己运行的主机上的 Pod 对象，确保这些 Pod 处于运行状态，并定期检查 Pod 的运行状态，将 Pod 的运行状态更新到 Pod 对象中

**kube-proxy**
- 是在每个工作节点的网络插件，实现了 Kubernetes 的 Service 的概念
- 通过维护集群上的网络规则，实现集群内部通过负载均衡的方式访问到后端的容器


Kubernetes 的成功不仅得益于其优秀的架构设计，更加重要的是 Kubernetes 提出了很多核心的概念，这些核心概念构成了容器编排的主要模型。

# Kubernetes 核心概念

| 概念                      | 解释                                                         |
| ------------------------- | :----------------------------------------------------------- |
| 集群                      | 一组被 Kubernetes 统一管理和调度的节点，被 Kubernetes 纳管的节点可以是物理机或者虚拟机。集群其中一部分节点作为 Master 节点，负责集群状态的管理和协调，另一部分作为 Node 节点，负责执行具体的任务，实现用户服务的启停等功能 |
| 标签（Label）             | 一组键值对，每个资源对象都会拥有此字段。Kubernetes 中使用 Label 对资源进行标记，然后根据 Label 对资源进行分类和筛选 |
| 命名空间（Namespace）     | Kubernetes 中通过命名空间来实现资源的虚拟化隔离，将一组相关联的资源放到同一个命名空间内，避免不同租户的资源发生命名冲突，从逻辑上实现了多租户的资源隔离。 |
| 容器组（Pod）             | 最小调度单位，由一个或多个容器组成，一个 Pod 内的容器共享相同的网络命名空间和存储卷。Pod 是真正的业务进程的载体，在 Pod 运行前，Kubernetes 会先启动一个 Pause 容器开辟一个网络命名空间，完成网络和存储相关资源的初始化，然后再运行业务容器。 |
| 部署（Deployment）        | 一组 Pod 的抽象，通过 Deployment 控制器保障用户指定数量的容器副本正常运行，并且实现了滚动更新等高级功能，当我们需要更新业务版本时，Deployment 会按照我们指定策略自动的杀死旧版本的 Pod 并且启动新版本的 Pod。 |
| 状态副本集（StatefulSet） | 和 Deployment 类似，也是一组 Pod 的抽象，但是 StatefulSet 主要用于有状态应用的管理，StatefulSet 生成的 Pod 名称是固定且有序的，确保每个 Pod 独一无二的身份标识。 |
| 守护进程集（DaemonSet）   | 确保每个 Node 节点上运行一个 Pod，当集群有新加入的 Node 节点时，Kubernetes 会自动帮助我们在新的节点上运行一个 Pod。一般用于日志采集，节点监控等场景。 |
| 任务（Job）               | 创建一个 Pod 并且保证 Pod 的正常退出，如果 Pod 运行过程中出现了错误，Job 控制器可以创建新的 Pod，直到 Pod 执行成功或者达到指定重试次数。 |
| 服务（Service）           | Service 是一组 Pod 访问配置的抽象。由于 Pod 的地址是动态变化的，不能直接通过 Pod 的 IP 去访问某个服务，Service 通过在主机上配置一定的网络规则，实现通过一个固定的地址访问一组 Pod。 |
| 配置集（ConfigMap）       | 存放业务的配置信息，使用 Key-Value 的方式，存放于 Kubernetes 中，将配置数据和应用程序代码分开 |
| 加密字典（Secret）        | 存放业务的敏感配置信息，类似于 ConfigMap，使用 Key-Value 的方式存在于 Kubernaters 中，主要用于存放密码和证书等敏感信息 |

# Kubernetes 的安装

Kubernetes 目前支持在多种环境下安装，可以在公有云，私有云，甚至裸金属中安装 Kubernetes。

以 Linux 平台为例，通过 minikube 来演示，如何快速安装和启动一个 Kubernetes 集群
- minikube 是官方提供的一个快速搭建本地 Kubernetes 集群的工具，主要用于本地开发和调试。


> 在其他平台使用 minikube 安装 Kubernetes，请参考官网[安装教程](https://minikube.sigs.k8s.io/docs/start/)。
在使用 minikube 安装 Kubernetes 之前，先确保已经正确安装并且启动 Docker。
{: .prompt-note }


第一步，安装 minikube 和 kubectl。首先执行以下命令安装 minikube
```bash
$ curl -LO https://github.com/kubernetes/minikube/releases/download/v1.13.1/minikube-linux-amd64
$ sudo install minikube-linux-amd64 /usr/local/bin/minikube
```
Kubectl 是 Kubernetes 官方的命令行工具，实现对 Kubernetes 集群的管理和控制。
使用以下命令来安装 kubectl：
```bash
$ curl -LO https://dl.k8s.io/v1.19.2/kubernetes-client-linux-amd64.tar.gz

$ tar -xvf kubernetes-client-linux-amd64.tar.gz
kubernetes/
kubernetes/client/
kubernetes/client/bin/
kubernetes/client/bin/kubectl

$ sudo install kubernetes/client/bin/kubectl /usr/local/bin/kubectl

```

第二步，安装 Kubernetes 集群，
执行以下命令使用 minikube 安装 Kubernetes 集群：
```bash
$ minikube start
```
执行完上述命令后，minikube 会自动帮助创建并启动一个 Kubernetes 集群。

命令输出如下，当命令行输出 Done 时，代表集群已经部署完成。
![Kubernetes Success tips](https://images.happymaya.cn/assert/docker/docker-kubernetes-2.png)

第三步，检查集群状态。集群安装成功后，使用以下命令检查 Kubernetes 集群是否成功启动。
```bash
$ kubectl cluster-info

Kubernetes master is running at https://172.17.0.3:8443
KubeDNS is running at https://172.17.0.3:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

执行 kubectl cluster-info 命令后，输出 "Kubernetes master is running" 表示的集群已经成功运行。

> 172.17.0.3 为演示环境机器的 IP 地址，这个 IP 会根据你的实际 IP 地址而变化。
{: .prompt-warning }

# 创建第一个应用

集群搭建好后，使用 Kubernetes 来创建第一个应用。

这里使用 Deployment 来定义应用的部署信息，使用 Service 暴露应用到集群外部，从而使得应用可以从外部访问到。

第一步，创建 deployment.yaml 文件，并且定义启动的副本数（replicas）为 3。
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - name: hello-world
        image: wilhelmguo/nginx-hello:v1
        ports:
        - containerPort: 80
```

第二步，发布部署文件到 Kubernetes 集群中
```bash
$ kubectl create -f deployment.yaml
```

部署发布完成后，可以使用 kubectl 来查看一下 Pod 是否被成功启动。
```bash
$ kubectl get pod -o wide
NAME   READY   STATUS    RESTARTS   AGE     IP     NODE       NOMINATED NODE   READINESS GATES
hello-world-57968f9979-xbmzt   1/1     Running   0     3m19s   172.18.0.7   minikube   <none>    <none>
hello-world-57968f9979-xq5w4   1/1     Running   0     3m18s   172.18.0.5   minikube   <none>    <none>
hello-world-57968f9979-zwvgg   1/1     Running   0     4m14s   172.18.0.6   minikube   <none>    <none>
```
这里可以看到 Kubernetes 创建了 3 个 Pod 实例。

第三步，创建 service.yaml 文件，将服务暴露出去，内容如下：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-world
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: hello-world
```

然后执行如下命令在 Kubernetes 中创建 Service：

```bash
kubectl create -f service.yaml
```

服务创建完成后，Kubernetes 会随机分配一个外部访问端口，可以通过以下命令查看服务信息：
```bash
$ kubectl  get service -o wide
NAME          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE   SELECTOR
hello-world   NodePort    10.101.83.18   <none>        80:32391/TCP   12s   app=hello-world
kubernetes    ClusterIP   10.96.0.1      <none>        443/TCP        40m   <none>
```

由于集群使用 minikube 安装，要想集群中的服务可以通过外部访问，还需要执行以下命令：

```bash
$ minikube service hello-world
```
输出如下：
![image.png](https://images.happymaya.cn/assert/docker/docker-kubernetes-3.png)

可以看到 minikube 将服务暴露在了 32391 端口上，通过 http://{U-IP}:32391 可以访问到已启动的服务，如下图所示。
![image.png](https://images.happymaya.cn/assert/docker/docker-kubernetes-4.png)