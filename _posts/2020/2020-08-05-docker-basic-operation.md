---
title: Docker 容器的操作（04）
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2020-08-05 23:33:00 +0800
categories: [虚拟化技术, Docker 落地笔记]
tags: [Docker]
math: true
mermaid: true
image:
  src: https://images.happymaya.cn/assert/docker/docker.jpeg
  width: 800
  height: 500
---
# 容器（Container）是什么？

- 容器是基于镜像创建的可运行实例，并且单独存在
- 一个镜像可以创建出多个容器

> 容器的本质是进程，启动需要一个不能退出的命令作为主进程
{: .prompt-warning }

运行容器化环境时，实际是在容器内部创建该文件的读写副本， 将会添加一个容器层，该层允许修改镜像的整个副本。

# 容器的生命周期

容器的生命周期分为 5 种：

1. created：初建状态，通过 `docker create`命令生成的容器状态为初建状态
2. running：运行状态，初建状态通过`docker start`命令可以转化为运行状态
3. stopped：停止状态
   - 运行状态通过`docker stop`命令转化为停止状态
   - 停止状态的容器通过`docker start`转化为运行状态
4. paused： 暂停状态
   - 运行状态的容器也可以通过`docker pause`命令转化为暂停状态
   - 处于暂停状态的容器可以通过`docker unpause`转化为运行状态
5. deleted：删除状态【虚状态】，通过 `docker delete`删除


> 处于初建状态、运行状态、停止状态、暂停状态的容器都可以直接删除。
{: .prompt-warning }

# 容器的操作
容器的操作可以分为五个步骤

1. 创建并启动容器
2. 终止容器
3. 进入容器
4. 删除容器
5. 导入和导出容器

## 创建并启动容器

容器十分轻量，可以随时创建和删除它。使用`docker create`命令来创建容器，例如：
```bash
$ docker create -it --name=busybox busybox
Unable to find image 'busybox:latest' locally
latest: Pulling from library/busybox
61c5ed1cbdf8: Pull complete
Digest: sha256:4f47c01fa91355af2865ac10fef5bf6ec9c7f42ad2321377c21e844427972977
Status: Downloaded newer image for busybox:latest
2c2e919c2d6dad1f1712c65b3b8425ea656050bd5a0b4722f8b01526d5959ec6

$ docker ps -a | grep busybox
2c2e919c2d6d  busybox  "sh" 34 seconds ago   Created    busybox

```

如果使用`docker create`命令创建的容器处于停止状态，可以使用`docker start`命令来启动它，如下所示：

```bash
$ docker start busybox
$ docker ps
CONTAINER ID IMAGE     COMMAND  CREATED         STATUS        PORTS      NAMES
d6f3d364fad3 busybox   "sh"     16 seconds ago  Up 8 seconds             busybox
```

容器启动有两种方式：

1. 使用`docker start`命令基于已经创建好的容器直接启动 。
2. 使用`docker run`命令直接基于镜像新建一个容器并启动，相当于先执行`docker create`命令从镜像创建容器，然后再执行`docker start`命令启动容器。


当使用`docker run`创建并启动容器时，Docker 后台执行的流程为：

1. Docker 会检查本地是否存在 busybox 镜像，如果镜像不存在则从 Docker Hub 拉取 busybox 镜像；
2. 使用 busybox 镜像创建并启动一个容器；
3. 分配文件系统，并且在镜像只读层外创建一个读写层；
3. 从 Docker IP 池中分配一个 IP 给容器；
5. 执行用户的启动命令运行镜像。

上述命令中:
- -t 参数的作用是分配一个伪终端
- -i 参数则可以终端的 STDIN 打开
- 同时使用 -it 参数可以让进入交互模式。

在交互模式下，用户可以通过所创建的终端来输入命令，例如：
```bash
$ ps aux

PID   USER     TIME  COMMAND
1     root      0:00  sh
6     root      0:00  ps aux

```

可以看到容器的 1 号进程为 sh 命令，在容器内部并不能看到主机上的进程信息，因为容器内部和主机是完全隔离的。同时由于 sh 是 1 号进程，意味着如果通过 exit 退出 sh，那么容器也会退出。所以对于容器来说，**杀死容器中的主进程，则容器也会被杀死。**

## 终止容器

想停止运行中的容器，可以使用`docker stop`命令。命令格式为 `docker stop [-t|--time[=10]]`。该命令的过程是：

1. 向运行中的容器发送 SIGTERM 信号，如果容器内 1 号进程接受并能够处理 SIGTERM，则等待 1 号进程处理完毕后退出
2. 如果等待一段时间后，容器仍然没有退出，则会发送 SIGKILL 强制终止容器。

```bash
$ docker stop busybox
busybox

```

查看停止状态的容器信息，使用 `docker ps -a` 命令。
```bash
$ docker ps -a

CONTAINERID   IMAGE   COMMAND CREATED         STATUS       PORTS               NAMES
28d477d3737a  busybox "sh"    26 minutes ago  Exited (137) About a minute ago  busybox

```

重启容器，使用命令 `docker restart`
```bash
$ docker restart busybox
busybox
$ docker ps
CONTAINER ID  IMAGE     COMMAND   CREATED          STATUS       PORTS   NAMES
28d477d3737a  busybox   "sh"      32 minutes ago   Up 3 seconds         busybox

```

## 进入容器

处于运行状态的容器可以通过`docker attach`、`docker exec`、`nsente`r等多种方式进入容器。

### 使用`docker attach`命令进入容器
使用 docker attach ，进入容器，如下所示。
```bash
$ docker attach busybox

# ps aux
PID   USER     TIME  COMMAND
1     root      0:00 sh
7 root      0:00 ps aux

```

需要注意：

1. 当同时使用`docker attach`命令同时在多个终端运行时，所有的终端窗口将同步显示相同内容
2. 当某个命令行窗口的命令阻塞时，其他命令行窗口同样也无法操作。
3. 由于`docker attach`命令不够灵活，因此一般不会使用`docker attach`进入容器。


### 使用 docker exec 命令进入容器

Docker 从 1.3 版本开始，提供了更加方便地进入容器的命令`docker exec`，通过`docker exec -it CONTAINER`的方式进入到一个已经运行中的容器，如下所示。
```bash
$ docker exec -it busybox sh

$ ps aux
PID   USER     TIME  COMMAND
 1    root     0:00  sh
 7    root     0:00  sh
 12   root     0:00  ps aux
```

## 删除容器

删除容器命令的使用方式如：`docker rm [OPTIONS] CONTAINER [CONTAINER...]`

如果要删除一个停止状态的容器，可以使用docker rm命令删除。
```bash
docker rm busybox
```

如果要删除正在运行中的容器，必须添加 -f (或 --force) 参数， Docker 会发送 SIGKILL 信号强制终止正在运行的容器
```bash
docker rm -f busybox
```
## 导出导入容器

### 导出容器
使用`docker export CONTAINER`命令导出一个容器到文件，不管此时该容器是否处于运行中的状态。导出容器前，先进入容器，创建一个文件，过程如下。

1. 进入容器创建文件
```bash
$ docker exec -it busybox sh
cd /tmp && touch test
```

2. 然后执行导出命令
```bash
$ docker export busybox > busybox.tar
```
执行以上命令后会在当前文件夹下生成 busybox.tar 文件，可以将该文件拷贝到其他机器上，通过导入命令实现容器的迁移。

### 导入容器
使用`docker import`命令导入，执行完`docker import`后变为本地镜像，最后再使用`docker run`命令启动该镜像，这样就实现了容器的迁移。

导入容器的命令格式为：` docker import [OPTIONS] file|URL [REPOSITORY[:TAG]]`。

首先，使用`docker import`命令导入上一步导出的容器
```bash
$ docker import busybox.tar busybox:test
```
此时，busybox.tar 被导入成为新的镜像，镜像名称为 busybox:test 。

然后，使用`docker run`命令启动并进入容器，查看上一步创建的临时文件
```bash
$ docker run -it busybox:test sh

$ ls /tmp/
test
```
可以看到之前在 /tmp 目录下创建的 test 文件也被迁移过来了。这样就通过`docker export`和`docker import`命令配合实现了容器的迁移。

> save 和 load 操作是针对镜像的，export 和 import 是针对容器的操作
{: .prompt-warning }

> 容器的文件系统要设计成写时复制(如图 1 所示)，而不是每一个容器都单独拷贝一份镜像文件吗？
{: .prompt-warning }

