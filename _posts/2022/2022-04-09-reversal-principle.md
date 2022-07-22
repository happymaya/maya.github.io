---
title: 反转原则：减少代码间相互影响
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2022-04-09 11:30:20 +0800
categories: [Design Pattern]
tags: [DIP]
math: true
mermaid: true
---

SOLID 五大设计原则中，其中有一个原则，可能用得很熟练，但是说到概念却又容易跟其他原则混淆的，那就是**依赖反转原则**（Dependence Inversion Principle，**简称 DIP**）。
在实际的研发工作中，可能会遇到以下场景：

- 一个平台系统，需要接入各种各样的业务系统，而这些业务系统都有自己的账号体系，平台需要兼容这些系统的账号体系，于是代码中出现了大量依赖于各种账号体系的代码；
- 一个网站页面，需要越来越多的频道（首页、搜索、分类等），不同频道对应的个性化需求各不相同，并且各种页面的标准组件、布局、模板，以及与后端交互框架也各不相同，不同体系的代码依赖非常紧密；
- 一个通用的订单处理平台，各条业务线都需要通过这个平台来处理自己的交易业务，但是垂直业务线上的个性化需求太多，代码里随处可见定制化的需求代码。

对于这些问题，可能已经有解决方案了：如果依赖和控制的东西过多了，就需要制定标准，反转控制，解耦，分层……甚至也知道该如何在代码中解决这些问题，比如，面向接口编程，而不是面向实现编程。
在这个解决过程中，其实已经在使用 DIP 了，可能对于 DIP 概念本身，你还没有透彻理解，甚至说到 DIP 时感觉还很陌生。
# DIP：统一代码交互的标准
对于“依赖反转”这种抽象性概念，可以结合现实生活中的场景或例子来剖析和理解。
比如，在没有电商的时代，商品交易时，通常是买家一手交钱、卖家一手交货，所以基本上卖家和买家必须强耦合（因为必须要见面交易）。而这时有一个中间商想出了一个更好的办法，让银行出面做交易担保——买家把钱先付给银行，银行收到钱后让卖家发货，买家收验货后，银行再把钱打给卖家。通过这样的方式，买卖双方把对对方的直接控制，反转到了让对方来依赖一个标准的交易模型的接口——银行。
这和浏览器的使用原理也很类似。浏览器（对应商品买家）并不依赖于后面的 Web 服务器（对应商品卖家），其只依赖于 HTTP 协议（对应银行），只要我们遵循 HTTP 协议就能在浏览器中提供很多丰富的 Web 功能，而不必针对特定的浏览器定制开发。
因此，可以总结一下，**依赖反转原则（DIP）就是一种统一代码交互标准的软件设计方法**。
回到 DIP 的概念上来，看一下它的原始定义：

- 高级组件不应依赖于低级组件，两者都应依赖抽象；
- 抽象不应该依赖实现，实现应该依赖抽象。

如何去理解 DIP 这个原始定义：

1. **定义中的高级组件和低级组件，主要对应的是调用关系上的层级**：
   1. 比如，汽车油门（高级组件）调用汽车引擎（低级组件），但并不是说汽车油门就比汽车引擎更复杂、功能更完善、能力更高；
   1. 再比如，软件程序都得依赖底层操作系统，而你不能说软件程序就一定比操作系统复杂；
2. **高级组件和低级组件都应依赖抽象，是为了消除组件间变化对对方造成的影响，换句话说，抽象是一种约束，让高级组件或低级组件不能太随意地变动**。因为两者间有相互依赖关系，一方变化或多或少都会带给对方影响。
   1. 比如，踩油门是加油，抬起油门是减油，这是一种抽象约束，只要约束不变，设计圆形油门还是方形油门都不会影响引擎的动力控制；反过来，引擎使用铝制还是铁制，也不影响引擎对油门加、减油的控制；
3. **抽象不应该依赖实现，实现应该依赖抽象**。什么意思呢？这里拿 JDBC 这个数据库驱动协议作为例子来简单解释一下。在用 Java 开发增删改查的数据业务时，我们通常会开发一个数据库访问层——DAO 层，而它并不直接依赖于数据库驱动（实现），而是依赖于 JDBC 这个抽象。JDBC 并没有受不同数据库设计的影响，只要不同数据库驱动都实现了 JDBC，就能被 DAO 层所使用，而为了让应用程序使用，数据库驱动也依赖于 JDBC，这便是抽象不应该依赖实现，实现应该依赖抽象。
# 为什么要使用 DIP？
使用 DIP 的意义，大致总结为如下两点：

- 可以有效地控制代码变化的影响范围；
- 可以使代码具有更强的可读性和可维护性。

正如文章开头提到的那些场景问题，它们都有一个共同点：如果别人的程序一发生变化，我们的代码就得跟着发生变化，也就意味着我们又得加班加点改代码，这是一件是非常痛苦的事情。简单来说，**外部系统的需求或功能变化影响到了内部系统。**
# 使用 DIP 的目的

1. **控制代码变化带来的影响。**
   1. 比如，为了解决平台接入权限问题，可以通过抽象一个账号权限体系的接口标准，让不同业务系统按照这个统一标准来接入平台，同时我们的平台也按照这个标准来实现。此时，我们的内部系统和外部系统不再是通过定制化的映射来通信，而是使用了一套统一的标准接口来通信，只要接口不发生变化，即使外部系统发生了巨大变化，接入的功能并没有发生改变，这样就能有效地控制外部系统的变化对内部系统带来的影响；
2. **增强代码的可读性和可维护性**：
   1. 虽然现如今软件行业提倡“敏捷开发”，少写文档和注释，需求能提前上线更重要，但是，少写文档不是让我们不写文档和不写注释。比如，你接手两个项目，一个项目不仅没有系统设计文档，还没有代码注释，同时代码逻辑依赖又很多，到处都是看不懂的定制化逻辑；而另一个项目，只有少量文档，并且还有很清晰的接口定义和代码注释。通过这样的对比，立马就能知道哪个项目的维护难度更低！
   1. 使用 DIP，就是从设计上减少系统的耦合性，更能帮助厘清代码逻辑，因为代码是通过统一抽象之后，功能相同的处理都在同一个地方，所以，代码变得更加顺畅，更容易让人理解，也就增强了代码的可读性和可维护性。
# 具体实现抽象标准接口
下面**通过设计一个 Java 组件来演示其应用思路**。
这个组件只实现一个功能：**读取一串字符后，再输出显示**：

- 组件中定义 `StringProcessor` 类，该类使用 `StringReader` 组件获取 `String` 值，然后使用 `StringWriter` 组件将值写入输出流并打印；
- 为方便理解组件的作用关系，将 StringReader 类和 StringWriter 类统称为低级组件，`StringProcessor `称为高级组件，这样能更清楚地了解每个设计选择是如何影响整体设计的。
## 设计一：低级组件和高级组件都作为具体类放在同一包中
StringProcessor 取决于 StringReader 和 StringWriter 的实现，读取字符并输出打印，整个逻辑实现在同一个包内通过三个具体类来完成，如下示例代码：
```java
public class StringProcessor { //具体类

    private final StringReader stringReader; //具体类

    private final StringWriter stringWriter; //具体类

    //通过构造函数来注入依赖组件

    public StringProcessor(StringReader stringReader, StringWriter stringWriter) {

        this.stringReader = stringReader;

        this.stringWriter = stringWriter;

    }

    public void readAndWrite() {

        stringWriter.write(stringReader.getValue());

    }

    //测试用例

    public static void main(String[] args) {

        StringReader sr = new StringReader();

        sr.read("1111111");

        StringWriter sw = new StringWriter();

        StringProcessor sp = new StringProcessor(sr,sw);

        sp.readAndWrite();

    }

}

```
这是最基本的设计方法。
优点：

- 逻辑简单、编码容易；

缺点：

- 程序高度耦合，任何一个低级组件的修改都会影响高级组件。这时，如果想要复用 StringProcessor 组件，则需要在引入的地方写很多重复代码；
- 从 DIP 应用的角度来看，控制逻辑这时依然是由高级组件发起，没有达到依赖反转的效果。
## 设计二：低级组件具体类与高级组件接口实现类放在同一程序包中
有了设计一的基础版本，要想优化，自然会想到 Java 中的**面向接口编程思想**。
下面继续改造。
首先，将 StringProcessor 变为一个接口：
```java
public interface IStringProcessor {
   void readAndWrite(StringReader stringReader, StringWriter stringWriter);
}
```
 这时，通过传递 StringReader 和 StringWriter 参数作为组件依赖，实现 StringProcessor 接口，就能实现读写功能。  
```java
public class StringProcessorImpl implements IStringProcessor{
    @Override
    public void readAndWrite(StringReader stringReader, StringWriter stringWriter) {
        stringWriter.write(stringReader.getValue());
    }

    public static void main(String[] args) {
        StringReader sr = new StringReader();
        sr.read("222222");
        StringWriter sw = new StringWriter();
        IStringProcessor sp = new StringProcessorImpl();
        sp.readAndWrite(sr,sw);
    }
}
```
现在，StringProcessorImpl 实现类依赖于抽象类 StringProcessor，符合 DIP——实现应该依赖抽象，按理说高级组件就应该很好地被复用。
其实不然，这里会遇见**最常见的误区之一：只要组件有接口就代表一定可复用。**
为什么说就算组件有接口，也并不代表实际上就真的能复用呢？因为低级组件没有依赖于抽象，这时的**高级组件依然是直接依赖于低级组件**，一旦低级组件发生变化，高级组件必然要发生变化，就像这里的例子一样，虽然给高级组件抽象了一个接口，但低级组件依然没有依赖抽象。
而对于外部复用这个组件的系统或服务来说，一旦 StringReader 或 StringWriter 发生变化，则会导致自己也要跟着变化，这就没有达到复用的基本效果——内部组件变化不影响外部引用。
因此，依然没有实现依赖关系的反转，控制权还是在高级组件那里，组件间还是高度耦合。
## 设计三：低级组件接口类与高级组件接口实现类放在同一程序包中
既然设计二里的低级组件变化会影响高级组件，那么，为了更好地解决问题，新的设计选择就应该把低级组件也进行抽象。
这时，分别将 StringReader 和 StringWriter 进行抽象，如下代码所示：
```java
public interface StringReader {
    void read(String path);
    String getValue();
}

public interface StringWriter {
    void write(String value);
}
```
 StringProcessor 接口定义保持不变，不过，StringWriter 和 StringReader 的类型已从具体类变为了接口。  
```java
public class StringProcessorNewImpl implements IStringProcessorNew{

    @Override
    public void readAndWrite(IStringReader stringReader, IStringWriter stringWriter) {
        stringWriter.write(stringReader.getValue());
    }

    public static void main(String[] args) {
        IStringReader sr = new StringReaderImpl();

        sr.read("333333");                                                                              

        IStringWriter sw = new StringWriterImpl();

        IStringProcessorNew sp = new StringProcessorNewImpl();

        sp.readAndWrite(sr,sw);
    }
}

```
此时，StringProcessor、StringReader 和 StringWriter 都依赖于抽象，整体组件的逻辑控制权真正发生了变化，通过抽象化组件之间的交互，已经实现了从上到下的依赖关系的反转。
过去，StringProcessor 依赖于 StringReader 和 StringWriter 的具体实现，而现在则是 StringProcessor 来定义一组抽象规则，由 StringReader 和 StringWriter 来依赖。
## 设计四：低级组件接口类与高级组件接口类放在不同包中
在设计三中，成功实现了依赖反转，使得在实现 StringReader 和 StringWriter 时也更加容易，不过，因为低级组件的实现和高级组件的实现这时还在同一个组件包内，不方便真正复用。而在真实场景中，是需要使用不同的包或框架。
下面的代码示例中，使用了 Spring 框架的注解注入来实现设计。
```java
public class SPTest {

    @Resource
    private StringProcessor sp;
    @Resource
    private StringReader sr;
    @Resource
    private StringWriter sw;
    
    public void main(String[] args) {
        sr.read("444444");
        sp.readAndWrite(sr,sw);
    }
}
```
对于 StringReader 和 StringWriter 的实现这时就变成了“黑盒”，换句话说，对于使用者来说，我们可以完全不用关心 StringReader 和 StringWriter 是如何实现的，实现可以是 JDK 原生实现，可以是第三方的包实现，也可以是我们自行实现的，只要低级组件按照抽象约定的提供读写功能即可。
到此，在上面的四种设计中，设计三和设计四都是对 DIP 的有效实现。虽然是用 Java 来编码的，但使用其他语言（Python、Go 等）时，同样能够运用这个思路来实践 DIP。
# IoC、DI、IoC 容器与 DIP 的区别
说起 DIP，会经常听到三个相关名词：

- **依赖注入（DI）**
- **控制反转（IoC）**
- **IoC 容器**

稍不注意，就容易把它们混淆在一起。那它们和 DIP 之间到底有什么区别和联系呢？
为了更好做出区分，先要搞清楚 IoC、DI 和 IoC 容器基本概念：

- **控制反转**（Inversion of Control，**简称 IoC**）：
   - 一种设计原则（也有人将其称为设计模式）；
   - 顾名思义，用于反转设计中各种组件的控制关系，以实现松耦合；
   - 这里的控制是指对象除自身主要职责以外的任何其他职责，通常包括对应用程序流控制，以及对象创建或子对象创建、绑定控制：
      - 比如，开车去上班，这意味着你要控制一辆汽车，而 IoC 的理念是反转你的控制，你不用自己开车而是选择打车，让出租车师傅带你去上班，这时你的控制就发生了反转，出租车师傅专注于开车，而你专注于上班。
- **依赖注入**（Dependency Injection，**简称 DI**）：
   - 用于实现 IoC 的设计模式；
   - 简单来说，**它允许在类之外创建依赖对象，并通过不同方式将这些对象提供给类**。一般来讲，主要有三种方式来注入类：
      - 通过构造函数
      - 通过属性
      - 通过方法
- **IoC 容器（又叫作 DI 容器）：**
   - **用于实现自动依赖注入的框架**；
   - 它的作用是管理对象的创建及其生命周期，并提供向类注入依赖项的具体实现，这样做是为了不用手动创建和管理对象；
   - **实际上更准确的描述应该是 DI 容器**，只不过因为 Spring 号称自己为 IoC 容器而造成了误解。

IoC、DI、IoC 容器和 DIP 之间的关系如下图所示：
![图片.png](https://cdn.nlark.com/yuque/0/2022/png/12442250/1657702494716-54c5a1b8-aa66-4094-97cc-4734913786c7.png#clientId=u4fb432a1-a4ec-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=336&id=u35a28ff1&margin=%5Bobject%20Object%5D&name=%E5%9B%BE%E7%89%87.png&originHeight=336&originWidth=741&originalType=binary&ratio=1&rotation=0&showTitle=false&size=127147&status=done&style=none&taskId=uab289079-36a1-495f-a156-2a68ea84a19&title=&width=741)
如上图所示，这四者之间关系可总结为：

- IoC 容器，一种技术框架，用来管理对象创建以及其生命周期，提供依赖注入实现，是 DI 的具体实现；
- DI， 一种设计模式，将依赖通过“注入”的方式提供给需要的类，是 DIP 和 IoC 的具体实现；
- IoC，一种设计原则（或设计模式），将代码本职之外的工作交由某个第三方（框架）完成，与 DIP 相似；
- DIP，	一种设计原则，它认为高层组件的功能不应该依赖下层组件的实现，而应该提供抽象层让下层依赖，与 IoC 有异曲同工之妙。

这里要重点注意，**DIP 既不是 DI 也不是 IoC，只不过因为它们长期一起“工作”，界限常常被模糊掉罢了；而常说的 IoC 容器通常是指 Java 中的 Spring IoC 容器，而实际上应该是 DI 容器。**
# 总结

1. **依赖反转原则（DIP）**
   1. 一种设计理念，是为了帮助我们解耦复杂的程序。换句话说，DIP 是一种简单但功能强大的设计思想，可以使用它来**实现结构良好、高度分离和可重用的软件组件**；
   1. 带来一个重要启示：不管是程序设计还是工作生活，如果依赖和控制的东西过多了，就要**学会制定标准，倒置依赖，反转控制，释放自身资源，专注于更重要的事；**
   1. 在理解反转真实含义后，在设计时会用到一个重要技巧：**要让高层组件拥有定义抽象的权力，而不是把这个权力下放到低层组件。**
2. **使用 DIP 的原因**
   1. 有效控制代码变化的影响范围；
   1. 使代码具有更强的可读性和可维护性。
3. **DI、IoC、IoC 容器与 DIP 的区别与联系**
   1. 切记，**DIP ≠ DI + IoC；**
   1. 依赖注入（DI），是一种设计模式，是 DIP 与 IoC 具体实现；
   1. 控制反转（IoC），是一种设计原则，核心点在于通过分离职责，让控制被反转，与 DIP 类似；
   1. 另外，通常所说的 IoC 容器其实是特指 Java 中的 Spring 的 IoC 容器，而实际上真正应用更广泛的是 DI 容器。
