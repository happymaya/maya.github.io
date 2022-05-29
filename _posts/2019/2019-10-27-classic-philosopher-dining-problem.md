---
title: HashMap 为什么是线程不安全的
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2019-09-14 23:33:00 +0800
categories: [Java, Concurrent]
tags: [thread]
math: true
mermaid: true
---
# 第71讲：讲一讲经典的哲学家就餐问题

本课时我们介绍经典的哲学家就餐问题。

### 问题描述

哲学家就餐问题也被称为刀叉问题，或者吃面问题。我们先来描述一下这个问题所要说明的事情，这个问题如下图所示：

![](https://images.happymaya.cn/assert/java/thread/java-71-1.png)

有 5 个哲学家，他们面前都有一双筷子，即左手有一根筷子，右手有一根筷子。当然，这个问题有多个版本的描述，可以说是筷子，也可以说是一刀一叉，因为吃牛排的时候，需要刀和叉，缺一不可，也有说是用两把叉子来吃意大利面。这里具体是刀叉还是筷子并不重要，重要的是**必须要同时持有左右两边的两个才行**，也就是说，哲学家左手要拿到一根筷子，右手也要拿到一根筷子，在这种情况下哲学家才能吃饭。为了方便理解，我们选取和我国传统最贴近的筷子来说明这个问题。

为什么选择哲学家呢？因为哲学家的特点是喜欢思考，所以我们可以把哲学家一天的行为抽象为**思考，然后吃饭，并且他们吃饭的时候要用一双筷子，而不能只用一根筷子**。

**1. 主流程**

我们来看一下哲学家就餐的主流程。哲学家如果想吃饭，他会先尝试拿起左手的筷子，然后再尝试拿起右手的筷子，如果某一根筷子被别人使用了，他就得等待他人用完，用完之后他人自然会把筷子放回原位，接着他把筷子拿起来就可以吃了（不考虑卫生问题）。这就是哲学家就餐的最主要流程。

**2. 流程的伪代码**

我们来看一下这个流程的伪代码，如下所示：

```java
while(true) { 
    // 思考人生、宇宙、万物...
    think();
 
    // 思考后感到饿了，需要拿筷子开始吃饭
    pick_up_left_chopstick();
    pick_up_right_chopstick();
    eat();
    put_down_right_chopstick();
    put_down_left_chopstick();
    // 吃完饭后，继续思考人生、宇宙、万物...
}

```

while(true) 代表整个是一个无限循环。在每个循环中，哲学家首先会开始思考，思考一段时间之后（这个时间长度可以是随机的），他感到饿了，就准备开始吃饭。在吃饭之前必须**先拿到左手的筷子，再拿到右手的筷子，然后才开始吃饭；吃完之后，先放回右手的筷子，再放回左手的筷子**；由于这是个 while 循环，所以他就会继续思考人生，开启下一个循环。这就是整个过程。

### 有死锁和资源耗尽的风险

这里存在什么风险呢？就是发生死锁的风险。如下面的动画所示：

![](https://images.happymaya.cn/assert/java/thread/java-71-2.png)

根据我们的逻辑规定，在拿起左手边的筷子之后，下一步是去拿右手的筷子。大部分情况下，右边的哲学家正在思考，所以当前哲学家的右手边的筷子是空闲的，或者如果右边的哲学家正在吃饭，那么当前的哲学家就等右边的哲学家吃完饭并释放筷子，于是当前哲学家就能拿到了他右手边的筷子了。

但是，如果每个哲学家都同时拿起左手的筷子，那么就形成了环形依赖，在这种特殊的情况下，**每个人都拿着左手的筷子，都缺少右手的筷子，那么就没有人可以开始吃饭了**，自然也就没有人会放下手中的筷子。这就陷入了死锁，形成了一个相互等待的情况。代码如下所示：

```java
public class DiningPhilosophers {
    public static class Philosopher implements Runnable {
        private Object leftChopstick;
        private Object rightChopstick;

        public Philosopher(Object leftChopstick, Object rightChopstick) {
            this.leftChopstick = leftChopstick;
            this.rightChopstick = rightChopstick;
        }


        @Override
        public void run() {
            try {
                while (true) {
                    doAction("思考人生、宇宙、万物、灵魂...");
                    synchronized (leftChopstick) {
                        doAction("拿起左边的筷子");
                        synchronized (rightChopstick) {
                            doAction("拿起右边的筷子");
                            doAction("吃饭");
                            doAction("放下右边的筷子");
                        }
                        doAction("放下左边的筷子");
                    }
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }


        private void doAction(String action) throws InterruptedException {
            System.out.println(Thread.currentThread().getName() + " " + action);
            Thread.sleep((long) (Math.random() * 10));
        }
    }

    public static void main(String[] args) {
        Philosopher[] philosophers = new Philosopher[5];
        Object[] chopsticks = new Object[philosophers.length];
        for (int i = 0; i < chopsticks.length; i++) {
            chopsticks[i] = new Object();
        }
        for (int i = 0; i < philosophers.length; i++) {
            Object leftChopstick = chopsticks[i];
            Object rightChopstick = chopsticks[(i + 1) % chopsticks.length];
            philosophers[i] = new Philosopher(rightChopstick, leftChopstick);
            new Thread(philosophers[i], "哲学家" + (i + 1) + "号").start();
        }
    }
}
```

在这里最主要的变化是，我们实例化哲学家对象的时候，传入的参数原本都是先传入左边的筷子再传入右边的，但是当我们发现他是最后一个哲学家的时候，也就是 if (i == philosophers.length - 1) ，在这种情况下，我们给它传入的筷子顺序恰好相反，这样一来，他拿筷子的顺序也就相反了，**他会先首先拿起的是右手边的筷子，然后拿起的是左手边的筷子**。那么这个程序运行的结果，是所有哲学家都可以正常地去进行思考和就餐了，并且不会发生死锁。

### 总结

下面我们进行总结。在本课时，我们介绍了什么是哲学家就餐问题，并且发现了这其中蕴含着死锁的风险，同时用代码去演示了发生死锁的情况；之后给出了几种解决方案，比如死锁的检测与恢复、死锁避免，同时我们对于死锁避免的这种情况给出了代码示例。



> 哲学家面前不是一双筷子吧，是一根筷子吧？ ——  一共是一双，在左边有一根，在右边也有一根。