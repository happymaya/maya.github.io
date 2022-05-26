---
title: Docker Dockerfile（06）
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2020-08-07 23:43:00 +0800
categories: [虚拟化技术, Docker 落地笔记]
tags: [Docker]
math: true
mermaid: true
image:
  src: https://images.happymaya.cn/assert/docker/docker.jpeg
  width: 800
  height: 500
---
**生产实践中一定优先使用 Dockerfile 的方式构建镜像。** 因为使用 Dockerfile 构建镜像可以带来很多好处：

- 易于版本化管理，Dockerfile 本身是一个文本文件，方便存放在代码仓库做版本管理，可以很方便找到各个版本之间的变更历史
- 过程可追溯，Dockerfile 的每一行指令代表一个镜像层，根据 Dockerfile 的内容即可很明确地查看镜像的完整构建过程
- 屏蔽构建环境异构，使用 Dockerfile 构建镜像无须考虑构建环境，基于相同 Dockerfile 无论在哪里运行，构建结果都一致

虽然有这么多好处，但是如果 Dockerfile 使用不当也会引发很多问题。比如：

1. 镜像构建时间过长，甚至镜像构建失败
2. 镜像层数过多，导致镜像文件过大

因此，本文对在生产环境中编写最优的 Dockerfile 的经验总结。

# Dockerfile 书写原则
遵循以下 Dockerfile 书写原则，不仅可以使得自己的 Dockerfile 简洁明了，让协作者清楚地了解镜像的完整构建流程，还可以减少镜像的体积，加快镜像构建的速度和分发速度。

## 单一职责
容器的本质是进程，一个容器代表一个进程，因此不同功能的应用应该尽量拆分为不同的容器，每个容器只负责单一业务进程。
## 提供注释信息
Dockerfile 也是一种代码，应该保持良好的代码编写习惯，晦涩难懂的代码尽量添加注释，让协作者可以一目了然地知道每一行代码的作用，并且方便扩展和使用。
## 保持容器最小化
避免安装无用的软件包，比如在一个 nginx 镜像中，不需要安装 vim 、gcc 等开发编译工具。这样不仅可以加快容器构建速度，而且可以避免镜像体积过大
## 合理选择基础镜像
容器的核心是应用，因此只要基础镜像能够满足应用的运行环境即可。例如一个`Java`类型的应用运行时只需要`JRE`，并不需要`JDK`，因此基础镜像只需要安装`JRE`环境即可。

## 使用 .dockerignore 文件
在使用`git`时，可以使用`.gitignore`文件忽略一些不需要做版本管理的文件。

同理，使用`.dockerignore`文件允许在构建时，忽略一些不需要参与构建的文件，从而提升构建效率。`.dockerignore`的定义类似于`.gitignore`。

`.dockerignore`的本质是文本文件，Docker 构建时可以使用换行符来解析文件定义，每一行可以忽略一些文件或者文件夹。具体使用方式如下：

| 规则 | 含义 |
| --- | --- |
| # | # 开头的表示注释，# 后面所有内容将会被忽略 |
| _/tmp_ | 匹配当前目录下任何以 tmp 开头的文件或者文件夹 |
| *.md | 匹配以 .md 为后缀的任意文件 |
| tem? | 匹配以 tem 开头并且以任意字符结尾的文件，？代表任意一个字符 |
| !README.md | ! 表示排除忽略。例如 .dockerignore 定义如下：*.md \n !README.md，表示除了 README.md 文件外所有以 .md 结尾的文件。 |

## 尽量构建缓存
Docker 构建过程中，每一条 Dockerfile 指令都会提交为一个镜像层，下一条指令都是基于上一条指令构建的。

如果构建时发现要构建的镜像层的父镜像层已经存在，并且下一条命令使用了相同的指令，即可命中构建缓存。

Docker 构建时判断是否需要使用缓存的规则如下：

- 从当前构建层开始，比较所有的子镜像，检查所有的构建指令是否与当前完全一致，如果不一致，则不使用缓存
- 一般情况下，只需要比较构建指令即可判断是否需要使用缓存，但是有些指令除外（例如ADD和COPY）
- 对于ADD和COPY指令不仅要校验命令是否一致，还要为即将拷贝到容器的文件计算校验和（根据文件内容计算出的一个数值，如果两个文件计算的数值一致，表示两个文件内容一致 ），命令和校验和完全一致，才认为命中缓存

因此，基于 Docker 构建时的缓存特性，可以把不轻易改变的指令放到 Dockerfile 前面（例如安装软件包），而可能经常发生改变的指令放在 Dockerfile 末尾（例如编译应用程序）。

例如，定义一些环境变量并且安装一些软件包，可以按照如下顺序编写 Dockerfile：
```bash
FROM centos:7
# 设置环境变量指令放前面
ENV PATH /usr/local/bin:$PATH
# 安装软件指令放前面
RUN yum install -y make
# 把业务软件的配置,版本等经常变动的步骤放最后
...

```
按照上面原则编写的 Dockerfile 在构建镜像时，前面步骤命中缓存的概率会增加，可以大大缩短镜像构建时间。
## 正确设置时区
从 Docker Hub 拉取的官方操作系统镜像大多数都是 UTC 时间（世界标准时间）。

如果想要在容器中使用中国区标准时间（东八区），则要根据使用的操作系统修改相应的时区信息，下面是几种常用操作系统的修改方式：

### Ubuntu 和Debian 系统
Ubuntu 和Debian 系统可以向 Dockerfile 中添加以下指令：
```bash
RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
RUN echo "Asia/Shanghai" >> /etc/timezone
```

### CentOS系统
CentOS 系统则向 Dockerfile 中添加以下指令：
```bash
RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```
## 使用国内软件源加快镜像构建速度

由于常用的官方操作系统镜像基本都是国外的，软件服务器大部分也在国外，所以构建镜像的时候想要安装一些软件包可能会非常慢。

这里以 CentOS 7 为例，介绍一下如何使用 163 软件源（国内有很多大厂，例如阿里、腾讯、网易等公司都免费提供的软件加速源）加快镜像构建。

首先在容器构建目录创建文件 CentOS7-Base-163.repo，文件内容如下：
```bash
# CentOS-Base.repo
#
# The mirror system uses the connecting IP address of the client and the
# update status of each mirror to pick mirrors that are updated to and
# geographically close to the client.  You should use this for CentOS updates
# unless you are manually picking other mirrors.
#
# If the mirrorlist= does not work for you, as a fall back you can try the 
# remarked out baseurl= line instead.
#
#
[base]
name=CentOS-$releasever - Base - 163.com
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=os
baseurl=http://mirrors.163.com/centos/$releasever/os/$basearch/
gpgcheck=1
gpgkey=http://mirrors.163.com/centos/RPM-GPG-KEY-CentOS-7
#released updates

[updates]

name=CentOS-$releasever - Updates - 163.com
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=updates
baseurl=http://mirrors.163.com/centos/$releasever/updates/$basearch/
gpgcheck=1
gpgkey=http://mirrors.163.com/centos/RPM-GPG-KEY-CentOS-7
#additional packages that may be useful

[extras]

name=CentOS-$releasever - Extras - 163.com
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=extras
baseurl=http://mirrors.163.com/centos/$releasever/extras/$basearch/
gpgcheck=1
gpgkey=http://mirrors.163.com/centos/RPM-GPG-KEY-CentOS-7
#additional packages that extend functionality of existing packages

[centosplus]
name=CentOS-$releasever - Plus - 163.com
baseurl=http://mirrors.163.com/centos/$releasever/centosplus/$basearch/
gpgcheck=1
enabled=0
gpgkey=http://mirrors.163.com/centos/RPM-GPG-KEY-CentOS-7
```
然后在 Dockerfile 中添加如下指令：
```bash
COPY CentOS7-Base-163.repo /etc/yum.repos.d/CentOS7-Base.repo
```
执行完上述步骤后，再使用`yum install`命令安装软件时就会默认从 163 获取软件包，这样可以大大提升构建速度。

## 最小化镜像层数
在构建镜像时尽可能地减少 Dockerfile 指令行数。例如要在 CentOS 系统中安装`make`和`net-tools`两个软件包，应该在 Dockerfile 中使用以下指令：
```bash
RUN yum install -y make net-tools
```
而不应该写成这样：
```bash
RUN yum install -y make
RUN yum install -y net-tools
```

# Dockerfile 指令书写建议
针对常用的指令，书写建议如下：
## RUN

`RUN`指令在构建时将会生成一个新的镜像层并且执行 RUN 指令后面的内容。
使用`RUN`指令时应该尽量遵循以下原则：

- 当`RUN`指令后面跟的内容比较复杂时，建议使用反斜杠（\） 结尾并且换行；
- `RUN`指令后面的内容尽量按照字母顺序排序，提高可读性。

例如，在官方的 CentOS 镜像下安装一些软件，一个建议的 Dockerfile 指令如下：
```bash
FROM centos:7
RUN yum install -y automake \
                    curl \
                   python \
                   vim
```
## CMD和 ENTRYPOINT 

`CMD` 和 `ENTRYPOINT` 指令都是容器运行的命令入口，这两个指令使用中有很多相似的地方，但是也有一些区别。

这两个指令的相同点：

- 第一种为 `CMD/ENTRYPOINT["command" , "param"]`。这种格式是使用 Linux 的 exec 实现的， 一般称为 exec 模式，这种书写格式为 CMD/ENTRYPOINT 后面跟 json 数组，也是Docker 推荐的使用格式。
- 另外一种格式为 `CMD/ENTRYPOINTcommand param`，这种格式是基于 shell 实现的， 通常称为 shell模式。当使用	shell 模式时，Docker 会以 /bin/sh -c command 的方式执行命令。

使用 exec 模式启动容器时，容器的 1 号进程就是 CMD/ENTRYPOINT 中指定的命令。

而使用 shell 模式启动容器时相当于把启动命令放在了 shell 进程中执行，等效于执行`/bin/sh -c "task command"` 命令。

因此 shell 模式启动的进程在容器中实际上并不是 1 号进程。

这两个指令的区别：

- `Dockerfile` 中如果使用了 `ENTRYPOINT` 指令，启动 `Docker` 容器时需要使用 `--entrypoint` 参数才能覆盖 `Dockerfile` 中的 `ENTRYPOINT` 指令 ，而使用 `CMD `设置的命令则可以被 `docker run `后面的参数直接覆盖。
- `ENTRYPOINT` 指令可以结合 `CMD` 指令使用，也可以单独使用，而 `CMD` 指令只能单独使用。

如果希望镜像足够灵活，推荐使用`CMD`指令。

如果镜像只执行单一的具体程序，并且不希望用户在执行`docker run`时覆盖默认程序，建议使用`ENTRYPOINT`。

最后再强调一下，无论使用`CMD`还是`ENTRYPOINT`，都尽量使用`exec`模式。

## ADD 和 COPY

ADD 和 COPY 指令功能类似，都是从外部往容器内添加文件。但是  COPY 指令只支持基本的文件和文件夹拷贝功能，ADD 则支持更多文件来源类型，比如自动提取 tar 包，并且可以支持源文件为 URL 格式。

那么在日常应用中，更推荐使用 COPY 指令，因为 COPY 指令更加透明，仅支持本地文件向容器拷贝，而且使用 COPY 指令可以更好地利用构建缓存，有效减小镜像体积。

当要使用 ADD 向容器中添加 URL 文件时，请尽量考虑使用其他方式替代。例如想要在容器中安装 memtester（一种内存压测工具），应该避免使用以下格式：

```bash
ADD http://pyropus.ca/software/memtester/old-versions/memtester-4.3.0.tar.gz /tmp/
RUN tar -xvf /tmp/memtester-4.3.0.tar.gz -C /tmp
RUN make -C /tmp/memtester-4.3.0 && make -C /tmp/memtester-4.3.0 install
```
下面是推荐写法：
```bash
RUN wget -O /tmp/memtester-4.3.0.tar.gz http://pyropus.ca/software/memtester/old-versions/memtester-4.3.0.tar.gz \
&& tar -xvf /tmp/memtester-4.3.0.tar.gz -C /tmp \
&& make -C /tmp/memtester-4.3.0 && make -C /tmp/memtester-4.3.0 install
```
## work
为了使构建过程更加清晰明了，推荐使用 WORKDIR 来指定容器的工作路径，应该尽量避免使用 RUN cd /work/path && do some work 这样的指令。
最后给出几个常用软件的官方 Dockerfile 示例链接：

- [Go](https://github.com/docker-library/golang/blob/4d68c4dd8b51f83ce4fdce0f62484fdc1315bfa8/1.15/buster/Dockerfile)
- [Nginx](https://github.com/nginxinc/docker-nginx/blob/9774b522d4661effea57a1fbf64c883e699ac3ec/mainline/buster/Dockerfile)
- [Hy](https://github.com/hylang/docker-hylang/blob/f9c873b7f71f466e5af5ea666ed0f8f42835c688/dockerfiles-generated/Dockerfile.python3.8-buster)

## 分离编译环境和运行环境

编写编译型语言（例如 Golang、Java）的 Dockerfile 时，将分离编译环境（编译运行程序所需要的环境）和运行环境（应用程序正常运行所依赖的环境。例如编译 Java 应用程序需要 JDK 环境，而真正运行的 Java 程序则仅需要 JRE 环境即可），使得镜像体积尽可能小。

把 编译环境 单独打包镜像，只提供编译好的二进制（注意运行 CPU 架构）。运行环境单独做镜像，只考虑基础运行环境的配置。好处有：

- 程序发布单独版本管理，运行环境也单独组合
- 运行环境打补丁升级等等，不用影响程序发布



