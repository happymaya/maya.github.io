---
title: Docker 的安装（01）
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2020-08-01 10:33:00 +0800
categories: [虚拟化技术, Docker 落地笔记]
tags: [Docker]
math: true
mermaid: true
image:
  src: https://images.happymaya.cn/assert/docker/docker.jpeg
  width: 800
  height: 500
---

# Docker 安装

## Docker 能做什么

Docker 的官方定义是：
> Package Software into Standardized Units for Deveoplement, Shipment and Deployment.

由官方定义可知，Docker 是用于**开发**、****发布**和**部署**的标准化单元

很多地方，将 Docker 类比集装箱。

> 大船上，各种货物要想被整齐摆放并且相互不受到影响，只需要把各种货物进行集装箱标准化。有了集装箱，就不需要专门运输水果或者化学用品的船。把各种货物通过集装箱打包，然后统一放到一艘船上运输。

- Docker 要做的就是把各种软件打包成一个集装箱（镜像），然后分发。
- 每个镜像运行在独立的容器中，因此在运行的时候相互隔离

## CentOS 下安装 Docker 

- Docker 是跨平台解决方案

- Docker 支持在当前主流的各大平台安装使用

  - 包括 Ubuntu、RHEL、CentOS、Debian 等 Linux 发行版
  - 包括 OSX、Microsoft Windows 等非 Linux 平台下
  - Linux 是 Docker 的原生支持平台，故推荐在 Linux 上使用 Docker
  - 由于使用 CentOS 较多，因此主要针对 CentOS 平台下安装和使用

  

### 操作系统要求

- CentOS 7 以上的发行版本
- 建议使用 overlay2 存储驱动程序

### 卸载已有的 Docker

- 如果已经安装过旧版本的 Docker ，执行以下命令卸载旧版本的 Docker

  ```bash
  $ sudo yum remove docker \
                    docker-clinet \
                    docker-client-latest \
                    docker-common \
                    docker-latest \
                    docker-latest-logrotate \
                    docker-logrotate \
                    docker-engine \
  ```

  

### 安装 Docker

- 首先添加 Docker 安装源，添加 Docker 安装源的命令如下：

  ```bash
  $ sudo yum-config-manager \
  	--add -repo \
  	https://download.docker.com/linux/centos/docker-ce.repo
  ```

- 然后从已经配好的源安装和更新 Docker

- 正常情况下，直接安装最新版本的 Docker 即可，因为最新版本的 Docker 有更好的稳定性和安全性，使用以下命令安装最新本的 Docker：

  ```bash
  $ sudo yum install docker-ce docker-ce-cli containerd.io
  ```

  如果安装指定版本的 Docker ，使用以下命令：

  ```bash
  $ sudo yum install docker-ce --showduplicates | sort -r
  $ sudo yum install docker-ce-<VERSION_STRING> docker-ce-cli-<VERSION_SPRING> containl'l'l'l'l'l'l'l'l'l'l'l'l'l'l'l'l'l'lerd.io
  ```

- 安装完成后，使用以下命令启动 docker

  ```bash
  $ sudo systemctl start docker
  ```

  这里有一个国际惯例，安装完成后，使用以下命令启动一个 hello world 的容器：

  ```bash
  $ sudo docker run hello-world
  ```

  运行上述命令:

  1. Dokcer 首先会检查本地是否有 `Helllo Word` 这个镜像，如果发现本地没有这个镜像，Docker 就会去 Docker Hub 官方仓库下载此镜像，
  2. 然后运行它，
  3. 最后，看到该镜像输出 "Hello from Docker" 并退出

  > 安装完成后，默认 docker 命令只能以 root 用户执行。如果想要允许普通用户执行 docker 命令，需要执行以下命令 `sudo groupadd docker && sudo gpasswd -a ${USER} docker && sudo systemctl restart docker` ，执行完命令后，退出当前命令行窗口并打开新的窗口即可  

  

  

## 容器技术原理

- 提起容器就不得不说 chroot，因为 chroot 是最早的容器雏形
- Chroot 意味着切换目录，我们可以把任何目录更改为当前进程的根目录，这与容器非常类型

### chroot

> chroot 是在 Unix 和 Linux 系统的一个操作，针对正在运作的软件进程和它的子进程，改变它外显的根目录。一个运行在这个环境下，经由 chroot 设置根目录的程序，它不能够对这个指定根目录之外的文件进行访问动作，不能读取，也不能更改它的内容。

- 通俗地说，Chroot 可以改变某进程的根目录，使这个程序不能访问目录之外的其它目录，这个跟在一个容易中是很相似的

栗子 ：

1. 首先在当前目录下创建一个 rootfs 目录：

   ```bash
   $ mkdir rootfs
   ```

2. 为了方便，使用现成的 busybox 镜像来创建一个系统，命令如下：

   ```bash
   $ cd rootfs
   $ docker export $(docker create busybox) -o busybox.tar
   $ tar -xf busybox.tar
   ```

3. 执行完上面的命令后，在 rootfs 目录下，会得到一些目录和文件，使用 ls 命令查看一下 rootfs 目录下的内容：

   ```bash
   $ ls
   bin  busybox.tar  dev  etc  home  proc  root  sys  tmp  usr  var
   ```

   可以看到 rootfs 目录下初始了一些目录

4. 下面通过一条命令` chroot /home/centos/rootfs /bin/sh`  见证 chroot 的神奇。启动一个 sh 进程，并且将 `/home/centos/roots` 作为 sh 进程的根目录。

   此时，命令窗口已经处于上述命令启动 sh 进程中，在当前 sh 命令窗口下，使用 ls 命令查看一个当前的进行，看是否真的与主机上的其它目录隔离开了

   ```bash
   / # /bin/ls /
   
   bin  busybox.tar  dev  etc  home  proc  root  sys  tmp  usr  var
   ```

   这里可以看到当前进程的根目录已经变成了主机上的 /home/centos/rootfs 目录。这样就实现了当前进程与主机的隔离。到此为止，一个目录隔离的容器就完成了。

   但是，此时还不能称之为一个容器，为什么呢？你可以在上一步（、

   使用 chroot 启动命令行窗口）执行以下命令，查看如下路由信息：

   ```bash
   /etc # /bin/ip route
   default via 172.20.1.1 dev eth0
   172.17.0.0/16 dev docker0 scope link  src 172.17.0.1
   172.20.1.0/24 dev eth0 scope link  src 172.20.1.3
   
   ```

   执行 ip route 命令后，你可以看到网络信息并没有隔离，实际上进程等信息此时也并未隔离。要想实现一个完整的容器，我们还需要 Linux 的其他三项技术： Namespace、Cgroups 和联合文件系统。

   Docker 是利用 Linux 的 **Namespace** 、**Cgroups** 和**联合文件系统**三大机制来保证实现的， 所以它的原理是使用 Namespace 做主机名、网络、PID 等资源的隔离，使用 Cgroups 对进程或者进程组做资源（例如：CPU、内存等）的限制，联合文件系统用于镜像构建和容器运行环境。

### Namespace

Namespace 是 Linux 内核的一项功能，该功能对内核资源进行隔离，使得容器中的进程都可以在单独的命名空间中运行，并且只可以访问当前容器命名空间的资源。Namespace 可以隔离进程 ID、主机名、用户 ID、文件名、网络访问和进程间通信等相关资源。

Docker 主要用到以下五种命名空间。

- pid namespace：用于隔离进程 ID。
- net namespace：隔离网络接口，在虚拟的 net namespace 内用户可以拥有自己独立的 IP、路由、端口等。
- mnt namespace：文件系统挂载点隔离。
- ipc namespace：信号量,消息队列和共享内存的隔离。
- uts namespace：主机名和域名的隔离。

### Cgroups

Cgroups 是一种 Linux 内核功能，可以限制和隔离进程的资源使用情况（CPU、内存、磁盘 I/O、网络等）。在容器的实现中，Cgroups 通常用来限制容器的 CPU 和内存等资源的使用。

### 联合文件系统

联合文件系统，又叫 UnionFS，是一种通过创建文件层进程操作的文件系统，因此，联合文件系统非常轻快。Docker 使用联合文件系统为容器提供构建层，使得容器可以实现写时复制以及镜像的分层构建和存储。常用的联合文件系统有 AUFS、Overlay 和 Devicemapper 等。

