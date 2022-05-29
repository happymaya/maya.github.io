---
title:  线程的实现方式 - 本质上只有一种
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2019-02-01 23:33:00 +0800
categories: [Java, Concurrent]
tags: [thread]
math: true
mermaid: true
---


## 方式一：实现 Runbale 接口

```java
class  RunnableThread implements Runnable {

    @Override
    public void run() {
        System.out.println("使用实现 Runnable 接口实现线程");
    }

}
```
步骤：
1. 通过 RunnableThread 类实现 Runnable 接口；
2. 重写 run() 方法；
3. 将实现了 run() 方法的实例传到 Thread 类中就可以实现多线程。

## 方式二：继承 Thread
```java
class  ExtendsThread extends Thread {

    @Override
    public void run() {
        System.out.println("使用继承 Thread 接口实现线程");
    }

}
```
步骤：
1. 与第一种方式不同的是没有实现接口，而是继承 Thread 类；
2. 重写了其中的 run() 方法。


上面两种方式，在日常工作中经常使用。

## 方式三：通过线程池创建线程

线程池确实实现了多线程，比如：给线程池的线程数量设置成为 10 ，则会有 10 个线程来工作。

对线程池而言，本质上是通过**线程工厂**创建线程的，默认采用 `DefaultThreadFactory`.

`DefaultThreadFactory` 会给线程池创建的线程设置一些默认值，比如：<b>线程的名字</b>、<b>是否为守护线程</b>、<b>线程的优先级</b> 等，但是无论怎么设置这些熟悉，最终它（DefaultThreadFactory）还是通过 new Thread() 创建线程的，不同的是构造函数传入的参数要多一些。


`DefaultThreadFactory` 的源码如下（jdk 8）：
```java
/**
 * The default thread factory
 */
static class DefaultThreadFactory implements ThreadFactory {
    private static final AtomicInteger poolNumber = new AtomicInteger(1);
    private final ThreadGroup group;
    private final AtomicInteger threadNumber = new AtomicInteger(1);
    private final String namePrefix;

    DefaultThreadFactory() {
        SecurityManager s = System.getSecurityManager();
        group = (s != null) ? s.getThreadGroup() :
                              Thread.currentThread().getThreadGroup();
        namePrefix = "pool-" +
                      poolNumber.getAndIncrement() +
                     "-thread-";
    }

    public Thread newThread(Runnable r) {
        Thread t = new Thread(group, r,
                              namePrefix + threadNumber.getAndIncrement(),
                              0);
        if (t.isDaemon())
            t.setDaemon(false);
        if (t.getPriority() != Thread.NORM_PRIORITY)
            t.setPriority(Thread.NORM_PRIORITY);
        return t;
    }
}
```

## 方式四：有返回值的 Callable 创建线程

Runnable 创建线程是无返回值的，而 Callable 和与之相关的 Future、FutureTask，可以把线程执行的结果最为返回值返回

下面的栗子，实现了 Callable 接口，并且将它的泛型设置成 Integer ，然后它会返回一个随机数。

```java
static class CallableTask implements Callable<Integer> {

    public Integer call() throws Exception {
        return new Random().nextInt();
    }
}

```

无论是 Callable 还是 FutureTask ，它们首先和 Runnable 一样，都是一个任务，是需要执行的。而不是说它们本身就是线程，它们可以放到线程池中执行。

如下代码，submit() 方法把任务放到线程池中，并由线程池创建线程，最终都是靠线程来执行的，而子线程的创建方式仍然离不开<b>实现 Runnable </b> 和<b>继承 Thread 类</b>

```java

 */
void createThread() {
    // 创建线程池
    ExecutorService e =  Executors.newFixedThreadPool(10);
    // 提交任务，并用 Future 提交返回结果
    Future<Integer> future = e.submit(new CallableTask());
}
```


## 方式五：其他

### 定时器 Timer

定时器也可以实现线程。

如果新建一个 Timer，令其每个 10 秒或设置两个小时后，执行一些任务，那么这时它确实也创建了线程并执行了任务，但深入了解定时器的源码，本质上它还是一个继承自Thread 的类 TimerThread，因此定时器创建线程又绕回到最基本的两种方式。

TimerThread 源码如下（jdk 8）：
```java
/**
 * This "helper class" implements the timer's task execution thread, which
 * waits for tasks on the timer queue, executions them when they fire,
 * reschedules repeating tasks, and removes cancelled tasks and spent
 * non-repeating tasks from the queue.
 */
class TimerThread extends Thread {
 // 具体实现
}
```

### 其他方法

#### 匿名内部类
```java
/**
* 匿名内部类创建线程
* 用一个匿名内部类将需要传入的 new Runnable 给实例出来
*/
void innerClass() {
   new Thread(new Runnable() {
       public void run() {
           System.out.println(Thread.currentThread().getName());
       }
   }).start();
}
```
#### lamdba 表达式
```java
/**
 * lambda 表达式创建线程
 */
void lambdaThread() {
    new Thread(() -> System.out.println(Thread.currentThread().getName())).start();
}
```

像匿名内部类或 lambda 表达式这些创建线程，都仅仅是在语法层面上，不能将它们归结于实现多线程的方式。


## 总结

### 实现线程只有一种方式

其他的创建方式，比如**线程池**或是**定时器**，仅仅是在 new Thread() 外做了一层封装，如果把这些都叫作一种新的方式，那么创建线程的方式便会千变万化、层出不穷。

比如 JDK 更新了，它可能会多出几个类，会把 new Thread() 重新封装，表面上看又会是一种新的实现线程的方式，透过现象看本质，打开封装后，会发现它们最终都是**基于 Runnable 接口**或**继承 Thread 类实现的**。

**基于 Runnable 接口**和**继承 Thread 类实现的** 是一样的理由是：
1. 启动线程需要调用 start() 方法，而 start() 方法最终还会调用 run() 方法；
2. 基于 Runable 接口 run() 方法的实现如下：
   ```java
   
    /**
     * If this thread was constructed using a separate 
     * <code>Runnable</code> run object, then that
     * <code>Runnable</code> object's <code>run</code> method is called;
     * otherwise, this method does nothing and returns.
     * <p>
     * Subclasses of <code>Thread</code> should override this method.
     *
     * @see     #start()
     * @see     #stop()
     * @see     #Thread(ThreadGroup, Runnable, String)
     */
    @Override
    public void run() {
        if (target != null) { // 判断 target 是否等于 null
            target.run();     // 如果不等于 null，执行 target.run()
        }
    }
   ```
   run() 方法中的 target 实际上就是一个 Runable，，即使用 Runnable 接口实现线程并传给 Thread 类的对象。
3. 继承 Thread 类实现。继承 Thread 类之后，会把上述的 run() 方法重写，重写后 run() 方法里直接就是所需要执行的任务，但它最终还是需要调用 thread.start() 方法来启动线程，而 start() 方法最终也会调用这个已经被重写的 run() 方法来执行它的任务

由此，创建线程只有一种方式，就是构造一个 Thread 类，这是创建线程的唯一方式。

可以得出结论：本质上，实现线程只有一种方式，而要想实现线程执行的内容，却有两种方式， **实现 Runnable 接口的方式**或是**继承 Thread 类重写 run() 方法的方式**，把想要执行的代码传入，让线程去执行，在此基础上，如果还想有更多实现线程的方式，比如**线程池**和 **Timer 定时器**只需要在此基础上进行封装即可。

### 实现 Runnable 接口比继承 Thread 类实现线程要好处
有三点:
1. 从代码的架构考虑，Runnable 里只有一个 run() 方法，它定义了需要执行的内容，在这种情况下，实现了 Runnable 与 Thread 类的解耦，Thread 类负责线程启动和属性设置等内容，权责分明；
2. 某些情况下可以提高性能
   1. 如果使用继承 Thread 类方式，每次执行一次任务，都需要新建一个独立的线程，执行完任务后线程走到生命周期的尽头被销毁，如果还想执行这个任务，就必须再新建一个继承了 Thread 类的类，如果此时执行的内容比较少，那么它所带来的开销并不大，相比于整个线程从开始创建到执行完毕被销毁，这一系列的操作比 run() 方法打印文字本身带来的开销要大得多，相当于捡了芝麻丢了西瓜，得不偿失；
   2. 如果使用实现 Runnable 接口的方式，就可以把任务直接传入线程池，使用一些固定的线程来完成任务，不需要每次新建销毁线程，大大降低了性能开销。 
3. Java 语言不支持双继承，如果类继承了 Thread 类，那么后续就没有办法再继承其他的类，这样一来，如果未来这个类需要继承其他类实现一些功能上的拓展，它就没有办法做到了，相当于限制了代码未来的可拓展性。 
4. shi

由此，应该优先选择通过实现 Runnable 接口的方式来创建线程（实现 runnable 可以说明使用了**组合的方式**要比**继承的方式**要好）。