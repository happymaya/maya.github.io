---
title: JVM
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2019-03-15 23:33:00 +0800
categories: [Java, JVM]
tags: [JVM]
math: true
mermaid: true
---


1. 为什么 Java 研发系统需要 JVM ?
2. JVM 的原理是什么 ？
3. 写的 Java 代码到底是如何运行起来的 ？

想要完美回答这三个问题，就需要首先了解 JVM 是什么 ？ 它和 Java 有什么关系 ？ 又与 JDK 有什么渊源 ？

为了弄清楚这写问题，我是从以下三个维度思考的：

1. JVM 和操作系统的关系 ？
2. JVM、JRE、JDK 的关系 ？
3. Java 虚拟机规范和 Java 语言规范的关系 ？

# JVM 和 操作系统的关系

在武侠小说中，想要炼制一把睥睨天下的宝剑，是需要下一番功夫的。除了上等的铸剑技术，还需要一鼎经百炼的剑炉，如果将工程师比作铸剑师，那 JVM 便是剑炉！

JVM 的全称是 Java Vitual Machine ，也就是常说的 Java 虚拟机。

JVM 可以识别以 `.class`后缀的文件（字节码文件），并且能够解析它（以`.class`后缀的文件），最终调用操作系统上的函数，完成我们想要的操作。

一般而言，使用 C++ 编写的程序，编译成二进制文件后，能够被操作系统直接识别，因此可以直接执行；而使用 Java 编写的程序不一样，使用 `javac`命令编译成`.class`文件（字节码文件）之后，操作系统并不认识这些`.class`文件，还需要使用 Java 命令去主动执行它。

为什么 Java 程序不能像 C++ 一样，直接在操作系统上运行编译后的二进制文件呢 ？而非要搞一个处于程序和操作系统中间层的虚拟机呢 ？

这就是 JVM 过人之处！众所周知，Java 是一门抽象程度很高的编程语言，提供了自动内存管理等一系列的特性。这些特性直接在操作系统上实现是不可能的，所以需要 JVM 进行一番转换。

可以做如下类比：

- JVM 等同于操作系统
- Java 字节码（以 .class 后缀结尾的文件），等同于汇编语言

对于 Java 字节码，比较容易读懂，这也从侧面反映 Java 语言的抽象程度比较高。可以把 JVM 认为是一个翻译器，会持续不断的翻译执行 Java 字节码，然后调用真正的操作系统的函数，这些操作系统函数是与平台息息相关的。

可以借助下图，更好的理解。
[点击查看【processon】](https://www.processon.com/view/link/62321ce41efad407e999ea63)

从上图可以看出，有了 JVM 这一层，Java 就可以实现跨平台了。JVM 只需要保证能够正确执行以`.class`文件，就可以运行在诸如 Linux、Windows、MacOS 等平台上了。

Java 跨平台的意义在于一次编译，处处运行，能够做到这一点，JVM 功不可没。比如在 Maven 仓库下载同一版本的 jar 包就可以到处运行，不需要在每个平台上再编译一次。

现在一些 JVM 的扩展语言，比如 Clojure、JRuby、Groovy 等，编译到最后都是`.class`文件，Java 语言的维护者，只需要控制好 JVM 这个解析器，就可以将这些扩展语言无缝的运行在 JVM 之上了。

一句话总结 JVM 与操作系统之间的关系，如下：
:::tips
JVM 上承开发语言，下接操作系统，它的中间接口就是字节码。
:::

而 Java 程序和 C++ 程序的不同如下图所示。

[点击查看【processon】](https://www.processon.com/view/link/623227470e3e7407da50bc7f)
通过上图的对比，可以看出

-  C++ 程序是编译成操作系统能够识别的 `.exe`文件；
- Java 程序是编译成 JVM 能够识别的 `.class`文件，然后由 JVM 负责调用系统函数执行程序

# JVM、JRE 、JDK 的关系
通过上文，可以了解到 JVM 是 Java 程序能够运行的核心。换个角度想一想，JVM 没有了以 `.class`为后缀的文件，什么也干不了。

需要为 JVM 提供生产原料，也就是以`.class`为后缀的文件。俗话说，巧妇难为无米之炊。JVM  虽然功能强大，但仍需要为它提供`.class`文件。

仅仅有 JVM 的话，Java 程序是无法完成一次编译，处处运行的。它需要一个基本的类库，比如：

- 怎么操作文件 ？
- 怎么连接网络等 ？

Java 体系是非常慷慨的，会一次性将 JVM 运行所需要的类库都传递给它。
:::tips
JVM 标准加上实现的一大堆基础类库，组成了 Java 运行时环境，也就是常说的 JRE ( Java Runtime Environment)
:::

有了 JRE 之后，Java 程序就可以运行了 。

:::tips
可以观察 Java 目录，如果只需要执行一些 Java 程序，只需要一个 JRE 就足够了。
:::

对于 JDK 来说，就更庞大了。除了 JRE，JDK 还提供了一些非常好用的小工具，比如：

- javac
- java
- jar 等
:::tips
JDK 是 Java 开发的核心
:::
JDK 的全称是 Java Development Kit . 其中，kit 这个单词是**装备**的意思，它就像一个无底洞，预示着我对它无休止的研究。

JVM、JRE、JDK 三者之间的关系，可以用一个包含关系表示，如下图所示。

[点击查看【processon】](https://www.processon.com/view/link/623231125653bb074b209563)
# Java 虚拟机规范和 Java 语言规范的关系
通常谈到 JVM ，首先想到的是它的垃圾回收器。其实它还很多部分，比如：

- 对字节码进行解析的执行引擎

广泛的讲，JVM 是一种规范，它是最官方、最为准确的文档；
侠义的讲，日常开发，使用 Hotspot 更多一些，谈到 Java 虚拟机规范时，会将它们等同起来

[点击查看【processon】](https://www.processon.com/view/link/62332b4fe0b34d074452bf3a)

上图的左半部分是 Java 虚拟机规范，其实就是为输入和执行字节码提供一个运行环境。
上图的右半部分是 Java 语言规范，比如 `switch`、`for`、`泛型`、`lambda`等相关程序。它们最终都会编译成字节码。而字节码是连接左右两部分的桥梁。

如果 `.class`文件的规格是不变的，这两部分是可以独立进行优化的。不过，Java 也会偶尔扩充一下 `.class`文件的格式，增加一些字节码指令，以便支持更多的特性。

如果把 Java 虚拟机看作一台抽象的计算机，它有自己的指令集以及各种运行时内存区域。

# Java 代码是如何运行起来的

比如下面的 HelloWorld.java ，它遵循的就是 Java 语言规范。其中，调用了 `System.out`等模块，也就是 JRE 里提供的类库。

```java
public class HelloWorld {
    public static void main(String[] args) {
    
        System.out.println("Hello World");
    
    }
}
```
       
使用 JDK 的工具 `javac`进行编译后，会产生 `HelloWorld`的字节码。

Java 字节码是沟通 JVM 和 Java 程序的桥梁，可以通过命令 `[javap](https://www.cnblogs.com/frinder6/p/5440173.html)`来看下字节码的结构：
```java
0 getstatic #2 <java/lang/System.out>
3 ldc #3 <Hello World>
5 invokevirtual #4 <java/io/PrintStream.println>
8 return
```
Java 虚拟机采用基于栈的架构，其指令由**操作码**和**操作数**组成。这些字节码指令，叫作 `opcode` 。其中，getstatic、Idc、invokevirtual、return 等，就是 opcode。

显然，Java 的字节码是非常容易理解的。

可以继续使用 [hexdump](https://www.cnblogs.com/kerrycode/p/5077687.html) 看一下字节码的二进制内容。与以上字节码对应的二进制，如下：

JVM 是依靠解析这些 opcode 和操作数来完成程序的执行的，当使用 Java 命令运行 `.class`文件的时候，相当于启动了一个 JVM 进程。

而后 JVM 会翻译这些字节码，它主要有两种执行方式

- 常见的是**解释执行**将 `opcode` 和操作数翻译成机器代码；
- 另一种执行方式是 JIT，就是常说的**即时编译**，它会在一定条件下将字节码解释成机器码后再执行

这些 `.class`文件会被加载、存放到 `metaspace`中，等待被调用，这里有一个**类加载器**的概念。

JVM 的程序运行，都是在栈上完成的，这和其他普通程序的执行是类似的，同样分为堆和栈。

比如 main 方法，就会将它分配给一个栈帧。当退出方法的时候，会弹出相应的栈帧，可以发现，大多数字节码指令，就是不断的对栈帧进行操作。

而其它大块数据，是存放在堆上的。

总之，Java 程序的运行如下图所示：
[点击查看【processon】](https://www.processon.com/view/link/62333a580e3e7407da54586f)

# 选用的版本

JVM 一个虚拟机规范，因此有非常多的实现。其中，最流行的就是 Oracle 的 HotSpot。

# 总结
由此，最开始的三个问题，就有了答案。

> 1 为什么 Java 研发需要 JVM

> JVM 解释的是类似于汇编语言的字节码，需要一个抽象的运行时环境。同时，这个虚拟环境也需要解决字节码加载、自动垃圾回收、并发等一系列问题。JVM 其实是一个规范，定义了 .class 文件的结构、加载机制、数据存储、运行时栈等诸多内容，最常用的 JVM 实现就是 Hotspot。
{: .prompt-warning }

> 2 JVM 的运行原理

> JVM 的生命周期是和 Java 程序的运行一样的，当程序运行结束，JVM 实例也跟着消失了。JVM 处于整个体系中的核心位置
{: .prompt-warning }

> 3. Java 代码到底是如何运行起来的

> 一个 Java 程序，首先经过 javac 编译成 .class 文件，然后 JVM 将其加载到`元数据(metaspace)`区，执行引擎将会通过`混合模式`执行这些字节码。执行时，会翻译成操作系统相关的函数。JVM 作为 .class 文件的黑盒存在，输入字节码，调用操作系统函数。

过程如下：Java 文件->编译器>字节码->JVM->机器码。
{: .prompt-warning }























