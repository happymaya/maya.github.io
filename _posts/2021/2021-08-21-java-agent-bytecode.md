---
title: 使用 Java Agent 修改字节码指令
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2021-08-21 21:23:00 +0800
categories: [Java, JVM]
tags: [JVM]
math: true
mermaid: true
---

Java 5 版本以后，JDK 有一个包叫做 instrument，可以实现一些非常酷的功能。一些 APM（Application Performance Management tools，最早是谷歌公开的论文提到的 [Google Dapper](http://bigbully.github.io/Dapper-translation)） 工具，就是通过它来增强的。比如 **Jrebel** 酷炫的热部署功能（该工具能够明显的增加开发效率）。



## Java Agent 

这些工具的基础，就是 Java Agent 技术，可以利用它来构建一个附加的代理程序，用来协助检测性能，还可以替换一些现有功能，甚至修改 JDK 的一些类，有点像 JVM 级别的 AOP 功能。



通常，Java 入口是一个 main 方法，而 Java Agent 的入口方法叫做 permain，表明实在 main 运行之前的一些操作。

Java Agent 是一个这样的 Jar 包，定义了一个标准的入口方法，它不需要继承或者实现任何其他的类，属于无侵入的一种开发模式。

> Java Agent 的入口方法为啥叫 permain，没有什么特别的理由，这个方法无论是第一次加载，还是每次加载新的 ClassLoader 加载，都会执行！！！

可以在这个前置的方法里，对字节码进行一些修改，来增加功能或者改变代码的行为，这种方法没有侵入性，只需要在启动命令上加上 -javaagent 参数就可以了。Java 6 以后，可以通过 attach 的方式，动态的给运行中的程序设置加载代理类。

其实，instrument 一共有两个 main 方法，一个是 premain，另一个是 agentmain，但在一个 JVM 中，只会调用一个；前者是 main 执行之前的修改，后者是控制类运行时的行为。它们区别是，agentmain 因为能够动态修改大部分代码，比较危险，限制会更大一些。



## 用处

### 获取统计信息

在很多的 APM 产品里，比如 Pinponit、SKyWalking 等，是使用 Java Agent 对代码进行的增强！

通过在方法执行前后动态加入的统计代码，来进行监控信息的收集；通过兼容 OpenTracing 协议，可以实现分布式链路的功能。

原理类似于 AOP，最终以字节码的形式存在，性能的损失取决于自实现的代码逻辑。

### 热部署

通过自定义的 ClassLoader ，可以实现代码的热替换。使用 agentmain ，实现热部署功能会更加方便，通过 agentmain 获取到 Instrumentaion 以后，就可以对类进行动态重定义了。

### 诊断

配合 JVMTI 技术，可以 attach 到某个进程进行运行时的统计和调试，比如流行的 btrace 和 arthas，就是用的这种技术。

## 练习

构建一个 agent 程序，大体的步骤如下：

1. 使用字节码增强工具，编写增强代码；
2. 在 mainfest 中指定 Permain-Class/Agent-Class 属性；
3. 使用参数加载或者使用 attacht 方式。

### 编写 Agent

Java Agent 最终的实现方式是一个 jar 包，使用 IDEA 创建一个默认的 Maven 工程就可以了。

创建一个普通的 Java 类，添加 permain 或者 agentmain 方法，二者的参数完全一样。

```java
package com.happymaya.javaagent;

import java.lang.instrument.Instrumentation;

public class AgentApp {
    public static void premain(String agentOps, Instrumentation inst) {
        System.out.println("==============enter premain==============");
        System.out.println(agentOps);
        inst.addTransformer(new Agent());
    }

    public static void agentmain(String agentOps, Instrumentation inst) {
        System.out.println("==============enter agentmain==============");
    }
}
```



### 编写 Transformer

实际的代码逻辑需要实现 ClassFileTransformer 接口。假如要统计某个方法的执行时间，使用 JavaAssist 工具来增强字节码，通过以下代码实现：

- 获取 MainRun 类的字节码实例；
- 获取 hello 方法的字节码实例；
- 在方法前后，加入时间统计，首先定义变量 _begin ，然后追加要实现的代码

由于借用的是 JavaAssist 完成字节码增强，因此不要忘记加入 Maven 依赖：

```xml
<dependency>
    <groupId>org.javassist</groupId>
    <artifactId>javassist</artifactId>
    <version>3.24.1-GA</version>
</dependency>
```

字节码增强也可以使用 Cglib、ASM 等其他工具！！！



### MANIFEST.MF 文件

编写的代码要让外界知晓，就是依靠 MANIFEST.MF 文件，具体路径在 `src/main/resources/META-INF/MANIFEST.MF`：

```makefile
Manifest-Version: 1.0
Can-Redefine-Classes: true
Can-Retransform-Classes: true
premain-class: com.happymaya.javaagent.AgentApp
agentmain-class: AgentApp
```

maven 打包会覆盖这个危机，因此需要为它指定一个：

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-jar-plugin</artifactId>
            <configuration>
                <archive>
                    <manifestFile>src/main/resources/META-INF/MANIFEST.MF</manifestFile>
                </archive>
            </configuration>
        </plugin>
    </plugins>
</build>
```

然后，在命令行，执行 mvn install 安装到本地代码库，或者使用 mvn deploy 发布到私服上！

MANIFEST.MF 完整参数清单：

```makefile
Premain-Class
Agent-Class
Boot-Class-Path
Can-Redefine-Classes
Can-Retransform-Classes
Can-Set-Native-Method-Prefix
```

### 使用

使用方式取决于使用的是 permain 还是 agentmain，它们之前的区别，如下：

#### premain

直接在启动命令行中加入参数即可，在 jvm 启动时启动代理：

```
java -javaagent:agent.jar MainRun
```

在 IDEA 中，将参数添加到 jvm options 里。

测试代码：

```java
package com.happymaya.javaagent.app;

import java.util.HashMap;

public class MainRun {
    public static void main(String[] args) {
        hello("world");
    }

    private static void hello(String name) {
        System.out.println("hello " + name + new HashMap<>() + new java.lang.String("fixme").replace("i","a"));
    }
}
```

执行后，直接输出 hello world。通过增强以后，还额外的输出了执行时间，以及一些 debug 信息。其中，debug 信息在 main 方法执行之前输出。



#### agentmain

这种模式用在一些诊断工具上。使用 `jdk/lib/tools.jar` 中的工具类，可以动态的为运行中的程序加入一些功能。它的主要运行步骤如下：

- 获取机器上运行的所有 JVM 进程 ID；
- 选择要诊断的 jvm；
- 将 jvm 使用 attach 函数链接上；
- 使用 loadAgent 函数加载 agent，动态修改字节码；
- 卸载 jvm。

代码如下：

```java
package com.happymaya.javaagent.app;


import com.sun.tools.attach.VirtualMachine;
import com.sun.tools.attach.VirtualMachineDescriptor;

import java.util.List;

public class JvmAttach {

    public static void main(String[] args)
            throws Exception {
        List<VirtualMachineDescriptor> list = VirtualMachine.list();
        for (VirtualMachineDescriptor vmd : list) {
            if (vmd.displayName().endsWith("MainRun")) {
                VirtualMachine virtualMachine = VirtualMachine.attach(vmd.id());
                virtualMachine.loadAgent("test.jar ", "...");
                //.....
                virtualMachine.detach();
            }
        }
    }
}
```

这些代码虽然强大，但都是比较危险的。这也是 Btrace 这么多年不能流行的原因。相对而言，Arthas 显得友好并且安全得多！！！



### 注意点：

jar 包依赖方式

- Agent 的 jar 包会以 fatjar 的方式提供，即将所有的依赖打包到一个大的 jar 包中。如果功能复杂、依赖多，这个 jar 包就会特别的大；
- 使用独立的 bom 文件维护这些依赖是另外一种方法。使用方自行管理依赖问题，但这通常会发生一些找不到 jar 包的错误，更糟糕的是，大多数在运行时才发现。

类名称重复

- 不要使用和 jdk 及 instrument 包中相同的类名（包括包名），虽然有时能够侥幸过关，但也会陷入无法控制的异常中。

做有限的功能

- 给系统动态的增加功能是非常酷的，但大多数情况下非常耗费性能。就像一些简单的诊断工具，会占用 1 核的 CPU，这是很平常的事情。

ClassLoader

- 如果用的 JVM 比较旧，频繁地生成大量的代理类，会造成元空间的膨胀，容易发生内存占用问题；
- ClassLoader 有双亲委派机制，如果想要替换相应的类，一定要搞清楚它的类加载器应该用哪个，否则替换的类，是不生效的；
- 具体的调试方法，可以在 Java 进程启动时，加入 -verbose:class 参数，用来监视引用程序对类的加载。

## Arthas

像 jstat 工具， jmap 等查看内存状态的工具 以及 jstack 这些工具，需要综合很多工具，对刚新手来说，很不友好。

Arthas 就是使用 Java Agent 技术编写的一个工具，具体采用的方式，就是 attach 方式，它会无侵入的 attach 到具体的执行进程上，方便进行问题分析。

可以像 debug 本地的 Java 代码一样，观测到方法执行的参数值，甚至做一些统计分析。这通常可以解决下面的问题：

- 哪个线程使用了最多的 CPU
- 运行中是否有死锁，是否有阻塞
- 如何监测一个方法哪里耗时最高
- 追加打印一些 debug 信息
- 监测 JVM 的实时运行状态

Arthas 官方文档十分详细，地址是：https://arthas.aliyun.com/doc/。

但无论工具如何强大，一些基础知识是需要牢固掌握的，否则，工具中出现的那些术语，也会让人一头雾水！！！！！

工具常变，基础更加重要。还是要多花点时间在原始的排查方法上。
