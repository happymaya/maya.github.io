---
title: 模板方法模式：实现同一模板框架下的算法扩展
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2018-02-24 21:11:22 +0800
categories: [Design Pattern, behavioral]
tags:  [设计模式, Design Pattern, Template Method, 模板方法, 类行为型模式]
math: true
mermaid: true
---

模板方法模式的原理和代码实现都比较简单，也被广泛应用，但是因为使用继承机制，副作用往往盖过了主要作用，所以在使用时尤其要小心谨慎。
## 原理

模板方法模式原始定义是：在操作中定义算法的框架，将一些步骤推迟到子类中。模板方法让子类在不改变算法结构的情况下重新定义算法的某些步骤。

关键：解决算法框架这类特定的问题，同时明确表示需要使用继承的结构。

UML 图：

![](https://images.happymaya.cn/assert/design-patterns/template-uml.png)

模板方法模式包含的关键角色有两个：

1. **抽象父类：**定义一个算法所包含的所有步骤，并提供一些通用的方法逻辑；
2. **具体子类：**继承自抽象父类，根据需要重写父类提供的算法步骤中的某些步骤。

UML 对应代码实现：

```java
public abstract class AbstractClassTemplate {
    void step1(String key){
        //dosomthing
        System.out.println("=== 在模板类里 执行步骤 1");
        if (step2(key)) {
            step3();
        } else {
            step4();
        }
        step5();
    }
    boolean step2(String key){
        System.out.println("=== 在模板类里 执行步骤 2");
        if ("x".equals(key)) {
            return true;
        }
        return false;
    }
    abstract void step3();
    abstract void step4();
    void step5() {
        System.out.println("=== 在模板类里 执行步骤 5");
    }
    void run(String key){
        step1(key);
    }
}

public class ConcreteClassA extends AbstractClassTemplate {
    @Override
    void step3() {
        System.out.println("===在子类 A 中 执行：步骤3");
    }
    @Override
    void step4() {
        System.out.println("===在子类 A 中 执行：步骤4");
    }
}

public class ConcreteClassB extends AbstractClassTemplate {
    @Override
    void step3() {
        System.out.println("===在子类 B 中 执行：步骤3");
    }
    @Override
    void step4() {
        System.out.println("===在子类 B 中 执行：步骤4");
    }
}

public class Demo {
    public static void main(String[] args) {
        AbstractClassTemplate concreteClassA = new ConcreteClassA();
        concreteClassA.run("");
        System.out.println("===========");
        AbstractClassTemplate concreteClassB = new ConcreteClassB();
        concreteClassB.run("x");
    }
}
// 输出结果：
// === 在模板类里 执行步骤 1
// === 在模板类里 执行步骤 2
// ===在子类 A 中 执行：步骤4
// === 在模板类里 执行步骤 5
// ===========
// === 在模板类里 执行步骤 1
// === 在模板类里 执行步骤 2
// ===在子类 B 中 执行：步骤3
// === 在模板类里 执行步骤 5
```

模板方法模式的实现原理很简单，就是**一个父类下面的子类通过继承父类而使用通用的逻辑，同时根据各自需要优化其中某些步骤。**

## 使用模板方法的理由

使用模板方法模式的原因主要有两个：

1. **期望在一个通用的算法或流程框架下进行自定义开发。** 比如，在使用 Jenkins 的持续集成发布系统中，可以定制一个固定的 Jenkins Job 任务，将打包、发布、部署的流程作为一个通用的流程。对于不同的系统来说，只需要根据自身的需求增加步骤或删除步骤，就能轻松定制自己的持续发布流程；
2. **避免同样的代码逻辑进行重复编码。**比如，当调用 HTTP 接口时，我们会使用 HttpClient、OkHttp 等工具类进行二次开发，不过我们经常会遇见这样的情况，明明有人已经封装过同样的功能，但还是忍不住想要对这些工具类进行二次封装开发，实际上最终实现的功能都只是调用 HTTP 接口，这样“重复造轮子”会非常浪费开发时间。这时如果使用模板方法模式定义个统一的调用 HTTP 接口的逻辑，就能很好地避免重复编码。

## 使用场景

模板方法模式的使用场景一般有：

1. 多个类有相同的方法并且逻辑可以共用时；
2. 将通用的算法或固定流程设计为模板，在每一个具体的子类中再继续优化算法步骤或流程步骤时；
3. 重构超长代码时，发现某一个经常使用的公有方法。

假设设计一个简单的持续集成发布系统——研发部开发的代码放在 GitLab 上，并使用一个固定的发布流程来进行程序的上线发布，使用模板方法模式来定义一系列规范的流程。

首先，定义一个通用的流程框架类 DeployFlow，定义的步骤有六步，分别是： 从 GitLab 上拉取代码、编译打包、部署测试环境、测试、上传包到线上环境以及启动程序。

其中，从 GitLab 上拉取代码、编译打包、部署测试环境和测试这四个步骤，需要子类来进行实现，代码如下： 
```java
public abstract class DeployFlow {

    /* 使用final关键字来约束步骤不能轻易修改 */
    public final void buildFlow() {
        /* 从GitLab上拉取代码 */
        pullCodeFromGitlab();
        /* 编译打包 */
        compileAndPackage();
        /* 部署测试环境 */
        copyToTestServer();
        /* 测试 */
        testing();
        /* 上传包到线上环境 */
        copyToRemoteServer();
        /* 启动程序 */
        startApp();
    }

    public abstract void pullCodeFromGitlab();

    public abstract void compileAndPackage();

    public abstract void copyToTestServer();

    public abstract void testing();

    private void copyToRemoteServer() {
        System.out.println("统一自动上传 启动App包到对应线上服务器");
    }
    private void startApp() {
        System.out.println("统一自动 启动线上App");
    }
}
```

然后，分别实现两个子类： 实现本地的打包编译和上传以及实现全自动化的持续集成式的发布。
```java
/* 实现全自动化的持续集成式的发布 */
public class CicdDeployFlow extends DeployFlow {
    @Override
    public void pullCodeFromGitlab() {
        System.out.println("持续集成服务器将代码拉取到节点服务器上......");
    }
    @Override
    public void compileAndPackage() {
        System.out.println("自动进行编译&打包......");
    }
    @Override
    public void copyToTestServer() {
        System.out.println("自动将包拷贝到测试环境服务器......");
    }
    @Override
    public void testing() {
        System.out.println("执行自动化测试......");
    }
}

/* 实现本地的打包编译和上传 */
public class LocalDeployFlow extends DeployFlow{
    @Override
    public void pullCodeFromGitlab() {
        System.out.println("手动将代码拉取到本地电脑......");
    }
    @Override
    public void compileAndPackage() {
        System.out.println("在本地电脑上手动执行编译打包......");
    }
    @Override
    public void copyToTestServer() {
        System.out.println("手动通过 SSH 上传包到本地的测试服务......");
    }
    @Override
    public void testing() {
        System.out.println("执行手工测试......");
    }
}
```

最后，运行一个单元测试，测试一下本地发布 LocalDeployFlow 和持续集成发布 CicdDeployFlow： 
```java
package cn.happymaya.ndp.template_method.example;

public class Client {
    public static void main(String[] args) {
        System.out.println("开始本地手动发布流程======");
        DeployFlow localDeployFlow = new LocalDeployFlow();
        localDeployFlow.buildFlow();
        System.out.println("********************");
        System.out.println("开始 CICD 发布流程======");

        DeployFlow cicdDeployFlow = new CicdDeployFlow();
        cicdDeployFlow.buildFlow();
    }
}
// 输出结果
// 开始本地手动发布流程======
// 手动将代码拉取到本地电脑......
// 在本地电脑上手动执行编译打包......
// 手动通过 SSH 上传包到本地的测试服务......
// 执行手工测试......
// 统一自动上传 启动App包到对应线上服务器
// 统一自动 启动线上App
// ********************
// 开始 CICD 发布流程======
// 持续集成服务器将代码拉取到节点服务器上......
// 自动进行编译&打包......
// 自动将包拷贝到测试环境服务器......
// 执行自动化测试......
// 统一自动上传 启动App包到对应线上服务器
// 统一自动 启动线上App
```

模板方法模式应用场景的最大特征在于，通常是对算法的特定步骤进行优化，而不是对整个算法进行修改。一旦整体的算法框架被定义完成，子类便无法进行直接修改，因为子类的使用场景直接受到了父类场景的影响。 

## 优缺点

模板方法模式主要有以下两个优点。
1.  有效去除重复代码。 模板方法模式的父类保存通用的代码逻辑，这样可以让子类不再需要重复处理公用逻辑，只用关注特定的逻辑，从而起到去除子类中重复代码的目的。 
2.  有助于找到更通用的模板。 由于子类间重复的代码逻辑都会被抽取到父类中，父类也就慢慢变成了更通用的模板，这样有助于积累更多通用的模板，提升代码复用性和扩展性。 

模板方法模式的缺点：
1. 不符合开闭原则。 一个父类调用子类实现操作，通过子类扩展增加新的行为，但是子类执行的结果便会受到父类的影响，不符合开闭原则的“对修改关闭”；
2. 增加代码阅读的难度。 由于父类的某些步骤或方法被延迟到子类执行，那么需要跳转不同的子类阅读代码逻辑，如果子类的数量很多的话，跳转会很多，不方便联系上下文逻辑线索。而且模板方法中的步骤越多，其维护工作就可能会越困难。
3. 违反里氏替换原则。 虽然模板方法模式中的父类会提供通用的实现方法，但是延迟到子类的操作便会变成某种定制化的操作，一旦替换子类，可能会导致父类不可用或整体逻辑发生变化。
