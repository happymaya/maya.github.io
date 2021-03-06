---
title: 原型模式：对象拷贝
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2018-02-04 22:24:55 +0800
categories: [Design Pattern, creational]
tags:  [设计模式, Design Pattern, Prototype, 原型模式, 对象创建型模式]
math: true
mermaid: true
---

原型模式最早出现于 1963 年的一个叫 Sketchpad 的系统中，说起 Sketchpad 或许并不熟悉，但是说起 CAD（计算机辅助设计），现在在工程设计领域几乎无人不知，其实 Sketchpad 就被认为是现代 CAD 程序的鼻祖，主要思想是**拥有一张可以实例化成许多副本的原图，如果用户更改了主工程图，则所有实例也会更改**。这便是**原型模式最初的思维模型**。

不过在面向对象编程中，对象的原型在变化时通常不会影响新实例对象。实际上，原型模式不仅在 Java、C++ 等主要基于类的编程语言中有广泛应用，而且还在一开始就是基于原型的 JavaScript 等编程语言中得到了发扬光大。

原型模式的原理与使用其实很简单，但是要想用好它，除了要了解它的优点以外，更要重点注意它带来的问题以及掌握如何规避这些问题。

# 模式原理
定义是：**使用原型实例指定创建对象的种类，然后通过拷贝这些原型来创建新的对象。**

定义中清晰地指出了两个关键点:
1. **建立原型**;
2. **基于原型做拷贝**。

原理很简单，在现代编程中，经常会用到的 Ctrl+C 加 Ctrl+V 编程，可以说就是最直接的原型模式的实践之一。

原型模式的 UML 图：
![](https://images.happymaya.cn/assert/design-patterns/prototype.jpg)

从这个 UML 图中，可以看到原型模式中的关键角色有三个：
1. 使用者；
2. 原型；
3. 新实例。

**使用者需要建立一个原型，才能基于原型拷贝出新实例**。除此之外，使用者还需要决策什么时候使用原型、什么时候使用新实例，以及从原型到新实例之间的拷贝应该采用什么样的算法策略，这些都是使用者来进行控制和决定的。只不过通常会使用一个通用的抽象拷贝接口来对外提供拷贝。

UML 图对应的代码该实现呢可参考下面这段基于 Java Cloneable 接口的代码实现：
```java
public interface PrototypeInterface extends Cloneable {
    PrototypeInterface clone() throws  CloneNotSupportedException;
}

public class ProtypeA implements PrototypeInterface{
    @Override
    public ProtypeA clone() throws CloneNotSupportedException {
        System.out.println("Cloning new objecct: A");
        return (ProtypeA)super.clone();
    }
}

public class ProtypeB implements PrototypeInterface{
    @Override
    public ProtypeB clone() throws CloneNotSupportedException {
        System.out.println("Cloning new objecct: B");
        return (ProtypeB)super.clone();
    }
}

public class Demo {
    public static void main(String[] args) throws CloneNotSupportedException {
        ProtypeA source = new ProtypeA();
        System.out.println(source);

        ProtypeA newInstanceA = source.clone();
        System.out.println(newInstanceA);
    }
}

```
代码中：

- 定义的 PrototypeInterface 接口通过继承 Cloneable 接口并重写 clone() 方法来实现对象的拷贝；
- ProtypeA 和 ProtypeB 都可以在建立自己的原型对象后，调用 clone() 方法来创建新对象；
- 需要注意的是，Cloneable 接口本身是空方法，调用的 clone() 方法其实是 Object.clone() 方法。

从以上内容会发现，原型模式封装了如下变化：

- 原始对象的构造方式；
- 对象的属性与其他对象间的依赖关系；
- 对象运行时状态的获取方式；
- 对象拷贝的具体实现策略。

所以说，**原型模式从建立原型到拷贝原型生成新实例，都是对用户透明的，一旦中间任何一个小细节出现问题，你可能获取的就是一个错误的对象**。
# 使用场景
原型模式常见的使用场景有以下六种：
1. **资源优化场景**。当进行对象初始化需要使用很多外部资源时，比如，IO 资源、数据文件、CPU、网络和内存等；
2. **复杂的依赖场景。** 比如，F 对象的创建依赖 A，A 又依赖 B，B 又依赖 C……于是创建过程是一连串对象的 get 和 set；
3. **性能和安全要求的场景。** 比如，同一个用户在一个会话周期里，可能会反复登录平台或使用某些受限的功能，每一次访问请求都会访问授权服务器进行授权，但如果每次都通过 new 产生一个对象会非常烦琐，这时则可以使用原型模式；
4. **同一个对象可能被多个修改者使用的场景。** 比如，一个商品对象需要提供给物流、会员、订单等多个服务访问，而且各个调用者可能都需要修改其值时，就可以考虑使用原型模式；
5. **需要保存原始对象状态的场景。** 比如，记录历史操作的场景中，就可以通过原型模式快速保存记录；
6. **结合工厂模式来使用。** 在实际项目中，原型模式除了单独基于对象使用外，还可以结合工厂方法模式一起使用，通过定义统一的复制接口，比如 clone、copy。使用一个工厂来统一进行拷贝和新对象创建， 然后由工厂方法提供给调用者使用。

在实际的一些类库和组件中都有原型模式的应用，比如：
- Spring 中使用`@Scope("prototype")`注解来使得注入的 Java Bean 使用原型模式；
- Fastjson 中的 `JSON.parseObject()` 也是一种原型模式的实践;
- JDK 中，使用 cloneable 接口的都能实现原型模式。

接下来还是通过一个具体的实例理解原型模式的使用。假设正在构建一个家庭的知识管理系统，系统会很频繁地使用电子书和电影类的实例对象，但不想每次创建对象时都等待很长的时间（像一部电影的大小通常都在 1GB 以上），于是决定使用原型模式来快速拷贝创建对象。
首先，创建一个继承接口 Cloneable 的接口 IPrototype，如下所示：
```java
public interface IPrototype extends Cloneable{

    // 继承 Cloneable 接口，重写 clone() 方法，以便能使用父类的 Object.clone() 负值方法
    // 也可以直接实现 Cloneable 接口，效果是一样的；
    // 为了统一业务接口层级，子类都要实现 IPrototype
    IPrototype clone() throws CloneNotSupportedException;
}

```
 然后，再分别让电影类 Movie 和电子书类 EBook 实现 IPrototype 接口的拷贝方法。  
```java
public class Movie implements IPrototype{

    /**
     * 打印并拷贝对象
     * @return movice object
     * @throws CloneNotSupportedException
     */
    @Override
    public Movie clone() throws CloneNotSupportedException {
        System.out.println("Cloning Movie object");
        return (Movie)super.clone();
    }

    /**
     * 方便结果展示
     * @return String
     */
    @Override
    public String toString() {
        return "Movice: {}";
    }
}

public class EBook implements IPrototype{

    /**
     * 打印并拷贝对象
     * @return EBook object
     * @throws CloneNotSupportedException
     */
    @Override
    public EBook clone() throws CloneNotSupportedException {
        System.out.println("Cloning Book object...");
        return (EBook) super.clone();
    }

    /**
     * 方便结果展示
     * @return String
     */
    @Override
    public String toString() {
        return "EBook: {}";
    }
}

```
 接下来，使用工厂模式来根据不同的对象类型进行对象的拷贝创建。  
```java
public enum ModeType {

    MOVICE("movice"),
    EBOOK("eBook");

    private String name;

    ModeType(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }
}

public class PrototypeFactory {

    /**
     * 充当注册表的作用，用于存放原始对象，作为对象拷贝的基础
     */
    private static Map<String, IPrototype> prototypes = new HashMap<>();


    /**
     * 初始化时，就将原始对象放入注册表中
     */
    static {
        prototypes.put(ModeType.MOVICE.getName(), new Movie());
        prototypes.put(ModeType.EBOOK.getName(), new EBook());
    }

    /**
     * 获取对象，使用 names 来进行对象拷贝
     */
    public static IPrototype getInstance(final String s) throws CloneNotSupportedException {
        return prototypes.get(s).clone();
    }
}



```
 准备完毕后，写一段简单的单元测试来调用原型工厂创建对象。  
```java
public class Clinet {

    public static void main(String[] args) {
        try {
            String moviePrototype = PrototypeFactory.getInstance(ModeType.MOVICE.getName()).toString();
            System.out.println(moviePrototype);

            String eBookPrototype = PrototypeFactory.getInstance(ModeType.EBOOK.getName()).toString();
            System.out.println(eBookPrototype);
        } catch (CloneNotSupportedException e) {
            e.printStackTrace();
        }
    }
}

// 输出结果如下：
// Cloning Movie object
// Movice: {}
// Cloning Book object...
// EBook: {}
```
在上面的代码实现中，我们通过实现 Cloneable 接口并使用 clone() 方法来进行对象的拷贝。电影类 Movie 和电子书 EBook 分别实现了各自的拷贝逻辑，当通过原型工厂 PrototypeFactory 获取指定类型的对象时，我们其实获得的对象就是原始电影类或电子书类的对象副本。

综合以上分析，你会发现，原型模式的适用场景通常都是对已有的复杂对象或大型对象的创建上，在这样的场景中，创建对象通常是一件非常烦琐的事情，通过拷贝对象能快速地创建对象。

其实这里还涉及一个扩展知识点：浅拷贝与深拷贝。当我们在做对象拷贝时，需要在浅拷贝和深拷贝之间做取舍。如果类仅包含原始字段和不可变字段，可以使用浅拷贝；如果类还包含有可变字段的引用（比如，对象中包含对象），那么我们就应该使用深拷贝。关于浅拷贝与深拷贝的话题这里我就不做更多的展开讲述了，你若感兴趣的话可以自行学习和研究。

## 使用原型模式的原因
使用原型的模式原因主要有以下四个：
1. **减少每次创建对象的资源消耗。**当类初始化消耗资源特别多时，原型模式特别有用。比如，在 AI 系统中，我们经常需要频繁使用大量不同分类的数据模型文件，在对这一类文件建立对象模型时，不仅会长时间占用 IO 读写资源，还会消耗大量 CPU 运算资源，如果频繁创建模型对象，就会很容易造成服务器 CPU 被打满而导致系统宕机。通过原型模式我们可以很容易地解决这个问题，当我们完成对象的第一次初始化后，新创建的对象便使用对象拷贝（在内存中进行二进制流的拷贝），虽然拷贝也会消耗一定资源，但是相比初始化的外部读写和运算来说，内存拷贝消耗会小很多，而且速度快很多；
2. **降低对象创建的时间消耗。** 比如，需要查询数据库来创建对象时，如果数据库正好繁忙或锁表中，那么这个创建过程就可能出现长时间等待的情况。在很多高并发场景中，稍微长时间的等待可能都是致命的，因为大量的数据和请求如洪水一般涌入服务器，很容易引起雪崩效应。这时使用原型模式就是相当于对对象创建的过程进行了一次缓存读取，而不必一直阻塞程序的执行；
3. **快速复制对象运行时状态**。原型模式相比于传统的使用 new 关键字创建对象还有一个巨大的优势，那就是当构造函数中包含大量属性或定制化业务逻辑时，不用完全了解创建过程也能快速创建对象。比如，当一个对象类有 30 个以上的属性或方法时（属性字段可能为另一个对象），如果你都通过 get 和 set 方法来创建对象，你会发现复制粘贴都是一件痛苦的事，因为你可能都忘记了哪些字段是必选、哪些又是有数据的。这也是我们在接收 HTTP 和 RPC 传输的 JSON 数据时，更愿意采用反序列化（也是一种原型模式的实践）到对象的方式，而不是 new 一个新对象再赋值的原因；
4. **能保存原始对象的副本。** 在某些需要保存历史状态的场景中，比如，聊天消息、上线发布流程、需要撤销操作的程序等，原型模式能快速地复制现有对象的状态并留存副本，方便快速地回滚到上一次保存或最初的状态，避免因网络延迟、误操作等原因而造成数据的不可恢复。
   
## 优缺点
主要有以下三个大的优点：
1. **原型并不基于继承，因此没有继承的缺点**。原型模式是对对象的直接复制，当新对象发生变化时，并不会对原始对象有任何影响，而继承的对象一旦发生了修改则会影响到父类；
2. **复制大对象时，性能更优**。比如，Java 使用的原型模式是基于内存二进制流的拷贝，而直接 new 一个大对象是 JVM 进行堆内分配并可能触发 Full GC，相比之下，使用 new 关键字时所做的操作实际上更多，而使用内存拷贝的方式在复制的性能上会更优；
3. **可以快速扩展运行时对象的属性和方法**。原型模式一方面简化了对象的创建过程，另一方面能够保留原始的对象状态，这样的优势是：在程序运行过程中，如果想要动态扩展对象的功能（增减方法或属性值），可以在不影响原有对象的情况下，动态扩展对象的功能。比如，结合 AOP 切面编程可以实现录制业务调用轨迹，加入应用性能监控，做动态数据埋点等操作。

原型模式也不是十全十美的，它也有一些缺点：
1. **虽然不基于继承，但原型需要一个被初始化过的正确对象**。如果被复制的对象在进行复杂的初始化时失败或出现错误的初始化后，那么复制的对象也可能是错误的；
2. **复制大对象时，可能出现内存溢出的 OOM 错误**。虽然复制对象有诸多优点，但是不要忘记内存的大小是有限制的，如果你想要复制的对象已经占用了 80% 的内存空间，那么复制时大概率会导致内存溢出，而这时的解决办法要么是增加内存，要么是拆分对象；
3. **动态扩展对象功能时可能会掩盖新的风险**。虽然原型模式能够在运行时帮助快速扩展功能，但同时也可能使新对象的负荷更重。比如，埋点服务中通常会拷贝一份对象在某个时间节点的数据，并添加一些追踪数据后再推送给埋点服务，这样就可能增加过多的内存消耗，影响原有功能执行的性能，有时还可能引起 OOM，导致系统宕机。切记，如果没有充分验证过动态扩展功能的话，不要轻易使用动态扩展，因为加入额外的新功能，大概率是会影响原有功能的。

## 总结

**原型模式可以说是创建型模式中灵活性最高的模式，**不过在带来灵活性的同时，也带来了更大的风险，这对我们的设计与实现间接提出了更高的要求。

使用原型模式时可能需要对 IO 流、内存和 JVM 等一些底层的原理有更加深入的理解才行，虽然对象的拷贝看上去很容易，如果一旦使用不当，很容易就导致系统直接崩溃，这也是不愿意使用原型模式的原因之一。

但是在很多类库和框架中，随处可见原型模式的身影，比如，JDK、Netty、Spring 等。还有 JavaScript、TypeScript 更是会时常用到原型模式。

在我看来，原型模式的实现——浅拷贝和深拷贝，本质上都是基于性能优化角度来更好地实现拷贝功能，只不过实现方式和使用场景有所不同而已。

决定了要采用原型模式后，再考虑使用哪种方式会更加合适。况且很多开源框架和组件里都有相关实现，并不一定非要从零去实现浅拷贝或深拷贝，比如：
- Apache Common 包中的 deepCopy；
- Spring 中的 BenUtils.copyProperties 等，你若感兴趣的话可以去研究相关源码，这里就不赘述了。
