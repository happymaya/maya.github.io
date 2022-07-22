---
title: 解释器模式：实现自定义配置规则功能
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2018-02-17 21:18:32 +0800
categories: [Design Pattern, behavioral]
tags:  [设计模式, Design Pattern, Interpreter, 解释器模式, 类行为型模式]
math: true
mermaid: true
---

解释器模式使用频率不算高，**通常用来描述如何构建一个简单“语言”的语法解释器。**它只在一些非常特定的领域被用到，比如：

- 编译器；
- 规则引擎；
- 正则表达式；
- SQL 解析等。

不过，了解它的实现原理，可以帮助思考**如何通过更简洁的规则来表示复杂的逻辑**。
### 模式原理分析
解释器模式的原始定义是：**用于定义语言的语法规则表示，并提供解释器来处理句子中的语法。**
语法也称文法，在语言学中指任意自然语言中句子、短语以及词等语法单位的语法结构与语法意义的规律：

- 比如：在编程语言中，if-else 用作条件判断的语法，for 用于循环语句的语法标识；
- 再比如，“我爱中国”是一个中文句子，可以用名词、动词、形容词等语法规则来直观地描述句子。

解释器模式的 UML 图：
![](https://cdn.nlark.com/yuque/0/2022/jpeg/12442250/1658392008202-d1d550c4-f8c1-4e58-b89a-34cf7976713e.jpeg)
从该 UML 图中，可以看出解释器模式包含四个关键角色：

1. **抽象表达式（AbstractExpression）**：定义一个解释器有哪些操作，可以是抽象类或接口，同时说明只要继承或实现的子节点都需要实现这些操作方法；
1. **终结符表达式（TerminalExpression）**：用于解释所有终结符表达式；
1. **非终结符表达式（NonterminalExpression）**：用于解释所有非终结符表达式；
1. **上下文（Context）**：包含解释器全局的信息。

解释器模式 UML 对应的代码实现如下：
```java
/**
 * 抽象表达式类
 */
public interface AbstractExpression {
    boolean interpreter(Context context);
}

/**
 * 上下文信息类
 */
public class Context {
    private String data;

    public Context(String data) {
        this.data = data;
    }

    public String getData() {
        return data;
    }

    public void setData(String data) {
        this.data = data;
    }
    
}

/**
 * 终结符表达式类
 */
public class TerminalExpression implements AbstractExpression {

    private final String data;

    public TerminalExpression(String data) {
        this.data = data;
    }

    @Override
    public boolean interpreter(Context context) {
        return context.getData().contains(data);
    }
    
}

/**
 * 非终结符表达式类
 */
public class NonTerminalExpression implements AbstractExpression{

    AbstractExpression expr1;
    AbstractExpression expr2;

    public NonTerminalExpression(AbstractExpression expr1, AbstractExpression expr2) {
        this.expr1 = expr1;
        this.expr2 = expr2;
    }

    @Override
    public boolean interpreter(Context context) {
        return expr1.interpreter(context) && expr2.interpreter(context);
    }

}

/**
 * 单元测试类
 */
public class Demo {

    public static void main(String[] args) {

        Context context1 = new Context("mick,mia");
        Context context2 = new Context("mick,mock");
        Context context3 = new Context("spike");

        AbstractExpression person1 = new TerminalExpression("mick");
        AbstractExpression person2 = new TerminalExpression("mia");
        AbstractExpression isSingle = new NonTerminalExpression(person1, person2);
        System.out.println(isSingle.interpreter(context1));
        System.out.println(isSingle.interpreter(context2));
        System.out.println(isSingle.interpreter(context3));
    }
}


//输出结果
// true
// false
// false

```
在上面的代码实现中，NonterminalExpression 用于判断两个表达式是否都存在，存在则在解释器判断时输出 true，如果只有一个则会输出 false。也就是说，表达式解释器的解析逻辑放在了不同的表达式子节点中，这样就能通过增加不同的节点来解析上下文。
所以说，解释器模式原理的本质就是**对语法配备解释器，通过解释器来执行更详细的操作。**
### 使用场景分析
一般来讲，解释器模式常见的使用场景有这样几种：

1. **当语言的语法较为简单并且对执行效率要求不高时**。比如，通过正则表达式来寻找 IP 地址，就不需要对四个网段都进行 0~255 的判断，而是满足 IP 地址格式的都能被找出来；
1. **当问题重复出现，且可以用一种简单的语言来进行表达时**。比如，使用 if-else 来做条件判断语句，当代码中出现 if-else 的语句块时都统一解释为条件语句而不需要每次都重新定义和解释；
1. **当一个语言需要解释执行时**。如 XML 文档中<>括号表示的不同的节点含义。

下面通过一个简单的例子来详细说明。
创建一个逻辑与的解释器例子。简单来说，就是通过字符串名字来判断表达式是否同时存在，存在则打印 true，存在一个或不存在都打印 false。
在下面的代码中，会创建一个接口 Expression 和实现 Expression 接口的具体类，并定义一个终结符表达式类 TerminalExpression 作为主解释器，再定义非终结符表达式类，这里 OrExpression、AndExpression 分别是处理不同逻辑的非终结符表达式。
```java
public interface IExpression {
    boolean interpreter(String con);
}

public class TerminalExpressionImpl implements IExpression{

    String data;

    public TerminalExpressionImpl(String data) {
        this.data = data;
    }

    @Override
    public boolean interpreter(String conn) {
        return conn.contains(data);
    }
}

public class AndExpressionImpl implements IExpression{

    IExpression expr1;
    IExpression expr2;

    public AndExpressionImpl(IExpression expr1, IExpression expr2) {
        this.expr1 = expr1;
        this.expr2 = expr2;
    }

    @Override
    public boolean interpreter(String con) {
        return expr1.interpreter(con) && expr2.interpreter(con);
    }
    
}

public class QrExpressionImpl implements IExpression{

    IExpression expr1;
    IExpression expr2;

    public QrExpressionImpl(IExpression expr1, IExpression expr2) {
        this.expr1 = expr1;
        this.expr2 = expr2;
    }

    @Override
    public boolean interpreter(String con) {
        return expr1.interpreter(con) || expr2.interpreter(con);
    }
    
}

public class Client {
    public static void main(String[] args) {
        IExpression person1 = new TerminalExpressionImpl("mick");
        IExpression person2 = new TerminalExpressionImpl("mia");
        IExpression isSingle = new QrExpressionImpl(person1, person2);

        IExpression spike = new TerminalExpressionImpl("spike");
        IExpression mock = new TerminalExpressionImpl("mock");
        IExpression isCommitted = new AndExpressionImpl(spike, mock);

        System.out.println(isSingle.interpreter("mick"));
        System.out.println(isSingle.interpreter("mia"));
        System.out.println(isSingle.interpreter("max"));
        System.out.println(isCommitted.interpreter("mock, spike"));
        System.out.println(isCommitted.interpreter("Single, mock"));
    }
}


```
在最终单元测试的结果中，可以看到：在表达式范围内的单词能获得 true 的返回，没有在表达式范围内的单词则会获得 false 的返回。
### 使用解释器模式的原因
使用解释器模式的原因，可总结为以下两个：

1. **将领域语言（即问题表征）定义为简单的语言语法**。这样做的目的是通过多个不同规则的简单组合来映射复杂的模型。比如，在中文语法中会定义主谓宾这样的语法规则，当我们写了一段文字后，其实是可以通过主谓宾这个规则来进行匹配的。如果只是一个汉字一个汉字地解析，解析效率会非常低，而且容易出错。同理，在开发中我们可以使用正则表达式来快速匹配IP地址，而不是将所有可能的情况都用 if-else 来进行编写；
1. **更便捷地提升解释数学公式这一类场景的计算效率**。我们都知道，计算机在计算加减乘除一类的数学运算时，和人类计算的方式是完全不同的，需要通过一定的规则运算才能得出最后的结果。比如，3+2-（4 X 5)，如果我们不告诉计算机先要运算括号中的表达式，计算机则只会按照顺序进行计算，这显然是错误的。而使用解释器模式，则能很好地通过预置的规则来进行判断和解释。
### 优缺点
使用解释器模式主要有以下优点：

1. **很容易改变和扩展语法逻辑**。由于在模式中是使用类来表示语法规则的，因此当我们需要新增或修改规则时，只需要新增或修改类即可。同时，还可以使用继承或组合方式来扩展语法；
1. **更容易实现语法**。我们可以定义节点的类型，并编写通用的规则来实现这些节点类，或者还可以使用编译器来自动生成它们。

同样，解释器模式也不是万能的，也有一些缺点：

1. **维护成本很高**。语法中的每个规则至少要定义一个类，因此，语法规则越多，类越难管理和维护；
1. **执行效率较低**。由于解释器模式会使用到树的数据结构，那么就会使用大量的循环和递归调用来访问不同的节点，当需要解释的句子语法比较复杂时，会执行大量的循环语句，性能很低；
1. **应用场景单一，复用性不高**。在开发中，除了要发明一种新的编程语言或对某些新硬件进行解释外，解释器模式的应用实例其实非常少，加上特定的数据结构，扩展性很低。
### 总结
**在实际的业务开发中，解释器模式很少使用，主要应用于 SQL 解析、符号处理引擎等场景中。**
在解释器模式中通常会使用树的结构，有点类似于组合模式中定义的树结构，**终端表达式对象是叶对象，非终端表达式是组合对象**。
虽然解释器模式很灵活，能够使用语法规则解析很多复杂的句子，比如，编程语法。但是稍不留神就很容易把解释逻辑写在一个类中，进而导致后期难以维护。除此之外，把解析逻辑拆分为单个的子节点后，又会因为类数量的暴增，导致代码的理解难度倍增。
不过，**解释器模式能够通过一些简短的规则来解决复杂的数据匹配问题，**比如，正则表达式 [0-9] 就能匹配数字字符串。所以说，理解解释器模式的原理还是很有必要的。
