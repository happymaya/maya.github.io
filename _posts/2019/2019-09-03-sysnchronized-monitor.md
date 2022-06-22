---
title: synchronized 背后的“monitor 锁”
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2019-09-03 14:33:00 +0800
categories: [Java, Concurrent]
tags: [thread]
math: true
mermaid: true
---

# 获取和释放 monitor 锁的时机

最简单的同步方式就是利用 synchronized 关键字来修饰代码块或者修饰一个方法，那么这部分被保护的代码，在同一时刻就最多只有一个线程可以运行，而 synchronized 的背后正是利用 monitor 锁实现的。

所以首先看下获取和释放 monitor 锁的时机，每个 Java 对象都可以用作一个实现同步的锁，这个锁也被称为**内置锁或 monitor 锁**，获得 monitor 锁的唯一途径就是进入由这个锁保护的同步代码块或同步方法，线程在进入被 synchronized 保护的代码块之前，会自动获取锁，并且无论是正常路径退出，还是通过抛出异常退出，在退出的时候都会自动释放锁。

synchronized 修饰方法的代码的例子：

```java
public synchronized void method() {
    method body
}
```

我们看到 method() 方法是被 synchronized 修饰的，为了方便理解其背后的原理，我们把上面这段代码改写为下面这种等价形式的伪代码。

```java
public void method() {
    this.intrinsicLock.lock();
    try {
        method body
    } finally {
        this.intrinsicLock.unlock();
    }
}
```

在这种写法中，进入 method 方法后，立刻添加内置锁，并且用 try 代码块把方法保护起来，最后用 finally 释放这把锁，这里的 intrinsicLock 就是 monitor 锁。经过这样的伪代码展开之后，相信你对 synchronized 的理解就更加清晰了。

# 用 javap 命令查看反汇编的结果

JVM 实现 synchronized 方法和 synchronized 代码块的细节是不一样的，

## 同步代码块

同步代码块的实现，如代码所示。

```java
public class SynTest {
    public void synBlock() {
        synchronized (this) {
            System.out.println("happymaya");
        }
}
```

在 SynTest 类中的 synBlock 方法，包含一个同步代码块，synchronized 代码块中有一行代码打印了 happymaya 字符串，下面通过命令看下 synchronized 关键字到底做了什么事情：

- 首先用 cd 命令切换到 SynTest.java 类所在的路径，然后执行 javac SynTest.java，于是就会产生一个名为 SynTest.class 的字节码文件;
- 然后执行 `javap -verbose SynTest.class`，就可以看到对应的反汇编内容。

关键信息如下：

```shell
  public void synBlock();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=1
         0: aload_0
         1: dup
         2: astore_1
         3: monitorenter
         4: getstatic     #2    // Field java/lang/System.out:Ljava/io/PrintStream;
         7: ldc           #3    // String lagou
         9: invokevirtual #4    // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        12: aload_1
        13: monitorexit
        14: goto          22
        17: astore_2
        18: aload_1
        19: monitorexit
        20: aload_2
        21: athrow
        22: return
```



## 同步方法

从上面可以看出，同步代码块是使用 monitorenter 和 monitorexit 指令实现的。

对于 synchronized 方法，并不是依靠 monitorenter 和 monitorexit 指令实现的，被 javap 反汇编后可以看到，synchronized 方法和普通方法大部分是一样的，不同在于，这个方法会有一个叫作 **ACC_SYNCHRONIZED** 的 flag 修饰符，来表明它是同步方法。

同步方法的代码如下所示：

```java
public synchronized void synMethod() {}
```

对应的反汇编指令如下所示：

```java
public synchronized void synMethod();
	descriptor: ()V
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED
    Code:
		stack=0, locals=1, args_size=1
            0: return
        LineNumberTable:
			line 16: 0
```

可以看出，被 synchronized 修饰的方法会有一个 ACC_SYNCHRONIZED 标志。当某个线程要访问某个方法的时候，会首先检查方法是否有 ACC_SYNCHRONIZED 标志，如果有则需要先获得 monitor 锁，然后才能开始执行方法，方法执行之后再释放 monitor 锁。

其他方面， synchronized 方法和刚才的 synchronized 代码块是很类似的，例如这时如果其他线程来请求执行方法，也会因为无法获得 monitor 锁而被阻塞。



> monitor锁是操作系统层面的么 ?  Java的线程是映射到操作系统原生线程之上的，如果要阻塞或唤醒一个线程就需要操作系统的帮忙。



> “如果线程已经拥有了这个 monitor ，则它将重新进入，并且累加计数。”synchronize是不可重入锁，每次进入前不是需要先释放锁吗 - synchronized是可重入锁



> synchronized 修饰方法的时候，使用对象的monitor 锁是当前对象吗？ -  是的

