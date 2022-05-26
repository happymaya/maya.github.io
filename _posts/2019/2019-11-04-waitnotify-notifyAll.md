---
title:  使用 wait/notify/notifyAll 方法的注意事项
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2019-11-04 23:33:00 +0800
categories: [Java, Concurrent]
tags: [thread]
math: true
mermaid: true
---

从三个问题入手：
1. 为什么 wait 方法必须在 synchronized 保护的同步代码中使用？
2. 为什么 wait/notify/notifyAll 被定义在 Object 类中，而 sleep 定义在 Thread 类中？
3. wait/notify 和 sleep 方法的异同？

## 为什么 wait 必须在 synchronized 保护的同步代码中使用？

wait 方法的源码注释是怎么写的。
```java
    /**
     * Causes the current thread to wait until another thread invokes the
     * {@link java.lang.Object#notify()} method or the
     * {@link java.lang.Object#notifyAll()} method for this object.
     * In other words, this method behaves exactly as if it simply
     * performs the call {@code wait(0)}.
     * <p>
     * The current thread must own this object's monitor. The thread
     * releases ownership of this monitor and waits until another thread
     * notifies threads waiting on this object's monitor to wake up
     * either through a call to the {@code notify} method or the
     * {@code notifyAll} method. The thread then waits until it can
     * re-obtain ownership of the monitor and resumes execution.
     * <p>
     * As in the one argument version, interrupts and spurious wakeups are
     * possible, and this method should always be used in a loop:
     * <pre>
     *     synchronized (obj) {
     *         while (&lt;condition does not hold&gt;)
     *             obj.wait();
     *         ... // Perform action appropriate to condition
     *     }
     * </pre>
     * This method should only be called by a thread that is the owner
     * of this object's monitor. See the {@code notify} method for a
     * description of the ways in which a thread can become the owner of
     * a monitor.
     *
     * @throws  IllegalMonitorStateException  if the current thread is not
     *               the owner of the object's monitor.
     * @throws  InterruptedException if any thread interrupted the
     *             current thread before or while the current thread
     *             was waiting for a notification.  The <i>interrupted
     *             status</i> of the current thread is cleared when
     *             this exception is thrown.
     * @see        java.lang.Object#notify()
     * @see        java.lang.Object#notifyAll()
     */
    public final void wait() throws InterruptedException {
        wait(0);
    }
```

其中，`wait method should always be used in a loop...This method should only be called by a thread that is the owner of this object's monitor.`的意思是说，在使用 wait 方法时，必须把 wait 方法写在 synchronized 保护的 while 代码块中，并始终判断执行条件是否满足，如果满足就往下继续执行，如果不满足就执行 wait 方法，而在执行 wait 方法之前，必须先持有对象的 monitor 锁，也就是通常所说的 synchronized 锁。那么设计成这样有什么好处呢？

逆向思考这个问题，如果不要求 wait 方法放在 synchronized 保护的同步代码中使用，而是可以随意调用，那么就有可能写出如下的代码：

```java
public class BlockingQueue {
    Queue<String> buffer = new LinkedList<String>();

    public void give(String data) {
        buffer.add(data);
        notify();   // Since someone may be waiting in take
    }

    public String  take() throws InterruptedException {
        while (buffer.isEmpty()) {
            wait();
        }
        return buffer.remove();
    }

}
```

在代码中可以看到有两个方法，give 方法负责往 buffer 中添加数据，添加完之后执行 notify 方法来唤醒之前等待的线程，而 take 方法负责检查整个 buffer 是否为空，如果为空就进入等待，如果不为空就取出一个数据，这是典型的生产者消费者的思想。

但是这段代码并没有受 synchronized 保护，于是就有可能发生以下场景：

1. 首先，消费者线程调用 take 方法并判断 buffer.isEmpty 方法是否返回 true，若为 true 代表 buffer 是空的，则线程希望进入等待，但是在线程调用 wait 方法之前，就被调度器暂停了，所以此时还没来得及执行 wait 方法。
2. 此时生产者开始运行，执行了整个 give 方法，它往 buffer 中添加了数据，并执行了 notify 方法，但 notify 并没有任何效果，因为消费者线程的 wait 方法没来得及执行，所以没有线程在等待被唤醒。
3. 此时，刚才被调度器暂停的消费者线程回来继续执行 wait 方法并进入了等待。

虽然刚才消费者判断了 buffer.isEmpty 条件，但真正执行 wait 方法时，之前的 buffer.isEmpty 的结果已经过期了，不再符合最新的场景了，因为这里的“判断-执行”不是一个原子操作，它在中间被打断了，是线程不安全的。

假设这时没有更多的生产者进行生产，消费者便有可能陷入无穷无尽的等待，因为它错过了刚才 give 方法内的 notify 的唤醒。

看到正是因为 wait 方法所在的 take 方法没有被 synchronized 保护，所以它的 while 判断和 wait 方法无法构成原子操作，那么此时整个程序就很容易出错。

> 在没有被synchronized保护的场景，take方法，在调用 wait()方法之前被调度器暂停了，没有调用 wait( )方法的情况下执行的，如果没有被调度器暂停是可能正常运行一段时间的。
{: .prompt-tip }

于是把代码改写成源码注释所要求的被 synchronized 保护的同步代码块的形式，代码如下。

```java
public class BlockingQueue {

    Queue<String> buffer = new LinkedList<String>();

    public void give(String data) {
        buffer.add(data);
        notify();   // Since someone may be waiting in take
    }

    public String  take() throws InterruptedException {
        while (buffer.isEmpty()) {
            wait();
        }
        return buffer.remove();
    }

    public String  takeSync() throws InterruptedException {
        synchronized (this) {
            while (buffer.isEmpty()) {
                wait();
            }
            return buffer.remove();
        }
    }

}
```

这样就可以确保 notify 方法永远不会在 buffer.isEmpty 和 wait 方法之间被调用，提升了程序的安全性。

另外，wait 方法会释放 monitor 锁，这也要求我们必须首先进入到 synchronized 内持有这把锁。

> 这里还存在一个“虚假唤醒”（spurious wakeup，虚假唤醒特指没有实际唤醒动作，但是却被唤醒的情况）的问题，线程可能在既没有被notify/notifyAll，也没有被中断或者超时的情况下被唤醒，这种唤醒是不希望看到的<br/>。
> 虽然在实际生产中，虚假唤醒发生的概率很小，但是程序依然需要保证在发生虚假唤醒的时候的正确性，所以就需要采用 while 循环结构。
```java
while (condition does not hold)
    obj.wait();
```
> 这样即便被虚假唤醒了，也会再次检查while里面的条件，如果不满足条件，就会继续wait，也就消除了虚假唤醒的风险。
{: .prompt-danger }


## 为什么 wait/notify/notifyAll 被定义在 Object 类中，而 sleep 定义在 Thread 类中？

第二个问题，为什么 wait/notify/notifyAll 方法被定义在 Object 类中？而 sleep 方法定义在 Thread 类中？主要有两点原因：

1. 因为 Java 中每个对象都有一把称之为 monitor 监视器的锁，由于每个对象都可以上锁，这就要求在对象头中有一个用来保存锁信息的位置。这个锁是对象级别的，而非线程级别的，wait/notify/notifyAll 也都是锁级别的操作，它们的锁属于对象，所以把它们定义在 Object 类中是最合适，因为 Object 类是所有对象的父类。

2. 如果把 wait/notify/notifyAll 方法定义在 Thread 类中，会带来很大的局限性，比如一个线程可能持有多把锁，以便实现相互配合的复杂逻辑，假设此时 wait 方法定义在 Thread 类中，如何实现让一个线程持有多把锁呢？又如何明确线程等待的是哪把锁呢？既然我们是让当前线程去等待某个对象的锁，自然应该通过操作对象来实现，而不是操作线程。

   

### wait/notify 和 sleep 方法的异同？

主要对比 wait 和 sleep 方法，相同点：

1. 它们都可以让线程阻塞。
2. 它们都可以响应 interrupt 中断：在等待的过程中如果收到中断信号，都可以进行响应，并抛出 InterruptedException 异常。

但是它们也有很多的不同点：

1. wait 方法必须在 synchronized 保护的代码中使用，而 sleep 方法并没有这个要求。
2. 在同步代码中执行 sleep 方法时，并不会释放 monitor 锁，但执行 wait 方法时会**主动释放 monitor 锁**。
3. sleep 方法中会要求必须定义一个时间，时间到期后会主动恢复，而对于没有参数的 wait 方法而言，意味着永久等待，直到被中断或被唤醒才能恢复，它并不会主动恢复。
4. wait/notify 是 Object 类的方法，而 sleep 是 Thread 类的方法。

> 调用了 sleep()方法的线程，可以使用中断interrupt来提前唤醒线程。
{: .prompt-tip }

> 原子性: 描述动作要么被完整执行 要么都不执行的，完整执行并不意味这一口气做完，仍旧受线程调度机制影响原子操作: 不会受线程调度机制而中断，也就是不会出现线程上下文切换。即便是单核处理器，synchronized 执行期间也是可以进行线程切换的，受到线程调度机制的控制。
{: .prompt-tip }


> “但是在线程调用 wait 方法之前，就被调度器暂停了” 这个调度器是线程调度，由操作系统负责
{: .prompt-tip }