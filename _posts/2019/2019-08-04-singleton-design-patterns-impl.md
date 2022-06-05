---
title:  单例设计模式的设计实现
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2019-03-15 23:33:00 +0800
categories: [Architecture Design, Design Patterns]
tags:  [设计模式, 抽象工厂模式 , Design Patterns, Abstract Factory]
math: true
mermaid: true
---

单例设计模式时 GoF23 种设计模式中最常用的设计模式之一，无论是三方库，还是日常开发中，几乎都可以看到单例的影子。

单例设计模式提供了一种在多线程情况下保证实例唯一性的解决方案。

虽然单例设计模式的实现非常简单，但实现方式却多种多样，

从三个维度对其进行评估：线程安全、高性能、懒加载。



## 饿汉式

```java
/* final 不允许被继承 */
public final class HungryManSingleton {

    /* 实例变量 */
    private byte[] data = new byte[1024];

    /* 在定义实例对象的时候直接初始化 */
    private static HungryManSingleton instance = new HungryManSingleton();

    /* 私有构造方法，不允许外部 new */
    private HungryManSingleton(){}

    private static HungryManSingleton getInstance() {
        return instance;
    }

}
```

关键点：instance 作为类变量，直接得到了初始化。

如果主动使用 Singleton 类，那么 instance 实例将会直接完成创建，包括其中的实例变量都会得到初始化，例如 1K 空间的 data 将会同时被创建。

instance 作为类变量在类初始化的过程中，会被收集进 `<clinit>()` 方法中，该方法能够百分之百保证同步，也就是说在多线程的情况下可能被实例化两次，但是 instance 被 ClassLoader 加载后可能很长一段时间才被使用，意味着 instance 实例所开辟的堆内存会驻留更久时间。

倘若类中的成员属性比较少，并且占用内存资源不多，饿汉式也未尝不可，相反，倘若类中的成员都是比较重的资源，那么这种方式就不太妥。

总之，饿汉式的单例模式可以保证多线程下的唯一实例，getInstance 方法性能也比较高，但是无法进行懒加载。

## 懒汉式

```java
/* final 不允许被继承 */
public class LazyManSingleton {

    /* 实例变量 */
    private byte[] data = new byte[1024];

    /* 在定义实例对象的时候直接初始化 */
    private static LazyManSingleton instance = null;

    /* 私有构造方法，不允许外部 new */
    private LazyManSingleton(){}

    private static LazyManSingleton getInstance() {
        if (null == instance) {
            instance = new LazyManSingleton();
        }
        return instance;
    }

}
```

关键点：在使用类实例的时候，再去创建（用时创建），这样可以避免类在初始化时提前创建。

Singleton 的类变量 instance = null，因此当 Singleton.class 被初始化的时候 instance 并不会被实例化；

在 getInstance 方法，会先判断 instance 实例是否被实例化，看起来没有什么问题，但是将 getInstance 方法放在多线程环境下进行分析，则会导致 instance 被实例化一次以上，不能保证单例的唯一性，如下图。

![懒汉式设计模式多线程](http://processon.com/chart_image/629cf1bbe0b34d3bc3b267b0.png)

总之，懒汉式的单例模式可以保证实例的懒加载，但无法保证实例的唯一性。

## 懒汉式 + 数据同步方法

在多线程的情况下，instance 成为共享资源（数据），当多个线程对其访问使用时，需要保证数据的同步性，只要对懒汉式的代码上稍加修改，增加同步的月约束就可以，修改后的代码如下：

```java
/* final 不允许被继承 */
public class LazyManSyncSingleton {

    /* 实例变量 */
    private byte[] data = new byte[1024];

    /* 在定义实例对象的时候直接初始化 */
    private static LazyManSyncSingleton instance = null;

    /* 私有构造方法，不允许外部 new */
    private LazyManSyncSingleton(){}

    /* 向 getInstance 方法加入同步控制，每次只能有一个线程能够进入 */
    private static synchronized LazyManSyncSingleton getInstance() {
        if (null == instance) {
            instance = new LazyManSyncSingleton();
        }
        return instance;
    }
}
```

懒汉式 + 数据同步的方式，既满足懒加载，又能够百分之百地保证 instance 实例的唯一性，又由于 synchronized 关键字天生的排他性，导致 getInstance() 方法只能在同一时刻被一个线程所访问，性能低下。

## Double-Check

```java
/* final 不允许被继承 */
public class DoubleCheckSingleton {

    /* 实例变量 */
    private byte[] data = new byte[1024];

    /* 在定义实例对象的时候直接初始化 */
    private static DoubleCheckSingleton instance = null;

    Connection connection;

    Socket socket;

    /* 私有构造方法，不允许外部 new */
    private DoubleCheckSingleton() {
        this.connection = null;    // 初始化 connection
        this.socket = null;        // 初始化 socket
    }

    private static DoubleCheckSingleton getInstance() {
        // 当 instance 为 null 时，进入同步代码块
        // 同时该判断避免了每次都需要进入同步代码块，可以提高些效率
        if (null == instance) {
            // 只有一个线程能够获取 DoubleCheckSingleton.class 关联的 monitor
            synchronized (DoubleCheckSingleton.class) {
                // 判断如果 instance 为 null ，则创建
                if (null == instance) {
                    instance = new DoubleCheckSingleton();
                }
            }
        }
        return instance;
    }
}

```

Double-Check ，提供了一种高效的数据同步策略，就是首次初始化时加锁，之后允许多个线程同时进行 getInstance 方法的调用来获得类的实例。



当两个线程发现 null=instance 成立时，只有一个线程有资格进人同步代码块，完成对 instance 的实例化；

随后的线程发现 null=instance 不成立则无须进行任何动作，以后对 getInstance 的访问就不需要数据同步的保护了。

这种方式看起来是那么的完美和巧妙，既满足了懒加载，又保证了 instance 实例的唯一性，Double-Check 的方式提供了高效的数据同步策略，可以允许多个线程同时对 getInstance 进行访问，但是这种方式在多线程的情况下有可能会引起空指针异常，原因是：在 Singleton 的构造函数中，需要分别实例化 conn 和 socket 两个资源，还有 Singleton 自身，根据 JVM 运行时指令重排序和 Happens--Before 规则，这三者之间的实例化顺序并无前后关系的约束，那么极有可能是 instance 最先被实例化，而 conn 和 socket 并未完成实例化，未完成初始化的实例调用其方法将会抛出空指针异常，如下：

![](http://processon.com/chart_image/629cf8586376895abf210648.png)

## Volatile + Double-Check

Double-Check 还有可能引起类成员变量的实例化 conn 和 socket 发生在 instance 实例化之后，这一切都是由 JVM 在运行时指令重排序所导致的。

volatile 关键字则可以防止这种重排序的发生，因此稍作修改就可以满足多线程下的单例：

```java
/* final 不允许被继承 */
public class VolatileDoubleCheckSingleton {

    /* 实例变量 */
    private byte[] data = new byte[1024];

    /* 在定义实例对象的时候直接初始化 */
    private volatile static VolatileDoubleCheckSingleton instance = null;

    Connection connection;

    Socket socket;

    /* 私有构造方法，不允许外部 new */
    private VolatileDoubleCheckSingleton() {
        this.connection = null;    // 初始化 connection
        this.socket = null;        // 初始化 socket
    }

    private static VolatileDoubleCheckSingleton getInstance() {
        // 当 instance 为 null 时，进入同步代码块
        // 同时该判断避免了每次都需要进入同步代码块，可以提高些效率
        if (null == instance) {
            // 只有一个线程能够获取 DoubleCheckSingleton.class 关联的 monitor
            synchronized (DoubleCheckSingleton.class) {
                // 判断如果 instance 为 null ，则创建
                if (null == instance) {
                    instance = new VolatileDoubleCheckSingleton();
                }
            }
        }
        return instance;
    }
}
```

 

## Holder 方式

关键点： 借助了类加载的特点。

```java
/* final 不允许被继承 */
public class HolderSingleton {

    /* 实例变量 */
    private byte[] data = new byte[1024];

    /* 私有构造方法，不允许外部 new */
    private HolderSingleton(){}

    /* 在静态内部类中持有 Singleton 的实例，并且可能直接初始化 */
    private static class Holder {
        private static HolderSingleton instance = new HolderSingleton();
    }

    /* 调用 getInstance 方法，事实上是获得 Holder 的 instance 静态属性 */
    public static HolderSingleton getInstance() {
        return Holder.instance;
    }
}
```

在 HolderSingleton 类中，没有 instance 的静态成员，而是将其放到了静态内部类 Holder 之中，因此在HolderSingleton 类的初始化过程中并不会创建 Singleton 的实例。

Holder 类中定义了 HolderSingleton 的静态变量，并且直接进行了实例化，当 Holder 被主动引用的时候则会创建 HolderSingleton 的实例，Singleton 实例的创建过程在 Java 程序编译时期收集至`<clinit>O`方法中，该方法又是同步方法，因此可以保证内存的可见性、JVM 指令的顺序性和原子性。

Holder方式的单例设计是最好的设计之一，也是目前使用比较广的设计之一。

## 枚举方式

使用枚举的方式实现单例模式是《Effective Java》中力推的方式，在很多优秀的开源代码中经常可以看到使用枚举方式实现单例模式的（身影)。

枚举类型不允许被继承，同样是线程安全的且只能被实例化一次，但是枚举类型不能够懒加载，对 Singleton 主动使用，比如调用其中的静态方法则 INSTANCE 会立即得到实例化.

```java
/* 枚举类型本身是 final 的，不允许被继承 */
public enum EnumSingleton {

    INSTANCE;

    // 实例变量
    private byte[] data = new byte[1024];


    EnumSingleton(){
        System.out.println("INSTANCE will be initialized immediately...");
    }

    public static void method() {
        // 调用该方法，则会主动使用 Singleton，INSTANCE 将会被实例化
    }

    public static EnumSingleton getInstance() {
        return INSTANCE;
    }

}
```

在对它进行改造，增加懒加载的特性，类似于 Holder  的方式，改进后的代码如下：

```java
/* 枚举类型本身是 final 的，不允许被继承 */
public class EnumLazySingleton {

    /* 实例变量 */
    private byte[] data = new byte[1024];

    private EnumLazySingleton(){}

    /* 使用枚举充当 holder */
    private enum EnumHolder{
        INSTANCE;

        private EnumLazySingleton instance;

        EnumHolder() {
            this.instance = new EnumLazySingleton();
        }

        private EnumLazySingleton getEnumLazySingleton() {
            return instance;
        }
    }

    public static EnumLazySingleton getInstance() {
        return EnumHolder.INSTANCE.getEnumLazySingleton();
    }
}

```

## 总结

虽然单例设计模式简单，但在多线程的情况下，设计单例程序未必就能满足单实例、懒加载以及高性能。优先 Holder 和枚举方式来设计单例。