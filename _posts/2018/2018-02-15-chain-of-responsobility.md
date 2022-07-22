---
title: 责任链模式
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2018-02-15 21:18:32 +0800
categories: [Design Pattern, behavioral]
tags:  [设计模式, Design Pattern, Chain of Responsibility, 责任链模式, 对象行为型模式]
math: true
mermaid: true
---

相较而言，责任链模式（Chain of Responsibility）是一个使用频率很高的模式，它属于一个行为型对象设计模式。
# 模式原理
责任链模式的原始定义是：**通过为多个对象提供处理请求的机会，避免将请求的发送者与其接收者耦合。链接接收对象并沿着链传递请求，直到对象处理它。**
这个定义读起来抽象难懂，实际上它只说了一个关键点：**通过构建一个处理流水线来对一次请求进行多次的处理**。
就像网购一样：当你收到了购的商品后，发现商品有质量问题，于是打电话询问客服关于退货流程，客服接到电话后，会先打开订单系统查询提供的订单信息并确认是否正确，确认后再使用物流系统通知快递小哥上门取件，快递小哥取件后会返回商品让仓储系统进行确认，并通知商品系统……这样的一个过程就是责任链模式的真实应用。
责任链模式的 UML 图：
![图片.png](https://cdn.nlark.com/yuque/0/2022/png/12442250/1657703472198-abb60b2d-7194-4423-8707-3da864f1e33e.png#clientId=u9ed7941a-33a4-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=322&id=u4f99c124&margin=%5Bobject%20Object%5D&name=%E5%9B%BE%E7%89%87.png&originHeight=322&originWidth=689&originalType=binary&ratio=1&rotation=0&showTitle=false&size=75016&status=done&style=none&taskId=u3d1f0534-d92d-4d37-9bcb-cef2dd1e4da&title=&width=689)
从该 UML 图中，可以看出责任链模式其实只有两个关键角色。

- **处理类（Handler）**：可以是一个接口，用于接收请求并将请求分派到处理程序链条中（实际上就是一个数组链表），其中，会先将链中的第一个处理程序放入开头来处理。
- **具体处理类（HandlerA、B、C）**：按照链条顺序对请求进行具体处理。

下面我们再来看看该 UML 对应的代码实现：
```java
public interface Handler {
    void setNext(Handler handler);
    void handle(Request request);
}

public class HandlerA implements Handler {

    private Handler next;

    @Override
    public void setNext(Handler handler) {
        this.next = handler;
    }

    @Override
    public void handle(Request request) {
        System.out.println("HandlerA 执行 代码逻辑, 处理: " + request.getData());
        request.setData(request.getData().replace("AB", ""));

        if (null != next) {
            next.handle(request);
        } else {
            System.err.println("执行中止!");;
        }
    }
}

public class HandlerB implements Handler{

    private Handler next;

    public HandlerB() {

    }

    @Override
    public void setNext(Handler handler) {
        this.next = handler;
    }

    @Override
    public void handle(Request request) {
        System.out.println("HandlerB 执行 代码逻辑, 处理: " + request.getData());
        request.setData(request.getData().replace("CD", ""));

        if (null != next) {
            next.handle(request);
        } else {
            System.err.println("执行中止!");;
        }
    }
}

public class HandlerC implements Handler{

    private Handler next;

    public HandlerC() {

    }

    @Override
    public void setNext(Handler handler) {
        this.next = handler;
    }

    @Override
    public void handle(Request request) {
        System.out.println("HandlerC 执行 代码逻辑, 处理: " + request.getData());

        if (null != next) {
            next.handle(request);
        } else {
            System.err.println("执行中止!");;
        }
    }
}

public class Request {

    private String data;

    public String getData() {
        return data;
    }

    public void setData(String data) {
        this.data = data;
    }
    
}
```
从这段代码实现可以看出，责任链模式的实现非常简单，每一个具体的处理类都会保存在它之后的下一个处理类。当处理完成后，就会调用设置好的下一个处理类，直到最后一个处理类不再设置下一个处理类，这时处理链条全部完成。在代码示例中，HandlerA 删除掉字符串 ABCDE 中的 AB，并交给 HandlerB 处理；HandlerB 删除掉 CDE 中的 CD，并交给 HandlerC；HandlerC 处理完后，整个执行过程中止。
# 使用场景
责任链模式常见的使用场景有以下几种情况。

- **在运行时需要动态使用多个关联对象来处理同一次请求时**。比如：**请假流程、员工入职流程、编译打包发布上线流程等。**
- **不想让使用者知道具体的处理逻辑时**。比如，**做权限校验的登录拦截器。**
- **需要动态更换处理对象时**。比如，工单处理系统、**网关 API 过滤规则系统**等。

下面通过一个简单的例子演示。这里创建一个获取数字并判断正负或零的程序，程序接收一个数字的请求，在链条上进行处理并打印对应的处理结果。
先创建一个链条 Chain，并设置起始的处理类，如下代码所示：
```java
public class Chain {

    Excutor chain;

    public Chain() {
        buildChain();
    }

    private void buildChain() {
        Excutor e1  = new NegativeExcutor();
        Excutor e2 = new ZeroExcutor();
        Excutor e3 = new PositiveExcutor();

        e1.setNext(e2);
        e2.setNext(e3);

        this.chain = e1;
    }

    public void process(Integer num) {
        chain.handle(num);
    }
}

```
 接下来创建抽象的处理类 Excutor，声明两个方法：

- setNext 用于设置下一个处理类；
- handle 是具体的业务逻辑。  
```java
/**
 * 抽象处理类
 */
public interface Excutor {

    /**
     * 用于设置下一个处理类
     * @param excutor
     */
    void  setNext(Excutor excutor);

    /**
     * 具体业务逻辑
     * @param num
     */
    void handle(Integer num);

}
```
 NegativeExcutor、PositiveExcutor 和 ZeroExcutor 分别代表处理负数、正数和零。  
```java
/**
 * 处理负数
 */
public class NegativeExcutor implements Excutor{

    private Excutor next;

    @Override
    public void setNext(Excutor excutor) {
        this.next = excutor;
    }

    @Override
    public void handle(Integer num) {
        if (null != num && num < 0) {
            System.out.println("NegativeExcutor 获取数字: " + num + ", 处理完成! ");
        } else {
            if (null != next) {
                System.out.println("=== 经过 NegativeExcutor");
                next.handle(num);
            } else {
                System.out.println("处理终止! - NegativeExcutor");
            }
        }
    }
}

/**
 * 处理正数
 */
public class PositiveExcutor implements Excutor{

    private Excutor next;

    @Override
    public void setNext(Excutor excutor) {
        this.next = excutor;
    }

    @Override
    public void handle(Integer num) {
        if (null != num && num > 0) {
            System.out.println("PositiveExcutor 获取数字: " + num + ", 处理完成! ");
        } else {
            if (null != next) {
                System.out.println("=== 经过 PositiveExcutor");
                next.handle(num);
            } else {
                System.out.println("处理终止! - PositiveExcutor");
            }
        }
    }
}

/**
 * 处理零
 */
public class ZeroExcutor implements Excutor{

    private Excutor next;

    @Override
    public void setNext(Excutor excutor) {
        this.next = excutor;
    }

    @Override
    public void handle(Integer num) {
        if (null != num && num == 0) {
            System.out.println("ZeroExcutor 获取数字: " + num + ", 处理完成! ");
        } else {
            if (null != next) {
                System.out.println("=== 经过 ZeroExcutor");
                next.handle(num);
            } else {
                System.out.println("处理终止! - ZeroExcutor");
            }
        }
    }
}
```
 最后，运行一个单元测试：  
```java
public class Client {
    public static void main(String[] args) {
        Chain chain = new Chain();

        chain.process(99);
        System.out.println("---------");

        chain.process(-11);
        System.out.println("---------");

        chain.process(0);
        System.out.println("---------");

        chain.process(null);
    }
}

// 输出结果
// === 经过 NegativeExcutor
// === 经过 ZeroExcutor
// PositiveExcutor 获取数字: 99, 处理完成! 
// ---------
// NegativeExcutor 获取数字: -11, 处理完成! 
// ---------
// === 经过 NegativeExcutor
// ZeroExcutor 获取数字: 0, 处理完成! 
// ---------
// === 经过 NegativeExcutor
// === 经过 ZeroExcutor
// 处理终止! - PositiveExcutor
```
从最后结果可以看到，当输入不同的数时，都会经过一整个链条的流转，直到最终的处理对象完成处理。
所以说，责任链模式就像工厂的流水线作业一样，按照某一个标准化的流程来执行，用于规则过滤、Web 请求协议解析等具备链条式的场景中，通过拆分不同的处理节点来完成整个流程的处理。
# 使用责任链模式的原因
使用责任链模式的原因，可总结为以下三个：

1. **解耦使用者和后台庞大流程化处理**。在线购物订单里包含了物流、商品、支付、会员等多个系统的处理逻辑，如果让使用者一一和它们对接，势必会造成使用困难、系统之间调用混乱的情况发生，而通过订单建立一个订单的状态变更流程，就能将这些系统很好地串联在一起，这不仅能够让使用者只需要关注订单流程这一个入口，同时还能够让不同的系统按照各自的职责来发挥作用。比如，订单在未完成支付前，商品系统是不会通知物流系统进行商品发货的；
1. **为了动态更换流程处理中的处理对象**。比如，在请假流程中，申请人一般会提交申请给直接领导审批，但有时直接领导可能无法进行审批操作，这时系统就可以更换审批人到其他审批人，这样就不会阻塞请假流程的审批；
1. **为了处理一些需要递归遍历的对象列表**。比如，权限的规则过滤。对于不同部门不同级别人员的权限，就可以采用一个过滤链条来进行权限的管控。
# 优缺点
通过上述分析，责任链模式的优点：

1. **降低客户端对象与处理链条上对象之间的耦合度**。比如，提交上线审核，提交人只知道最开始申请的处理人是谁，而后续是否需要别的审核人其实是由处理链条来控制的；
1. **提升系统扩展性**。对于需要多次处理的同一个请求，可以在链条上增加新的具体处理类，满足开闭原则，能极大地提升系统扩展性；
1. **增强了具体处理类的职责独立性**。即便链条上的工作流程发生了变化，也可以动态地改变具体处理类的调用次序和增加类的新的职责。每个类只需要处理自己该处理的工作，不该处理的就传递给下一个对象完成，明确各类的责任范围，同时也符合类的单一职责原则；
1. **简化了对象之间前后关联处理的复杂性**。每个对象只需存储一个指向后继者的引用，不需保持其他所有处理者的引用，这避免了使用众多的 if 或者 if···else 语句。

同样，责任链模式的缺点：

1. **降低性能**。由于每一个请求都需要经历一次完整的链条上具体处理类的处理，系统性能势必会受到一定影响，比如，依赖更多的代码行或依赖更复杂的代码逻辑；
1. **调试难度增大**。调试代码需要验证每个具体处理者是否都能接收到请求，一旦出现错误，排查与修改也变得更加麻烦；
1. **容易出现死锁异常**。一旦某一个对象设置后继者出现错误，就会出现循环调用，进而导致堆栈溢出的错误。
# 总结
在实际软件开发中，责任链模式应用非常广泛，可以说**只要是与流程相关的软件系统都能够使用责任链模式来构建：**

- 一方面可以用在代码中实现松散耦合；
- 另一方面可以动态增删子处理流程。

责任链模式的原理和实现虽然都非常简单，但是在实际使用中需要**注意：**

- **维护上下文关系的正确性**，一旦出现循环调用，很容易死锁而导致程序崩溃；
- **控制责任链中的处理对象数量**。如果处理对象的数量过多，比如超过 20 个，容易让代码变得难以维护，这时还是应该尽可能减少处理对象的数量，将其合并到相类似的处理对象中去。
