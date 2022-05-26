---
title:  Spring 依赖注入
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2021-03-02 23:33:00 +0800
categories: [Java, Spring]
tags:  [Spring, Ioc]
math: true
mermaid: true
---
# Spring 依赖注入

在 IDEA 中使用 Spring 框架时，经常遇到这样一个提示：`Field injection is not recommended !`

这个提示的意思是：不建议使用字段注入。

众所周知，Spring 的依赖注入有三大类，分别是：
- 字段注入
- 构造器注入
- Setter 方法注入

其中的字段注入就是把一个 Bean 以字段的形式注入到另一个 Bean 中，日常开发中这种注入方式用的非常多。既然字段注入这么常用，那为什么 IDEA 会不推荐使用呢？和其他注入类型相比，字段注入存在哪些问题呢？这就需要从 Spring 底层实现机制的角度分析这些问题，从而得知产生的原因。


如果想要创建一个对象，就需要知道如何**创建**、**使用**以及**销毁**这个对象。这个过程显然是繁杂而重复的。

这时候，可以把这部分工作交给一个容器，容器负责控制对象的生命周期和对象之间的关联关系。而 Spring 框架扮演的角色就是这样一个容器，如下图所示。

![](https://image.happymaya.cn/assert/blog/java/java-spring-ioc-1.png)

Spring 会在适当的时候创建一个 Bean，然后像注射器一样把它注入到目标对象中，这样就完成了对各个对象之间关系的控制。


那么，面试时关于依赖注入的一个基本问题就是：如何在 Spring 中注入对象呢？Spring 为开发人员提供了三种不同的依赖注入类型，分别是字段注入、构造器注入和 Setter 方法注入，如下图所示。

![](https://image.happymaya.cn/assert/blog/java/java-spring-ioc-2.png)

下面用代码示例来解释这三种依赖注入类型。假设，有如下所示的一个 HealthRecordService 接口以及它的实现类：

```
public interface HealthRecordService {
    public void recordUserHealthData();

}

public class HealthRecordServiceImpl implements HealthRecordService {

    @Override
    public void recordUserHealthData () {
        System.out.println("HealthRecordService has been called.");

    }
}
```

现在，如果让你把这个 HealthRecordServiceImpl 实现类注入到其他类中，我一般会在这样做：

#### 字段注入
首先，使用字段注入。想要在一个类中通过字段的形式注入某个对象，可以采用这样的方式：

```
public class ClientService {

    @Autowired
    private HealthRecordService healthRecordService;

    public void recordUserHealthData() {
        healthRecordService.recordUserHealthData();
    }
}
```

可以看到，通过 @Autowired 注解，字段注入的实现方式非常简单而直接，代码的可读性也很强。事实上，字段注入是三种注入方式中最常用、也是最容易使用的一种。但它也是三种注入方式中最应该避免使用的。就像开篇说的，在 IDEA 中使用字段注入时会遇到`Field injection is not recommended`的提示。这样的原因有三点：

1. **字段注入的最大问题是对象的外部可见性**
   ClientService 类中，通过定义一个私有变量 HealthRecordService 来注入该接口的实例。显然，这个实例只能在 ClientService 类中被访问，脱离了容器环境就无法访问这个实例，来看下面这段代码：
   ```java
   	ClientService clientService = new ClientService();
	clientService.recordUserHealthData (); 
   ```
   执行这段代码的结果就是抛出一个 NullPointerException 空指针异常，原因就在于无法在 ClientService 的外部实例化 HealthRecordService 对象。
   
   采用字段注入，类与容器的耦合度过高，无法脱离容器来使用目标对象。
   
   如果编写测试用例来验证 ClientService 类的正确性，那么想要使用 HealthRecordService 对象，就只能使用反射的方式，这种做法实际上是不符合 JavaBean 开发规范的，而且可能一直无法发现空指针异常的存在。

2. **字段注入的第二个问题是可能导致潜在的循环依赖**
   即两个类之间互相进行注入，例如下面这段示例代码：
   ```java
   	public class ClassA {
		@Autowired
		private ClassB classB;
	}
	public class ClassB {
		@Autowried
		private ClassA classA;
	}
   ```
   这里的 ClassA 和 ClassB 发生了循环依赖。上述代码在 Spring 中是合法的，容器启动时并不会报任何错误，而只有在使用到具体某个 ClassA 或 ClassB 时才会报错。

3. **字段注入的第三个问题是无法设置需要注入的对象为 final，也无法注入那些不可变对象**
   这是因为字段必须在类实例化时进行实例化。

基于以上三点，无论是 IDEA，还是 Spring 官方，都不推荐开发人员使用字段注入这种注入模式，而是推荐构造器注入。


#### 构造器注入

关于构造器注入，面试中往往会以这样的形式考察你：构造器是 Spring 官方推荐的依赖注入类型，你知道它有哪些特性吗？或者换种问法，构造器注入相比字段注入的优势在哪里？接下来，我们就一起来分析这些问题。

构造器注入的形式也很简单，就是通过类的构造函数来完成对象的注入，示例代码如下所示：
```java
public class ClientService {
    private HealthRecordService healthRecordService;

    @Autowired
    public ClientService(HealthRecordService healthRecordService) {
        this.healthRecordService = healthRecordService;
    }

    public void recordUserHealthData() {
        healthRecordService.recordUserHealthData();

    }
}
```

可以看到构造器注入能解决对象外部可见性的问题，因为 HealthRecordService 是通过 ClientService 构造函数进行注入的，所以势必可以脱离 ClientService 而独立存在。

关于构造器注入， Spring 官方文档是这样解释它的功能特性：

> The Spring team generally advocates constructor injection as it enables one to implement application components as *immutable objects* and to ensure that required dependencies are not null. Furthermore constructor-injected components are always returned to client (calling) code in a fully initialized state.

这段话的核心意思在于：**构造器注入能够保证注入的组件不可变，并且确保需要的依赖不为空**。这里的组件不可变也就意味着你可以使用 final 关键词来修饰所依赖的对象，而依赖不为空是指所传入的依赖对象肯定是一个实例对象，避免出现空指针异常。

同时，基于构造器注入，如果存在诸如前面介绍的 ClassA 和 ClassB 之间的循环依赖关系，如下所示：

```
public class ClassA {
	private ClassB classB;
	@Autowired
    public ClassA(ClassB classB) {
        this.classB = classB;
    }

}

public class ClassB {
	private ClassA classA;

	@Autowired
    public ClassB(ClassA classA) {
        this.classA = classA;

    }
}

```

那么在 Spring 容器启动的时候，就会抛出一个循环依赖异常，从而提醒避免循环依赖。

如此看来，字段注入的三大问题都可以通过使用构造器注入的方式来解决。

但是，当构造函数中存在较多依赖对象的时候，大量的构造器参数会让代码显得比较冗长。假设一个类的构造器需要 10 个参数，那么想要使用这个类时，就需要事先准备好这 10 个参数，并严格按照构造器指定的顺序一一进行传入。

那么，无论从代码的可读性还是可维护角度而言，这都不是很符合最佳实践。这时候就可以引入 Setter 方法注入。


#### Setter 方法注入

它的实现代码如下所示：

```java
public class ClientService {
    private HealthRecordService healthRecordService;

    @Autowired
    public void setHealthRecordService(HealthRecordService healthRecordService) {
        this.healthRecordService = healthRecordService;

    }

    public void recordUserHealthData() {
        healthRecordService.recordUserHealthData();
    }
}
```

Setter 方法注入和构造器注入看上去有点类似，但它比构造函数更具可读性，因为可以把多个依赖对象分别通过 Setter 方法逐一进行注入。而且，Setter 方法注入对于非强制依赖项注入很有用，我们可以有选择地注入一部分想要注入的依赖对象。换句话说，可以实现按需注入，帮助我们只在需要时注入依赖关系。

另一方面，Setter 方法可以很好解决应用程序中的循环依赖问题，如下所示的代码是可以正确执行的：

```java
public class ClassA {
	private ClassB classB;

	@Autowired
    public void setClassB(ClassB classB) {
        this.classB = classB;
    }
}

public class ClassB {

	private ClassA classA;

	@Autowired
    public void setClassA(ClassA classA) {
        this.classA = classA;
    }
}

```
> 请注意，上述代码能够正确执行的前提是 ClassA 和 ClassB 的作用域都是“Singleton”。


最后，通过 Setter 注入，可以对依赖对象进行多次重复注入，这在构造器注入中是无法实现的。

那么，概括起来就是：

- 构造器注入适用于强制对象注入；
- Setter 注入适合于可选对象注入；
- 字段注入应该避免，因为对象无法脱离容器而独立运行。

好了，现在你能回答出三种依赖注入类型的相关内容了，那么放到一些场景中，该怎么灵活使用呢？这也是面试中会重点考察的问题。

### 依赖注入用得好，Spring 框架轻松搞

- 把一个 Bean 注入到 Spring 容器中，如何合理设置它的作用域？
- 在实现依赖注入时，想要针对一个接口注入多个实现类，你有什么办法？
- Bean 的注入是需要消耗性能的，那么如何高效完成这一过程呢？

#### 把握好 Bean 的作用域

作用域描述了 Bean 在 Spring IoC 容器上下文中的生命周期和可见性。

通过注解来设置 Bean 的作用域，可以使用这样的示例代码：

```java
@Configuration
public class AppConfig {          

    @Bean
	@Scope("singleton")
    public HealthRecordService createHealthRecordService() {
        return new HealthRecordServiceImpl();
    }
}

```

可以看到这里使用了一个 @Scope 注解来指定 Bean 的作用域为单例的“singleton”。

在 Spring 中，除了单例作用域之外，还有一个“prototype”，即原型作用域，也可以称为多例作用域，以示与单例进行区别。使用方式上，同样可以使用枚举值来对它们进行设置：

```java
@Scope(value = ConfigurableBeanFactory.SCOPE_SINGLETON) 
@Scope(value = ConfigurableBeanFactory.SCOPE_PROTOTYPE)
```

在 Spring IoC 容器中，Bean 的默认作用域是单例，也就是说不管对 Bean 的引用有多少个，容器只会创建一个实例。而原型作用域则不同，每次请求 Bean 时，Spring IoC 容器都会创建一个新的对象实例。


从两种作用域的效果来看，总结一条开发上的结论：对于无状态的 Bean，应该使用单例作用域，反之应该使用原型作用域。

那么，什么样的 Bean 是属于有状态的呢？结合 Web 应用程序，我们可以明确对于每次 HTTP 请求而言，都应该创建一个 Bean 来代表这一次的请求对象。同样，对于会话而言，也需要针对每个会话创建一个会话状态对象。这些都是常见的有状态的 Bean。


#### 怎样灵活使用注解配置？

在使用 Spring 依赖注入类型时，通常可以使用 XML 配置、Java 代码配置以及注解配置这三种实现方式。随着 Spring Boot 框架的流行，使用注解配置已经成为目前最主流的开发方式。除了前面已经给出的最常见的 @Autowired 注解，Spring 框架还提供了一组非常有用的注解帮助更好地管理所注入的对象，包括 `@Primary` 注解和 `@Qualifier `注解。

在 Spring IoC 容器中，针对 HealthRecordService 这样一种接口类型，原则上只允许注入一个实现类。如果存在该类型的多个对象实例，那么容器就会报 NoUniqueBeanDefinitionException，意味着容器无法决定选择哪一个实例来进行注入。这时候就可以使用 @Primary 注解来帮助容器做出选择，该注解使用方式如下所示：

```java
@Component
public class HealthRecordServiceImplA implements HealthRecordService {
    …
}

@Component
@Primary
public class HealthRecordServiceImplB implements HealthRecordService {
    …
}
```
现在，Spring IoC 容器只会注入 HealthRecordServiceImplB 这个实例类，这在管理针对某种类型的多个实例时非常有用。
和 @Primary 注解的应用场景类似，@Qualifier 注解为我们如何选择实例类进行注入提供了更加灵活的实现方式，如下所示：
```
@Component
@Qualifier("healthRecordServiceA")
public class HealthRecordServiceImplA implements HealthRecordService {
    …
}

@Component
@Qualifier("healthRecordServiceB")
public class HealthRecordServiceImplB implements HealthRecordService {
    …
}
```
可以看到这里针对不同的实现类，我们通过 @Qualifier 注解设置了不同的名称，这样在使用时就可以通过该名称获取不同的实例：
```java
@Autowired
@Qualifier("healthRecordServiceB")
private HealthRecordService healthRecordService;
```

#### 不同配置的性能有哪些异同？


在 Spring 中，可以通过设置组件扫描范围来简化 JavaBean 的注入配置。因为任何的类都是位于某一个包结构之下，所以 Spring 提供了一个 @ComponentScan 注解，该注解在需要大规模对象注入的场景下非常有用，其基本用法如下所示：

```java
@Configuration 
@ComponentScan(basePackages="com.lagou.spring") 
public class AppConfig { }
```

在这个示例中，Spring 会扫描由 basePackages 指定的包路径 "cn.happymay.spring" 及其子路径下的所有 Bean，并把它们注入到容器中。当然，首先需要在这些类上添加 @Component 注解以及由该注解衍生的 @Service、@Repository、@Controller 等注解。


然而，因为该注解会扫描 basePackages 指定包中的所有组件，所以如果所指定包中的组件并不需要在应用程序启动时就全部加载到容器中，那么对包路径进行精细化设计就是一项最佳实践。例如，我们可以通过设置一个列表来细化具体的包结构路径，如下所示：

```java
@Configuration 
@ComponentScan(basePackages="com.lagou.spring.service", "com.lagou.spring.controller") 
public class AppConfig { }
```

#### 单例模式和原型模式对性能的影响

在 Spring 中，默认情况下定义的所有 Bean 都是单例的。但当把 Bean 范围设置为“prototype”，每次请求 Bean 时，Spring IoC 容器都会创建一个新的对象实例。所以，使用原型模式在创建过程中会对性能产生影响，对那些初始化过程需要消耗巨大资源的对象而言尤其如此，这类对象常见的包括网络连接对象、数据库连接对象等。因此，针对这些对象，应该完全避免使用原型模式。或者，你应该仔细设计并对性能进行充分测试。

#### Spring IoC 容器的延迟加载（Lazy Loading）和预加载（Preloading）机制

通过 @Autowired 注入的 Bean 都是在 Spring IoC 容器启动时被创建和初始化的，这个过程被称为预加载。但有时候，希望能够延迟 Bean 的加载时机，这时候就可以使用 @Lazy 注解，使用方法如下所示：

```java
@Component
@Lazy
public class HealthRecordServiceImpl implements HealthRecordService {
    …
}
```

添加了 `@Lazy` 注解的效果在于只有在使用到这个 Bean 的时候才会去初始化，而不是在 Spring IoC 容器启动时，这样就可以节省容器资源。

延迟加载确保在请求时动态加载 Bean，预加载确保在使用 Bean 之前加载 Bean。Spring IoC 容器默认使用预加载。然而，在容器启动时就加载所有类（即使它们没有被使用）并不是一个明智的决定，因为有些 Bean 实例会非常消耗资源。

所以应该根据实际情况选择具体的加载方法。如果需要尽可能快地加载应用程序，那么就采用延迟加载；如果需要应用程序尽可能快地运行并更快地为请求提供服务，那么就进行预加载。
