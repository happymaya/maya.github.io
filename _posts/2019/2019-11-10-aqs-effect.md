---
title: AQS 的作用和重要性
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2019-11-10 23:23:00 +0800
categories: [Java, Concurrent]
tags: [thread]
math: true
mermaid: true
---
# AQS 的重要性

![Subclass of AbstractQueuedSynchronizer](https://images.happymaya.cn/assert/java/thread/java-thread-aqs-class.png)

如上图所示，AQS 在 **ReentrantLock、ReentrantReadWriteLock、Semaphore、CountDownLatch、ThreadPoolExcutor 的 Worker** 中都有运用（JDK 1.8），AQS 是这些类的底层原理。

以上这些类，大多数经常使用的类，可见 JUC 包里很多重要的工具类背后都离不开 AQS 框架，因此 AQS 的重要性不言而喻。

<!-- # 我学习 AQS 的思路

AQS 类的内部结构要比一般的类**复杂得多**，里面有很多细节，不容易完全掌握，所以上来就直接看源码，容易把自己给绕晕。

在实际开发中，大多数是面向业务，平时并不需要自己来开发类似于 ReentrantLock 这样的工具类。

因此**不会直接使用到 AQS 来进行开发**，因为 JDK 已经提供了很多封装好的线程协作工具类。

像 ReentrantLock、Semaphore 就是 JDK 提供给我们的，其内部就用到了 AQS，而这些工具类已经基本**足够覆盖大部分的业务场景**了，这就使得即便不了解 AQS，也能利用这些工具类顺利进行开发。

因此学习 AQS 的目的主要是想理解其背后的**原理**、**设计思想**，以及**提高技术**并**应对面试**。 -->

# 锁和协作类有共同点：阀门功能

从熟悉的类作为学习 AQS 的切入点。

ReentrantLock 和 Semaphore，二者之间的共同点是：当作一个阀门来使用

比如：把 Semaphore 的许可证数量设置为 1，那么由于它只有一个许可证，所以只能允许一个线程通过，并且当之前的线程归还许可证后，会允许其他线程继续获得许可证。这点和 ReentrantLock 很像，只有一个线程能获得锁，并且当这个线程释放锁之后，会允许其他的线程获得锁。如果线程发现当前没有额外的许可证时，或者当前得不到锁，那么线程就会被阻塞，并且等到后续有许可证或者锁释放出来后，被唤醒，所以这些环节都是比较类似的。

除了 ReentrantLock 和 Semaphore 之外，CountDownLatch、ReentrantReadWriteLock 等工具类都有类似的让**线程“协作”的功能**，它们背后都是利用 AQS 来实现的。

# 为什么需要 AQS

上面刚讲的那些协作类，它们有很多工作是类似的，所以如果能把实现类似工作的代码给提取出来，变成一个新的底层工具类（或称为框架）的话，就可以直接使用这个工具类来构建上层代码了，而这个工具类其实就是 AQS。

有了 AQS 之后，对于 ReentrantLock 和 Semaphore 等线程协作工具类而言，它们就不需要关心这么多的**线程调度**细节，只需要实现它们各自的设计逻辑即可。

# 如果没有 AQS

如果没有 AQS，就需要每个线程协作工具类自己去实现至少以下内容，包括：
- **状态的原子性管理**
- **线程的阻塞与解除阻塞**
- **队列的管理**

这里的状态对于不同的工具类而言，代表不同的含义，比如对于 ReentrantLock 而言，它需要维护**锁被重入的次数**，但是保存重入次数的变量是会被多线程同时操作的，就需要进行处理，以便保证线程安全。不仅如此，对于那些未抢到锁的线程，还应该让它们陷入阻塞，并进行排队，并在合适的时机唤醒。所以说这些内容其实是比较繁琐的，而且也是比较重复的，而这些工作目前都由 AQS 来承担了。

如果没有 AQS，就需要 ReentrantLock 等类来自己实现相关的逻辑，但是让每个线程协作工具类自己去正确并且高效地实现这些内容，是相当有难度的。AQS 可以帮我们把 “脏活累活” 都搞定，所以对于 ReentrantLock 和 Semaphore 等类而言，它们只需要关注自己特有的业务逻辑即可。正所谓是“哪有什么岁月静好，不过是有人替你负重前行”。


# AQS 的作用

**AQS 是一个用于构建锁、同步器等线程协作工具类的框架**。

有了 AQS 以后，很多用于线程协作的工具类就都可以很方便的被写出来；
有了 AQS 之后，可以让更上层的开发极大的减少工作量，避免重复造轮子，同时也避免了上层因处理不当而导致的线程安全问题，因为 AQS 把这些事情都做好了。
总之，有了 AQS 之后，构建线程协作工具类就容易多了。