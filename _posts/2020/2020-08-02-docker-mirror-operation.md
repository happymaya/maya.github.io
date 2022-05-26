---
title: Docker 镜像的操作（03）
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2020-08-02 21:33:00 +0800
categories: [虚拟化技术, Docker 落地笔记]
tags: [Docker]
math: true
mermaid: true
image:
  src: https://images.happymaya.cn/assert/docker/docker.jpeg
  width: 800
  height: 500
---

# 镜像
- 一个只读的 Docker 容器模板，包含启动容器所需要的所有文件系统结构和内容
- 一个特殊的文件系统，它提供了容器运行时所需的程序、软件库、资源、配置等静态数据。即**镜像不包含任何动态数据，镜像内容在构建后不会被改变**。

# 镜像的操作
镜像的操作分为：

1. 拉取镜像，使用docker pull命令拉取远程仓库的镜像到本地
2. 重命名镜像，使用docker tag命令“重命名”镜像
3. 查看镜像，使用docker image ls或docker images命令查看本地已经存在的镜像
4. 删除镜像，使用docker rmi命令删除无用镜像 
5. 构建镜像，构建镜像有两种方式。第一种方式是使用docker build命令基于 Dockerfile 构建镜像，也是我比较推荐的镜像构建方式；第二种方式是使用docker commit命令基于已经运行的容器提交为镜像


## 拉取镜像
命令：`docker pull`，
命令格式为： `docker pull [Registry]/[Repository]/[Image]:[Tag]`。

- Registry：注册服务器（默认会从 `docker.io` 拉取镜像，如果有自己的镜像仓库，可以把 Registry 替换为自己的注册服务器）
- Reponsitory：镜像仓库（通常把一组相关联的镜像归为一个镜像仓库，`library `为 Docker 默认的镜像仓库)
- image: 镜像名称
- Tag：镜像的标签，如果不指定拉取镜像的标签，默认为`latest`。

栗子：需要获取一个 busybox 镜像，可以执行以下命令：

> busybox 是一个集成了数百个 Linux 命令（例如 curl、grep、mount、telnet 等）的精简工具箱，只有几兆大小，被誉为 Linux 系统的瑞士军刀。我经常会使用 busybox 做调试来查找生产环境中遇到的问题。

```bash
$ docker pull busybox
Using default tag: latest
latest: Pulling from library/busybox
61c5ed1cbdf8: Pull complete
Digest: sha256:4f47c01fa91355af2865ac10fef5bf6ec9c7f42ad2321377c21e844427972977
Status: Downloaded newer image for busybox:latest
docker.io/library/busybox:latest

$ docker 

```
> 执行`docker pull busybox`命令，都是先从本地搜索，如果本地搜索不到`busybox`镜像则从 Docker Hub 下载镜像。
{: .prompt-warning }

## 查看镜像

Docker 镜像查看使用 `docker images` 或者`docker image ls`命令。

- `docker images`命令列出本地所有的镜像

```bash
$ docker images

REPOSITORY  TAG       IMAGE ID            CREATED             SIZE
nginx       latest    4bb46517cac3        9 days ago          133MB
nginx       1.15      53f3fd8007f7        15 months ago       109MB
busybox     latest    018c9d7b792b        3 weeks ago         1.22MB

```

- `docker image ls`命令来查询指定的镜像

```bash
$ docker image ls busybox

REPOSITORY  TAG      IMAGE ID      CREATED        SIZE
busybox     latest   018c9d7b792b  3 weeks ago    1.22MB

```

- `docker images`命令列出所有镜像，然后使用`grep`命令进行过滤

```bash
$ docker images |grep busybox
busybox    latest    018c9d7b792b    3 weeks ago   1.22MB
```
## 重命名镜像

命令：`docker tag`自定义镜像名称或者推送镜像到其他镜像仓库
命令格式：docker tag [SOURCE_IMAGE][:TAG] [TARGET_IMAGE][:TAG]

栗子：
```bash
$ docker tag busybox:latest mybusybox:latest
```
执行完docker tag命令后，可以使用查询镜像命令查看一下镜像列表：
```bash
$ docker images

REPOSITORY   TAG      IMAGE ID         CREATED             SIZE
busybox      latest   018c9d7b792b     3 weeks ago         1.22MB
mybusybox    latest   018c9d7b792b     3 weeks ago         1.22MB

```
可以看到，镜像列表中多了一个mybusybox的镜像。但细心的同学可能已经发现，busybox和mybusybox这两个镜像的 IMAGE ID 是完全一样的。为什么呢？实际上它们指向了同一个镜像文件，只是别名不同而已。

## 删除镜像
命令：`docker rmi`或`docker image rm`命令删除镜像
举例：使用以下命令删除`mybusybox`镜像
```bash
$ docker rmi mybusybox 
Untagged: mybusybox:latest 
```
此时，再次使用docker images命令查看下机器上的镜像列表。
```bash
$ docker images 
REPOSITORY   TAG      IMAGE ID       CREATED             SIZE 
busybox      latest   018c9d7b792b   3 weeks ago         1.22MB 
```
通过上面的输出，可以看到`mybusybox`镜像已经被删除。

## 构建镜像

构建镜像主要有两种方式：

1. 从运行中的容器提交为镜像；
   1. 使用`docker commit`命令
2. 从 Dockerfile 构建镜像
   1. 使用`docker build`命令
   2. 最重要最常用的镜像构建方式
   3. Dockerfile 包含了用户所有构建命令的文本

### 从运行中的容器提交为镜像
命令：docker commit

1. 使用命令创建一个名为 busybox 的容器并进入 busybox 容器；
```bash
$ docker run --rm --name=busybox -it busybox sh
```

2. 执行完上面的命令后，当前窗口会启动一个 busybox 容器并且进入容器中。在容器中，执行以下命令创建一个文件并写入内容：
```bash
$ touch hello.txt && echo "I love Docker. " > hello.txt
```

3. 新打开另一个命令行窗口，运行以下命令提交镜像：
```bash
$ docker commit busybox busybox:hello
sha256:cbc6406aaef080d1dd3087d4ea1e6c6c9915ee0ee0f5dd9e0a90b03e2215e81c
```

4. 最后使用`docker image ls`命令查看镜像：


此时就看到主机上新生成了 busybox:hello 这个镜像。

### 从 Dockerfile 构建镜像

命令：docker build
使用 Dockerfile 构建的镜像具有以下特性：

- Dockerfile 的每一行命令都会生成一个独立的镜像层，并且拥有唯一的 ID
- Dockerfile 的命令是完全透明的，通过查看 Dockerfile 的内容，就可以知道镜像是如何一步步构建的
- Dockerfile 是纯文本的，方便跟随代码一起存放在代码仓库并做版本管理

Dockerfile 常用的指令：

```bash
# 基于 `centos:7`这个镜像来构建自定义镜像
# 每个 Dockerfile 的第一行除了注释都必须以 FROM 开头
FROM centos:7
# 拷贝贝本地文件 `nginx.repo` 文件到容器内的 `/etc/yum.repos.d` 目录下
# 这里拷贝 `nginx.repo` 文件是为了添加 nginx 的安装源
COPY nginx.repo /etc/yum.repos.d/nginx.repo
# 
RUN yum install -y nginx
# 声明容器内业务（nginx）使用 80 端口对外提供服务
ENV HOST=mynginx
# 定义容器的启动命令，命令格式为 json 数组
# 设置了容器的启动命令为 nginx 
# 添加了 nginx 的启动参数 -g 'daemon off;' ，使得 nginx 以前台的方式启动
CMD ["nginx","-g","daemon off;"]

```
上面这个 Dockerfile 的例子基本涵盖了常用的镜像构建指令。

# 镜像的实现原理

- Docker 镜像是由一系列镜像层（layer）组成的，
- 每一层代表了镜像构建过程中的一次提交

下面以一个镜像构建的 Dockerfile 来说明镜像是如何分层的。
```bash
# 基于 busybox 创建一个镜像层
FROM busybox
# 拷贝本机 test 文件到镜像内
COPY test /tmp/test
# 在 /tmp 文件夹下创建一个目录 testdir
RUN mkdir /tmp/testdir
```

为验证镜像存储结构，使用`docker build`命令在上面 Dockerfile 所在目录构建一个镜像：

```bash
$ docker build -t busybox .
```
这里的 Docker 使用的是 overlay2 文件驱动，进入到/var/lib/docker/overlay2目录下使用tree .命令查看产生的镜像文件：

```bash
$ tree .
# 以下为 tree . 命令输出内容
|-- 3e89b959f921227acab94f5ab4524252ae0a829ff8a3687178e3aca56d605679
|   |-- diff  # 这一层为基础层，对应上述 Dockerfile 第一行，包含 busybox  镜像所有文件内容，例如 /etc,/bin,/var 等目录

... 此次省略部分原始镜像文件内容

|   `-- link 
|-- 6591d4e47eb2488e6297a0a07a2439f550cdb22845b6d2ddb1be2466ae7a9391
|   |-- diff   # 这一层对应上述 Dockerfile 第二行，拷贝 test 文件到 /tmp 文件夹下，因此 diff 文件夹下有了 /tmp/test 文件
|   |   `-- tmp
|   |       `-- test
|   |-- link
|   |-- lower
|   `-- work
|-- backingFsBlockDev
|-- bec6a018080f7b808565728dee8447b9e86b3093b16ad5e6a1ac3976528a8bb1
|   |-- diff  # 这一层对应上述 Dockerfile 第三行，在 /tmp 文件夹下创建 testdir 文件夹，因此 diff 文件夹下有了 /tmp/testdir 文件夹
|   |   `-- tmp
|   |       `-- testdir
|   |-- link
|   |-- lower
|   `-- work
...
```

通过上面的目录结构可以看到，Dockerfile 的每一行命令，都生成了一个镜像层，每一层的 diff 夹下只存放了增量数据，如图 2 所示。
![镜像文件系统.png](https://images.happymaya.cn/assert/docker/docker_image_file_system.png)

分层的结构使得 Docker 镜像非常轻量，每一层根据镜像的内容都有一个唯一的 ID 值，当不同的镜像之间有相同的镜像层时，便可以实现不同的镜像之间共享镜像层的效果。
总结一下， 

- Docker 镜像是静态的分层管理的文件组合
- 镜像底层的实现依赖于联合文件系统（UnionFS）

# 总结

镜像操作命令：
1. 拉取镜像，使用 docker pull 命令拉取远程仓库的镜像到本地
2. 重命名镜像，使用 docker tag 命令“重命名”镜像
3. 查看镜像，使用 docker image ls 或 docker images 命令查看本地已经存在的镜像
4. 删除镜像，使用 docker rmi 命令删除无用镜像
5. 构建镜像，构建镜像有两种方式。第一种方式是使用 docker build 命令基于 Dockerfile 构建镜像，也是我比较推荐的镜像构建方式；第二种方式是使用 docker commit 命令基于已经运行的容器提交为镜像


镜像的实现原理：
镜像是由一系列的镜像层（layer ）组成，每一层代表了镜像构建过程中的一次提交，当我们需要修改镜像内的某个文件时，只需要在当前镜像层的基础上新建一个镜像层，并且只存放修改过的文件内容。分层结构使得镜像间共享镜像层变得非常简单和方便。



