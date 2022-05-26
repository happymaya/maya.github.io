---
title: 获取代码性能数据的工具总结（04）
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2019-03-10 23:33:00 +0800
categories: [Java, Performance Optimization]
tags: [性能优化, Performance Optimization]
math: true
mermaid: true
---

为什么 Kafka 操作磁盘，吞吐量还能那么高？这是因为，磁盘之所以慢，主要就是慢在寻道的操作上面。Kafka 官方测试表明，这个寻道时间长达 10ms。磁盘的顺序写和随机写的速度比，可以达到 6 千倍，Kafka 就是采用的**顺序写**的方式。

如果深入排查，需要收集较详细的性能数据，包括**操作系统性能数据**、**JVM 的性能数据**、**应用的性能数据**等。

在平常工作中，我经常使用以下五款性能测试性能工具

## nmon：获取系统性能数据

除了 top、free 等命令，还有一些将资源整合在一起的监控工具。

nmon 是一个老牌的 Linux 性能监控工具，它不仅有漂亮的监控界面（如下图所示），还能产出细致的监控报表。

在对应用做性能评估时，通常会加上 nmon 的报告，这会让测试结果更加有说服力。

操作系统性能指标，都可从 nmon 中获取。它的监控范围很广，包括 **CPU**、**内存**、**网络**、**磁盘**、**文件系统**、**NFS**、**系统资源**等信息。

nmon 在 sourceforge 发布，它的使用非常简单，比如 CentOS 7 系统，选择对应的版本即可执行。
```bash
./nmon_x86_64_centos7
```
- 按 C 键可加入 CPU 面板；
- 按 M 键可加入内存面板；
- 按 N 键可加入网络；
- 按 D 键可加入磁盘等。

通过下面的命令，表示每 5 秒采集一次数据，共采集 12 次，它会把这一段时间之内的数据记录下来。

比如本次生成了 localhost_200623_1633.nmon 这个文件，把它从服务器上下载下来
```bash
./nmon_x86_64_centos7  -f -s 5 -c 12 -m  -m .
```

**注意：执行命令之后，可以通过 ps 命令找到这个进程。**

```bash
[root@localhost nmon16m_helpsystems]# ps -ef| grep nmon
root      2228     1  0 16:33 pts/0    00:00:00 ./nmon_x86_64_centos7 -f -s 5 -c 12 -m .
```
此外，使用 nmonchart 工具，即可生成 html 文件。

## jvisualvm：获取 JVM 性能数据

jvisualvm 原是随着 JDK 发布的一个工具，Java 9 之后开始单独发布。通过它，可以了解应用在运行中的内部情况。可以连接本地或者远程的服务器，监控大量的性能数据。

通过插件功能，jvisualvm 能获得更强大的扩展（建议把所有的插件下载下来进行体验）

要想监控远程的应用，需要在被监控的 App 上加入 jmx 参数。
```bash
-Dcom.sun.management.jmxremote.port=14000
-Dcom.sun.management.jmxremote.authenticate=false 
-Dcom.sun.management.jmxremote.ssl=false

# 注意：jmx 参数是加在java命令后面的
```

上述配置的意义是开启 JMX 连接端口 14000，同时配置不需要 SSL 安全认证方式连接。

对于性能优化来说，主要用到它的采样器。注意，由于抽样分析过程对程序运行性能有较大的影响，一般只在测试环境中使用此功能。

对于一个 Java 应用来说，除了要关注 CPU 指标，垃圾回收方面也是不容忽视的性能点，主要关注以下三点。

- **CPU 分析**：统计方法的执行次数和执行耗时，这些数据可用于分析哪个方法执行时间过长，成为热点等
- **内存分析**：可以通过内存监视和内存快照等方式进行分析，进而检测内存泄漏问题，优化内存使用情况
- **线程分析**：可以查看线程的状态变化，以及一些死锁情况

## JMC：获取 Java 应用详细性能数据

对于常用的 HotSpot 来说，更更强大的工具 JMC（Java Mission Control）。

JMC 集成了一个非常好用的功能：JFR（Java Flight Recorder）。

Flight Recorder 源自飞机的黑盒子，是用来录制信息然后事后分析的。

在 Java 11 中，它可以通过 jcmd 命令进行录制，主要包括 configure、check、start、dump、stop 这五个命令，其执行顺序为，start — dump — stop，例如：

```bash
jcmd <pid> JFR.start
jcmd <pid> JFR.dump filename=recording.jfr
jcmd <pid> JFR.stop
```
JFR 功能是建在 JVM 内部的，不需要额外依赖，可以直接使用，它能够监测大量数据。

比如，常提到的锁竞争、延迟、阻塞等；甚至在 JVM 内部，比如 SafePoint、JIT 编译等，也能去分析。

**JMC 集成了 JFR 的功能，参考**[**JMC 使用教程**](https://www.freesion.com/article/1313912924/)

# Arthas：获取单个请求的调用链耗时

Arthas 是一个 Java 诊断工具，可以排查内存溢出、CPU 飙升、负载高等内容，可以说是一个 jstack、jmap 等命令的大集合。详细使用方法详见[官方文档](https://arthas.aliyun.com/doc/)

# wrk 获取 Web 接口的性能数据
wrk 是一款 HTTP 压测工具，和 ab 命令类似，是一个命令行工具。

它的执行结果一般如下：
```bash
Running 30s test @ http://127.0.0.1:8080/index.html
12 threads and 400 connections
Thread Stats   Avg      Stdev     Max   +/- Stdev
Latency   635.91us    0.89ms  12.92ms   93.69%
Req/Sec    56.20k     8.07k   62.00k    86.54%
22464657 requests in 30.00s, 17.76GB read
Requests/sec: 748868.53
Transfer/sec:    606.33MB
```
可以看到，wrk 统计了常见的性能指标，对 Web 服务性能测试非常有用。同时，wrk 支持 Lua 脚本，用来控制 setup、init、delay、request、response 等函数，可以更好地模拟用户请求。

## 总结

- nmon 获取系统性能数据；
- jvisualvm 获取 JVM 性能数据；
- jmc 获取 Java 应用详细性能数据；
- arthas 获取单个请求的调用链耗时；
- wrk 获取 Web 接口的性能数据。

这些工具有偏低层的、有偏应用的、有偏统计的、有偏细节的，在定位性能问题时，灵活使用这些工具，既从全貌上掌握应用的属性，也从细节上找到性能的瓶颈，对应用性能进行全方位的掌控。

一般在服务器上使用图形化工具的不多，要真使用的话，还需要开启 JMX 端口。性能优化不仅仅是面对线上环境的，像很多公司已经具备了**全链路压测**的功能，在开发测试或者预发布环境，都可以进行性能优化。在工具中，nmon 和 arthas都是命令行的
