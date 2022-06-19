---
title: 解决突入起来 GC 问题的思路
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2019-12-18 22:13:00 +0800
categories: [Java, JVM]
tags: [JVM, GC]
math: true
mermaid: true
---

想要下手解决 GC 问题，首先需要掌握下面三种问题：

1. 使用 jstat 命令查看 JVM 的 GC 情况；
2. 面对海量 GC 日志参数，快速抓住问题根源！
3. 掌握的日志分析工具。

工欲善其事，必先利其器。优化手段，包括代码优化、扩容、参数优化，甚至估算，都需要一些支撑信息加以判断。

对于 JVM 来说，一种情况是 GC 时间过长，会影响用户的体验，这个时候就需要调整某些 JVM 参数、观察日志。


另外一种情况就比较严重了，发生了 OOM，或者操作系统的内存溢出。服务直接宕机，需要寻找背后的原因。


这时，GC 日志能够帮找到问题的根源。

## GC 日志输出

最近几年 Java 的版本更新速度是很快的，JVM 的参数配置其实变化也很大。

就拿 GC 日志这一块来说，Java 9 几乎是推翻重来。网络上的一些文章，把这些参数写的乱七八糟，根本不能投入生产。如果碰到不能被识别的参数，先确认一下自己的 Java 版本。


在事故出现的时候，通常并不是那么温柔。在半夜里就能接到报警电话，这是因为很多定时任务都设定在夜深人静的时候执行。

这个时候，再去看 jstat 已经来不及了，需要保留现场。这个便是`看门狗`的工作，看**门狗可以通过设置一些 JVM 参数进行配置。**下面命令行：

在 JDK8 中的使用

```shell
#!/bin/sh
LOG_DIR="/tmp/logs"
JAVA_OPT_LOG=" -verbose:gc"
JAVA_OPT_LOG="${JAVA_OPT_LOG} -XX:+PrintGCDetails"
JAVA_OPT_LOG="${JAVA_OPT_LOG} -XX:+PrintGCDateStamps"
JAVA_OPT_LOG="${JAVA_OPT_LOG} -XX:+PrintGCApplicationStoppedTime"
JAVA_OPT_LOG="${JAVA_OPT_LOG} -XX:+PrintTenuringDistribution"
JAVA_OPT_LOG="${JAVA_OPT_LOG} -Xloggc:${LOG_DIR}/gc_%p.log"
JAVA_OPT_OOM=" -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=${LOG_DIR} -XX:ErrorFile=${LOG_DIR}/hs_error_pid%p.log "
JAVA_OPT="${JAVA_OPT_LOG} ${JAVA_OPT_OOM}"
JAVA_OPT="${JAVA_OPT} -XX:-OmitStackTraceInFastThrow"
```

合成一行，如下：

```shell
-verbose:gc -XX:+PrintGCDetails -XX:+PrintGCDateStamps 
-XX:+PrintGCApplicationStoppedTime -XX:+PrintTenuringDistribution 
-Xloggc:/tmp/logs/gc_%p.log -XX:+HeapDumpOnOutOfMemoryError 
-XX:HeapDumpPath=/tmp/logs -XX:ErrorFile=/tmp/logs/hs_error_pid%p.log 
-XX:-OmitStackTraceInFastThrow
```

这些参数的含义如下表：

| 参数                          | 意义                                                       |
| ----------------------------- | ---------------------------------------------------------- |
| -verbose:gc                   | 打印 GC 日志                                               |
| PrintGCDetails                | 打印详细 GC 日志                                           |
| PrintGCDateStamps             | 系统时间，更加可读，PrintGCTimeStamps 是 JVM 启动时间      |
| PrintGCApplicationStoppedTime | 打印 STW 时间                                              |
| PrintTenuringDistribution     | 打印对象年龄分布，对调优 MaxTenuringThreshold 参数帮助很大 |
| loggc                         | 将以上 GC 内容输出到文件中                                 |

OOM 时的参数：

| 参数                       | 意义                       |
| -------------------------- | -------------------------- |
| HeapDumpOnOutOfMemoryError | OOM 时 Dump 信息，非常有用 |
| HeapDumpPath               | Dump 文件保存路径          |
| ErrorFile                  | 错误日志存放路径           |

最后的 OmitStackTraceInFastThrow ，是 JVM 用来缩简日志输出的。开启这个参数之后，如果多次发生了空指针异常，将会打印以下信息：

```
java.lang.NullPointerException
java.lang.NullPointerException
java.lang.NullPointerException
java.lang.NullPointerException
```

在实际生产中，这个参数是默认开启的，这样导致有时候排查问题非常不方便（很多研发对此无能为力），这里把它关闭，但这样它会输出所有的异常堆栈，日志会多很多！！！

## GC 日志的意义

1. 表示 GC 发生的时间，一般使用可读的方式打印；
2. 表示日志表明是 G1 的“转移暂停: 混合模式”，停顿了约 223ms；
3. 表明由 8 个 Worker 线程并行执行，消耗了 214ms；
4. 表示 Diff 越小越好，说明每个工作线程的速度都很均匀；
5. 表示外部根区扫描，外部根是堆外区。JNI 引用，JVM 系统目录，Classloaders 等；
6. 表示更新 RSet 的时间信息；
7. 表示该任务主要是对 CSet 中存活对象进行转移（复制）；
8. 表示花在 GC 之外的工作线程的时间；
9. 表示并行阶段的 GC 总时间；
10. 表示其他清理活动；
11. 表示收集结果统计；
12. 表示时间花费统计。

**GC 日志描述了垃圾回收器过程中的几乎每一个阶段。**但即使了解了这些数值的意义，在分析问题时，也会感到吃力，可以借助图形化的分析工具进行分析。

尤其注意的是最后一行日志，需要详细描述。可以看到 GC 花费的时间，竟然有 3 个数值。如果你手有 Linux 机器，可以执行以下命令：

```
```

可以看到一段命令的执行，有三种纬度的时间统计：

- real 实际花费的时间，指的是从开始到结束所花费的时间。比如进程在等待 I/O 完成，这个阻塞时间也会被计算在内；
- user 指的是进程在用户态（User Mode）所花费的时间，只统计本进程所使用的时间，注意是指多核；
- sys 指的是进程在核心态（Kernel Mode）花费的 CPU 时间量，指的是内核中的系统调用所花费的时间，只统计本进程所使用的时间。

在上面的 GC 日志中，real < user + sys，因为使用了多核进行垃圾收集，所以实际发生的时间比 (user + sys) 少很多。在多核机器上，这很常见。

`[Times: user=1.64 sys=0.00, real=0.23 secs]`


下面是一个串行垃圾收集器收集的 GC 时间的示例。由于串行垃圾收集器始终仅使用一个线程，因此实际使用的时间等于用户和系统时间的总和：

`[Times: user=0.29 sys=0.00, real=0.29 secs]`


统计 GC的以哪个时间为准呢？一般来说，用户只关心系统停顿了多少秒，对实际的影响时间非常感兴趣。至于背后是怎么实现的，是多核还是单核，是用户态还是内核态，它们都不关心。所以直接使用 real 字段。

## GC日志可视化

肉眼可见的这些日志信息，让人非常头晕，尤其是日志文件特别大的时候。所幸现在有一些在线分析平台，可以帮助我们分析这个过程。最常用的 [gceasy](https://blog.gceasy.io/) 。


以下是一个使用了 G1 垃圾回收器，堆内存为 6GB 的服务，运行 5 天的 GC 日志。

（1）堆信息

（2）关键信息

（3）交互式图表

（4）G1 的时间耗时

（5）其他



### jstat

上面的可视化工具，必须经历导出、上传、分析三个阶段，这种速度太慢了。有没有可以实时看堆内存的工具？


你可能会第一时间想到 jstat 命令。第一次接触这个命令，我也是很迷惑的，主要是输出的字段太多，不了解什么意义。

但其实了解我们在前几节课时所讲到内存区域划分和堆划分之后，再看这些名词就非常简单了。

拿 -gcutil 参数来说明一下。


jstat -gcutil $pid 1000

只需要提供一个 Java 进程的 ID，然后指定间隔时间（毫秒）就 OK 了。



可以看到，E 其实是 Eden 的缩写，S0 对应的是 Surivor0，S1 对应的是 Surivor1，O 代表的是 Old，而 M 代表的是 Metaspace。


YGC 代表的是年轻代的回收次数，YGC T对应的是年轻代的回收耗时。那么 FGC 肯定代表的是 Full GC 的次数。


你在看日志的时候，一定要注意其中的规律。-gcutil 位置的参数可以有很多种。我们最常用的有 gc、gcutil、gccause、gcnew 等，其他的了解一下即可。

    gc: 显示和 GC 相关的 堆信息；
    
    gcutil: 显示 垃圾回收信息；
    
    gccause: 显示垃圾回收 的相关信息（同 -gcutil），同时显示 最后一次 或 当前 正在发生的垃圾回收的 诱因；
    
    gcnew: 显示 新生代 信息；
    
    gccapacity: 显示 各个代 的 容量 以及 使用情况；
    
    gcmetacapacity: 显示 元空间 metaspace 的大小；
    
    gcnewcapacity: 显示 新生代大小 和 使用情况；
    
    gcold: 显示 老年代 和 永久代 的信息；
    
    gcoldcapacity: 显示 老年代 的大小；
    
    printcompilation: 输出 JIT 编译 的方法信息；
    
    class: 显示 类加载 ClassLoader 的相关信息；
    
    compiler: 显示 JIT 编译 的相关信息；


如果 GC 问题特别明显，通过 jstat 可以快速发现。我们在启动命令行中加上参数 -t，可以输出从程序启动到现在的时间。如果 FGC 和启动时间的比值太大，就证明系统的吞吐量比较小，GC 花费的时间太多了。另外，如果老年代在 Full GC 之后，没有明显的下降，那可能内存已经达到了瓶颈，或者有内存泄漏问题。


下面这行命令，就追加了 GC 时间的增量和 GC 时间比率两列。 	

## GC 日志也会搞鬼

ElasticSearch 的速度是非常快的，为了压榨它的性能，对磁盘的读写几乎是全速的。它在后台做了很多 Merge 动作，将小块的索引合并成大块的索引。还有 TransLog 等预写动作，都是 I/O 大户。


使用 iostat -x 1 可以看到具体的 I/O 使用状况。


问题是，我们有一套 ES 集群，在访问高峰时，有多个 ES 节点发生了严重的 STW 问题。有的节点竟停顿了足足有 7~8 秒。

 [Times: user=0.42 sys=0.03, real=7.62 secs] 


从日志可以看到在 GC 时用户态只停顿了 420ms，但真实的停顿时间却有 7.62 秒。


盘点一下资源，唯一超额利用的可能就是 I/O 资源了（%util 保持在 90 以上），GC 可能在等待 I/O。


通过搜索，发现已经有人出现过这个问题，这里直接说原因和结果。

原因就在于，写 GC 日志的 write 动作，是统计在 STW 的时间里的。在我们的场景中，由于 ES 的索引数据，和 GC 日志放在了一个磁盘，GC 时写日志的动作，就和写数据文件的动作产生了资源争用。

解决方式也是比较容易的，把 ES 的日志文件，单独放在一块普通 HDD 磁盘上就可以了。