---
title: 内存泄漏的思路
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2019-12-20 12:13:00 +0800
categories: [Java, JVM]
tags: [JVM]
math: true
mermaid: true
---

当一个系统在发生 OOM 的时候，这种行为经常让我感到非常困惑。

因为 JVM 是运行在操作系统之上的，操作系统的一些限制，会严重影响 JVM 的行为。

故障排查是一个综合性的技术问题，在日常工作中要一定要增加自己的知识广度。多总结、多思考、多记录！

现在的互联网服务，一般都做了负载均衡。如果一个实例发生了问题，一定不要着急去重启。

万能的重启会暂时缓解问题，但如果不保留现场，可能就错失了解决问题的根本，担心的事情还会到来。

所以，当实例发生问题的时候：

1. **第一步是隔离**；隔离，就是把这台机器从请求列表里摘除；比如把 nginx 相关的权重设成零。在微服务中，也有相应的隔离机制，
2. **第二步才是问题排查。**

## GC 引起 CPU 飙升

我们有个线上应用，单节点在运行一段时间后，CPU 的使用会飙升，一旦飙升，一般怀疑某个业务逻辑的计算量太大，或者是触发了死循环（比如著名的 HashMap 高并发引起的死循环），但排查到最后其实是 GC 的问题。

在 Linux 上，分析哪个线程引起的 CPU 问题，通常有一个固定的步骤，如下：

1. `top`，使用 top 命令，查找到使用 CPU 最多的某个进程，记录它的 pid。使用 Shift + P 快捷键可以按 CPU 的使用率进行排序；
2. `top -Hp $pid`，再次使用 top 命令，加 -H 参数，查看某个进程中使用 CPU 最多的某个线程，记录线程的 ID；
3. `printf %x $tid`，使用 printf 函数，将十进制的 tid 转化成十六进制；
4. `jstack $pid > $pid.log`，使用 jstack 命令，查看 Java 进程的线程栈；
5. `less $pid.log`；使用 less 命令查看生成的文件，并查找刚才转化的十六进制 tid，找到发生问题的线程上下文。

在 jstack 日志中找到了 CPU 使用最多的几个线程。可以看到问题发生的根源，是堆已经满了，但是又没有发生 OOM，于是 GC 进程就一直在那里回收，回收的效果又非常一般，造成 CPU 升高应用假死。

接下来，就是具体问题排查，需要把内存 dump 一份下来，使用 MAT 等工具分析具体原因！

## 现场保留

这个过程是繁杂而冗长的，需要记忆很多内容。现场保留可以使用自动化方式将必要的信息保存下来，在线上系统会保留信息有：

### 瞬时态和历史态

这里引入了工作中经常使用的两个名词：瞬时态和历史态。

- 瞬时态是指当时发生的、快照类型的元素；
- 历史态是指按照频率抓取的，有固定监控项的资源变动图。


有很多信息，比如 CPU、系统内存等，瞬时态的价值就不如历史态来的直观一些。因为瞬时状态无法体现一个趋势性问题（比如斜率、求导等），而这些信息的获取一般依靠监控系统的协作。


但对于 lsof、heap 等，这种没有时间序列概念的混杂信息，体积都比较大，无法进入监控系统产生有用价值，就只能通过瞬时态进行分析。在这种情况下，瞬时态的价值反而更大一些。常见的堆快照，就属于瞬时状态。

**问题不是凭空产生的，在分析时，一般要收集系统的整体变更集合，比如代码变更、网络变更，甚至数据量的变化。**

![](C:\Users\hsc\Documents\maya\jvm\image\problem-hanlder-12.jpg)

### 保留信息

#### 系统当前网络连接

```shell
ss -antp > $DUMP_DIR/ss.dump 2>&1
```

ss 命令将系统的所有网络连接输出到 ss.dump 文件中。

使用 ss 命令而不是 netstat 的原因，是因为 netstat 在网络连接非常多的情况下，执行非常缓慢。


后续的处理，通过查看各种网络连接状态的梳理，来排查 TIME_WAIT 或者 CLOSE_WAIT，或者其他连接过高的问题，非常有用。

线上有个系统更新之后，监控到 CLOSE_WAIT 的状态突增，最后整个 JVM 都无法响应。**CLOSE_WAIT 状态的产生一般都是代码问题，使用 jstack 最终定位到是因为 HttpClient 的不当使用而引起的，多个连接不完全主动关闭。**

![](C:\Users\hsc\Documents\maya\jvm\image\network-ss.png)

#### 网络状态统计

```shell
netstat -s > $DUMP_DIR/netstat-s.dump 2>&1
```

此命令**将网络统计状态输出到 netstat-s.dump 文件中**。它能够**按照各个协议进行统计输出**，对把握当时整个网络状态，有非常大的作用。

```shell
sar -n DEV 1 2 > $DUMP_DIR/sar-traffic.dump 2>&1
```

上面这个命令，会**使用 sar 输出当前的网络流量**。在一些速度非常高的模块上，比如 Redis、Kafka，就经常发生跑满网卡的情况。如果你的 Java 程序和它们在一起运行，资源则会被挤占，表现形式就是网络通信非常缓慢。

#### 进程资源

```
lsof -p $PID > $DUMP_DIR/lsof-$PID.dump
```

非常强大的命令，通过查看进程，能看到打开了哪些文件，这是一个神器，可以以进程的维度来查看整个资源的使用情况，包括每条网络连接、每个打开的文件句柄。同时，也可以很容易的看到连接到了哪些服务器、使用了哪些资源。这个命令在资源非常多的情况下，输出稍慢，耐心等待！！！

#### CPU 资源

```java
mpstat > $DUMP_DIR/mpstat.dump 2>&1
vmstat 1 3 > $DUMP_DIR/vmstat.dump 2>&1
sar -p ALL  > $DUMP_DIR/sar-cpu.dump  2>&1
uptime > $DUMP_DIR/uptime.dump 2>&1
```

输出当前系统的 CPU 和负载，便于事后排查。这几个命令的功能，有不少重合。注意选择！！！

#### I/O 资源

```
iostat -x > $DUMP_DIR/iostat.dump 2>&1
```

以计算为主的服务节点，I/O 资源会比较正常，但有时也会发生问题，比如日志输出过多，或者磁盘问题等。此命令可以输出每块磁盘的基本性能信息，用来排查 I/O 问题。**GC 日志分磁盘问题，就可以使用这个命令去发现。**

#### 内存问题

```
free -h > $DUMP_DIR/free.dump 2>&1
```

free 命令能够大体展现操作系统的内存概况，这是故障排查中一个非常重要的点，比如 SWAP 影响了 GC，SLAB 区挤占了 JVM 的内存。

#### 其他全局

```shell
ps -ef > $DUMP_DIR/ps.dump 2>&1
dmesg > $DUMP_DIR/dmesg.dump 2>&1
sysctl -a > $DUMP_DIR/sysctl.dump 2>&1
```

dmesg 是许多静悄悄死掉的服务留下的最后一点线索。当然，ps 作为执行频率最高的一个命令，它当时的输出信息，也必然有一些可以参考的价值。


另外，由于内核的配置参数，会对系统和 JVM 产生影响，所以也输出了一份。

#### 进程快照，最后的遗言（jinfo）

```
${JDK_BIN}jinfo $PID > $DUMP_DIR/jinfo.dump 2>&1
```

此命令将输出 Java 的基本进程信息，包括环境变量和参数配置，可以查看是否因为一些错误的配置造成了 JVM 问题。

#### dump 堆信息

```
${JDK_BIN}jstat -gcutil $PID > $DUMP_DIR/jstat-gcutil.dump 2>&1
${JDK_BIN}jstat -gccapacity $PID > $DUMP_DIR/jstat-gccapacity.dump 2>&1
```

将输出当前的 gc 信息。一般，基本能大体看出一个端倪，如果不能，可将借助 jmap 来进行分析。

#### 堆信息

```bash
${JDK_BIN}jmap $PID > $DUMP_DIR/jmap.dump 2>&1
${JDK_BIN}jmap -heap $PID > $DUMP_DIR/jmap-heap.dump 2>&1
${JDK_BIN}jmap -histo $PID > $DUMP_DIR/jmap-histo.dump 2>&1
${JDK_BIN}jmap -dump:format=b,file=$DUMP_DIR/heap.bin $PID > /dev/null  2>&1
```

jmap 将会得到当前 Java 进程的 dump 信息。如上所示，其实最有用的就是第 4 个命令，但是前面三个能够让你初步对系统概况进行大体判断。


因为，第 4 个命令产生的文件，一般都非常的大。而且，需要下载下来，导入 MAT 这样的工具进行深入分析，才能获取结果。这是分析内存泄漏一个必经的过程。

#### JVM 执行栈

```
${JDK_BIN}jstack $PID > $DUMP_DIR/jstack.dump 2>&1
```

jstack 将会获取当时的执行栈。一般会多次取值，我们这里取一次即可。这些信息非常有用，能够还原 Java 进程中的线程情况。

```
top -Hp $PID -b -n 1 -c >  $DUMP_DIR/top-$PID.dump 2>&1
```

为了能够得到更加精细的信息，我们使用 top 命令，来获取进程中所有线程的 CPU 信息，这样，就可以看到资源到底耗费在什么地方了。

#### 高级替补

```
kill -3 $PID
```

有时候，jstack 并不能够运行，有很多原因，比如 Java 进程几乎不响应了等之类的情况。尝试向进程发送 kill -3 信号，这个信号将会打印 jstack 的 trace 信息到日志文件中，是 jstack 的一个替补方案。

对于 jmap 无法执行的问题，也有替补，那就是 GDB 组件中的 gcore，将会生成一个 core 文件。我们可以使用如下的命令去生成 dump：

```
${JDK_BIN}jhsdb jmap --exe ${JDK}java  --core $DUMP_DIR/core --binaryheap
```

jmap 命令，它在 9 版本里被干掉了，取而代之的是 jhsdb，可以像下面的命令一样使用：

```
jhsdb jmap  --heap --pid  37340
jhsdb jmap  --pid  37288
jhsdb jmap  --histo --pid  37340
jhsdb jmap  --binaryheap --pid  37340
```

- heap 参数能够看到大体的内存布局，以及每一个年代中的内存使用情况。这内存布局，以及在 VisualVM 中看到的 没有什么不同。但由于它是命令行，所以使用更加广泛。
- histo 能够大概的看到系统中每一种类型占用的空间大小，用于初步判断问题。比如某个对象 instances 数量很小，但占用的空间很大，这就说明存在大对象。但它也只能看大概的问题，要找到具体原因，还是要 dump 出当前 live 的对象。

### 内存泄漏的现象

一般内存溢出，表现形式就是 Old 区的占用持续上升，即使经过了多轮 GC 也没有明显改善。内存泄漏的根本就是，有些对象并没有切断和 GC Roots 的关系，可通过一些工具，能够看到它们的联系。

### 一个卡顿实例

有一个关于服务的某个实例，经常发生服务卡顿。由于服务的并发量是比较高的，所以表现也非常明显。每多停顿 1 秒钟，几万用户的请求就会感到延迟。


经过统计、类比了此服务其他实例的 CPU、内存、网络、I/O 资源，区别并不是很大，所以一度怀疑是机器硬件的问题。


接着对比了节点的 GC 日志，发现无论是 Minor GC，还是 Major GC，这个节点所花费的时间，都比其他实例长得多。


通过仔细观察，发现在 GC 发生的时候，vmstat 的 si、so 飙升的非常严重，这和其他实例有着明显的不同。使用 free 命令再次确认，发现 SWAP 分区，使用的比例非常高，引起的具体原因是什么呢？


更详细的操作系统内存分布，从 /proc/meminfo 文件中可以看到具体的逻辑内存块大小，有多达 40 项的内存信息，这些信息都可以通过遍历 /proc 目录的一些文件获取。注意到 slabtop 命令显示的有一些异常，dentry（目录高速缓冲）占用非常高。

问题最终定位到是运维小伙伴执行了一句命令：

```
find / | grep "x"
```

他是想找一个叫做 x 的文件，看看在哪台服务器上，结果，这些老服务器由于文件太多，扫描后这些文件信息都缓存到了 slab 区上。而服务器开了 swap，操作系统发现物理内存占满后，并没有立即释放 cache，导致每次 GC 都要和硬盘打一次交道。


解决方式就是关闭 SWAP 分区。


swap 是很多性能场景的万恶之源，建议禁用。当你的应用真正高并发了，SWAP 绝对能让你体验到它魔鬼性的一面：进程倒是死不了了，但 GC 时间长的却让人无法忍受。

### 内存泄漏

内存溢出和内存泄漏的区别：

- 内存溢出是一个结果，原因有内存空间不足、配置错误等因素。

- 内存泄漏是一个原因。原因不再被使用的对象、没有被回收、没有及时切断与 GC Roots 的联系，很大程度上是一些错误的编程方式，或者过多的无用对象创建引起的。

  - 举个例子，使用了 HashMap 做缓存，但是并没有设置超时时间或者 LRU 策略，造成了放入 Map 对象的数据越来越多，而产生了内存泄漏；

  - 还有一个经常发生的内存泄漏的例子，也是由于 HashMap 产生的。代码如下：

    ```java
    package cn.happymaya.jvmpractise.oom;
    
    import java.util.HashMap;
    import java.util.Map;
    import java.util.Objects;
    
    // leak examole
    public class HashMapLeakDemo {
    	// 由于没有重写 Key 类的 hashCode 和 equals 方法
    	// 造成了放入 HashMap 的所有对象都无法被取出来，它们和外界失联了。所以下面的代码结果是 null。
    	public static class Key {
    		String title;
    
    		public Key(String title) {
    			this.title = title;
    		}
    
    //		@Override
    //		public boolean equals(Object o) {
    //			if (this == o) return true;
    //			if (o == null || getClass() != o.getClass()) return false;
    //			Key key = (Key) o;
    //			return Objects.equals(title, key.title);
    //		}
    //
    //		@Override
    //		public int hashCode() {
    //			return Objects.hash(title);
    //		}
    	}
    
    	public static void main(String[] args) {
    		Map<Key, Integer> map = new HashMap<>();
    
    		map.put(new Key("1"), 1);
    		map.put(new Key("2"), 2);
    		map.put(new Key("3"), 2);
    
    		Integer integer = map.get(new Key("2"));
    		System.out.println(integer);
    	}
    }
    ```

    由于没有重写 Key 类的 hashCode 和 equals 方法，造成了放入 HashMap 的所有对象都无法被取出来，它们和外界失联了。所以代码结果是 null。

    即使提供了 equals 方法和 hashCode 方法，也要非常小心，**尽量避免使用自定义的对象作为 Key。**如下：


​    

  - 关于文件处理器的应用，在读取或者写入一些文件之后，由于发生了一些异常，close 方法又没有放在 finally 块里面，造成了文件句柄的泄漏。由于文件处理十分频繁，产生了严重的内存泄漏问题。
  - 另外，对 Java API 的一些不当使用，也会造成内存泄漏。有喜欢使用 String 的 intern 方法，但如果字符串本身是一个非常长的字符串，而且创建之后不再被使用，则会造成内存泄漏。