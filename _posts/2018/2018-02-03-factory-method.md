---
title: 工厂方法模式：解决生成对象时的不确定性
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2018-02-03 22:45:00 +0800
categories: [Design Pattern, creational]
tags:  [设计模式, Design Pattern, Factory Method, 工厂方法模式, 对象创建型模式]
math: true
mermaid: true
---

和抽象工厂模式很类似，但**工厂方法模式因为只围绕着一类接口来进行对象的创建与使用，使用场景更简单和单一，在实际的项目中使用频率反而比抽象工厂模式更高。**

## 原理

工厂方法模式的原始定义是：定义一个创建对象的接口，但让实现这个接口的类来决定实例化哪个类。

工厂方法模式的目的很简单，就是封装对象创建的过程，提升创建对象方法的可复用性。

工厂方法模式的 UML 图：

![工厂方法模式的 UML 图](https://images.happymaya.cn/assert/design-patterns/factory-uml.png)

工厂方法模式包含三个关键角色：
1. 抽象接口（也叫抽象产品）；
2. 核心工厂；
3. 具体产品（也可以是具体工厂）。

其中，核心工厂通常作为父类负责定义创建对象的抽象接口以及使用哪些具体产品，具体产品可以是一个具体的类，也可以是一个具体工厂类，负责生成具体的对象实例。于是，工厂方法模式便将对象的实例化操作延迟到了具体产品子类中去完成。

不同于抽象工厂模式，工厂方法模式侧重于直接对具体产品的实现进行封装和调用，通过统一的接口定义来约束程序的对外行为。换句话说，用户通过使用核心工厂来获得具体实例对象，再通过对象的统一接口来使用对象功能。

工厂方法模式对应 UML 图的代码实现如下：

```java
/* 抽象产品 */
public interface IProduct {
    void apply();
}

/* 核心工厂类 */
public class ProductFactory {
    public static IProduct getProduct(String name){
        if ("a".equals(name)) {
            return new ProductAImpl();
        }
        return new ProductBImpl();
    }
}

/* 具体产品实现 A */
public class ProductAImpl implements IProduct {
    @Override
    public void apply() {
        System.out.println("use A product now");
    }
}

/* 具体产品实现 B */
public class ProductBImpl implements IProduct{
    @Override
    public void apply() {
        System.out.println("use B product now");
    }
}

/* client 使用者 */
public class Client {
    public static void main(String[] args) {
        IProduct iProductb = ProductFactory.getProduct("");
        iProductb.apply();
        IProduct iProducta = ProductFactory.getProduct("a");
        iProducta.apply();
    }
}
```

总体来说，工厂方法模式是围绕着特定的抽象产品（一般是接口）来封装对象的创建过程，客户端只需要通过工厂类来创建对象并使用特定接口的功能。

## 使用场景

工厂方法模式有以下几个使用场景：

1. 需要使用很多重复代码创建对象时，比如，DAO 层的数据对象、API 层的 VO 对象等;
2. 创建对象要访问外部信息或资源时，比如，读取数据库字段，获取访问授权 token 信息，配置文件等；
3. 创建需要统一管理生命周期的对象时，比如，会话信息、用户网页浏览轨迹对象等；
4. 创建池化对象时，比如，连接池对象、线程池对象、日志对象等。这些对象的特性是：有限、可重用，使用工厂方法模式可以有效节约资源。
5. 希望隐藏对象的真实类型时，比如，不希望使用者知道对象的真实构造函数参数等。

一段经典的源码实现 —— MyBatis 实现的 Log 日志功能——来学习。代码如下：

```java
public final class LogFactory {
    public static final String MARKER = "MYBATIS";

    private static Constructor<? extends Log> logConstructor;
    
    private LogFactory() {}

    public static Log getLog(Class<?> clazz) {
        return getLog(clazz.getName());
    }

    public static Log getLog(String logger) {
        try {
            return (Log)logConstructor.newInstance(logger);
        } catch (Throwable var2) {
            throw new LogException("Error creating logger for logger " + logger + ".  Cause: " + var2, var2);
        }
    }
    ...省略具体工厂实现类...
    private static void tryImplementation(Runnable runnable) {
        if (logConstructor == null) {
            try {
                runnable.run();
            } catch (Throwable var2) {
            }
        }
    }

    private static void setImplementation(Class<? extends Log> implClass) {
        try {
            Constructor<? extends Log> candidate = implClass.getConstructor(String.class);
            Log log = (Log)candidate.newInstance(LogFactory.class.getName());
            if (log.isDebugEnabled()) {
                log.debug("Logging initialized using '" + implClass + "' adapter.");
            }
            logConstructor = candidate;
        } catch (Throwable var3) {
            throw new LogException("Error setting Log implementation.  Cause: " + var3, var3);
        }
    }
    static {
        tryImplementation(LogFactory::useSlf4jLogging);
        tryImplementation(LogFactory::useCommonsLogging);
        tryImplementation(LogFactory::useLog4J2Logging);
        tryImplementation(LogFactory::useLog4JLogging);
        tryImplementation(LogFactory::useJdkLogging);
        tryImplementation(LogFactory::useNoLogging);
    }
}
```

这段代码的实现很简单，却充分体现了**工厂方法模式使用场景的本质：尽可能地封装对象创建过程中所遇见的所有可能变化**。

这里 LogFactory 的职责就是核心工厂的创建职责，所需要创建的具体产品就是实现 Log 这个接口的特定实现，比如，Slf4j、Log4J 等。Log 接口代码如下所示：

```java
public interface Log {
    boolean isDebugEnabled();
    boolean isTraceEnabled();
    void error(String var1, Throwable var2);
    void error(String var1);
    void debug(String var1);
    void trace(String var1);
    void warn(String var1);
}
```

这些具体类的代码实现，各自的代码风格可能完全不同，但最终实现的日志功能却是一样的。其中，Slf4jImpl 甚至还为不同的 Slf4j 版本 API 接口做了兼容性处理，如果想要扩展一个新的日志实现，那么新增一个实现类并在核心工厂类里加入调用代码即可。

这里的关键其实是 Log 这个接口，这个接口设计得非常好，也就是抽象产品设计得特别好，不仅满足接口隔离原则，而且还是找到了正确抽象的典型代表。每一个操作几乎都是日志相关的原子操作，即便具体类的实现不同，但只要使用 LogFactory 就能获得满足要求的日志功能。

## 使用工厂模式的理由

使用工厂方法模式的原因，主要有以下三个：

1. **为了把对象的创建和使用过程分开，**降低代码耦合性。 这是使用工厂方法模式最直接的理由之一。在实际的软件开发中，你可能更喜欢使用 new 来创建对象，同时紧接着便开始使用新创建的对象，这看上去并没有什么问题，但是随着创建对象数量的增多，你会发现，当你想要重构、修改已有的对象属性和方法时，你几乎不敢轻易修改，因为你早已记不清哪些对象在哪里被创建和使用，以及跟哪些对象发生了关联和交互。而使用工厂方法模式，就能很好地避免这个问题，创建的过程始终在工厂内部管理，只要对外使用的方法不发生变化，那么就不会对创建对象造成影响；
2. **减少重复代码。** 对于要写代码的程序员或架构师来说，面对成千上万相同的数据对象进行增删改查时，如果每次都使用 new 来创建对象的话，那么 80% 的时间都会浪费在同样属性的 get 与 set 上。这时要是使用的对象之间还有相互引用的话（A 引用 B，B 又引用 C……），重复的代码就会剧增。而对于多个相同对象的构建过程，除了使用建造者模式以外，还可以使用工厂方法模式来避免出现过多的重复代码，将相同的创建规则统一放在一起
3. **统一管理创建对象的不同实现逻辑。** 比如，当一个业务对象发生业务逻辑变化时，使用工厂方法模式后，你不需要找到所有创建对象的地方去修改，而只需要在工厂里修改即可。即便这时你想要扩展对象为新的子类，也不需要把所有调用父类的地方都改成子类，只需要在工厂中修改其生产的对象为新的子类。同时，还隐藏了具体的创建过程，减少了使用者误用逻辑而导致未知错误出现的概率。

## 优缺点

优点：

1. **能根据用户的需求定制化地创建对象。** 工厂方法模式是基于某一个抽象产品角色来进行具体的实现工厂的设计。这样的好处就在于具体工厂可以根据自己的需求来决定创建什么样的具体产品，同时，还能把不同的算法细节完全封装在具体的工厂内部
2. **隐藏了具体使用哪种产品来创建对象。** 由于工厂方法模式对外使用统一的抽象接口，这样就向用户隐藏了具体正在使用的产品实例，让用户只需要关心抽象接口即可，无须关心创建细节，甚至都不用知道具体产品类的真实类名；
3. **实现同一抽象父类的多态性，满足“里氏替换原则（LSP）”。** 在使用工厂方法模式时，因为是围绕着统一的抽象接口来实现具体的功能，那么就能很便捷地使用不同的算法策略来实现同一功能，所以这样更好地实现了不同具体产品之间的可替换性；
4. **满足“开闭原则”。** 当你想要在系统中加入新的具体对象时，不用再修改抽象接口和核心工厂，也不用修改客户端，更不用修改其他具体工厂和具体产品，而只需要新增一个具体工厂和具体产品就可以了。这样系统的可扩展性也就变得非常好，完全符合“开闭原则”。

缺点：

1. **抽象接口新增方法时，会增加开发成本。** 当统一的抽象接口中新增方法时，相应的每个具体工厂都需要新增实现。不管具体工厂是否需要这个方法，都必须要新写代码，这样在一定程度上增加了开发工作量，因为修改后就需要编译、运行和测试，自然增加了开发成本。
2. **具体工厂实现逻辑不统一，增加代码理解难度。** 虽然核心工厂已经保证了部分共有逻辑的实现，但是具体产品依然是由具体工厂封装实现的，一旦具体工厂采用非通用的实现策略，那么对于维护的人员来说，就需要耗费大量的精力和时间去学习和理解。

**如果对象的属性数量并不多，并且创建过程也不复杂的话，那么用不着使用工厂方法模式来创建对象，毕竟工厂方法模式强调使用继承来实现对象的创建，会引入继承相关的副作用。**
