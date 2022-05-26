---
title: Spring 技术体系
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2018-03-02 17:32:00 +0800
categories: [Spring]
tags: [Spring]
---

Spring 框架自 2003 年由 Rod Johnson 设计并实现以来，经历了多个重大版本的发展和演进，已经形成了一个庞大的家族式技术生态圈。

目前，Spring 已经是 Java EE 领域最流行的开发框架，在全球各大企业中都得到了广泛应用。

## Spring 家族技术生态全景图

![](https://images.happymaya.cn/assert/spring-boot/what_spring_can_do.png)

通过来自 [Spring 官网主页](https://spring.io)中的图可以看到，Spring 框架的七大核心技术体系：
1. [微服务](https://spring.io/microservices)
2. [响应式编程](https://spring.io/reactive)
3. [云原生](https://spring.io/cloud)
4. [Web 应用](https://spring.io/web-applications)
5. [Serverless 架构](https://spring.io/serverless)
6. [事件驱动](https://spring.io/event-driven)
7. [批处理](https://spring.io/batch)

这些技术体系虽各自独立，但也有一定交集，例如：
- 微服务架构往往会与基于 Spring Cloud 的云原生技术结合在一起使用，并且微服务架构的构建过程也需要依赖于能够提供 RESTful 风格的 Web 应用程序等；

另一方面，在具备特定的技术特点之外，这些技术体系也各有其应用场景。例如：
- 实现日常报表等轻量级的批处理任务，而又不想引入 Hadoop 这套庞大的离线处理平台时，使用基于 Spring Batch 的批处理框架是一个不错的选择；
- 实现与 Kafka、RabbitMQ 等各种主流消息中间件之间的集成，但又希望开发人员不需要了解这些中间件在使用上的差别，那么使用基于 Spring Cloud Stream 的事件驱动架构是首选，因为这个框架对外提供了统一的 API，从而屏蔽了内部各个中间件在实现上的差异性。

这里不对 Spring 中的所有七大技术体系做全面的展开。在日常开发过程中：
- 如果构建单体 Web 服务，可以采用 Spring Boot；
- 如果想要开发微服务架构，则采用基于 Spring Boot 的 Spring Cloud，而 Spring Cloud 同样内置了基于 Spring Cloud Stream 的事件驱动架构。
- 同时，响应式编程是 Spring 5 引入的最大创新，代表了一种系统架构设计和实现的技术方向。

现在的 Spring 家族技术体系都是在 Spring Framework 基础上逐步演进而来的。Spring Framework 的整体架构，如下图所示：
![Spring Framework Runtime](https://img-blog.csdnimg.cn/2020031110534271.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2hlbGxvX3dvcmQy,size_16,color_FFFFFF,t_70)

Spring 从诞生之初就被认为是一种容器，上图中的 “核心容器" 部分就包含了一个容器所应该具备的核心功能，包括：
- 基于依赖注入机制的 JavaBean 处理；
- 面向切面 AOP；
- 上下文 Context；
- Spring 自身所提供的表达式工具等一些辅助功能。

图中最上面的两个框就是构建应用程序所需要的最核心的两大功能组件，也是日常开发中最常用的组件，即**数据访问**和** Web 服务**。

这两大部分功能组件中包含的内容非常多，而且充分体现了 Spring Framework 的集成性，也就是说，框架内部整合了业界主流的**数据库驱动**、**消息中间件**、**ORM 框架**等各种工具，我们根据需要灵活地替换和调整自己想要使用的工具。

从开发语言上讲，虽然 Spring 应用最广泛的是在 Java EE 领域，但在当前的版本中，也支持 **Kotlin****、Groovy** 以及各种**动态开发语言**。


## Spring Boot 与 Web 应用程序

Spring Boot 构建在 Spring Framework 基础之上，是新一代的 Web 应用程序开发框架。可以通过下面这张图来了解 Spring Boot 的全貌：
![Spring Boot 整体架构图](https://images.happymaya.cn/assert/spring-boot/spring-boot-core.png)

通过浏览 Spring 的官方网站，可以看到 Spring Boot 已经成为 Spring 中顶级的子项目。自 2014 年 4 月发布 1.0.0 版本以来，Spring Boot 俨然已经发展为 Java EE 领域开发 Web 应用程序的首选框架。


## Spring Cloud 与微服务架构

Spring Cloud 构建在 Spring Boot 基础之上，它的整体架构图如下所示：
![Spring Cloud 与微服务整体架构图（来自 Spring 官网）](https://spring.io/images/diagram-microservices-88e01c7d34c688cb49556435c130d352.svg)
技术组件的完备性是 Spring Cloud 框架的主要优势，它集成了业界一大批知名的微服务开发组件。例如：
- 服务治理：Spring Cloud Netfilx Eureka
- 服务容错：Spring Cloud Circuit Breaker
- 服务网关：Spring Cloud Gateway
- 配置中心：Spring Cloud Config
- 事件驱动：Spring Cloud Stream
- 服务安全：Spring Cloud Security
- 链路跟踪：Spring Cloud Sleuth
- 服务测试：Spring Cloud Contract
- 等等.....

基于 Spring Boot 的开发便利性，Spring Cloud 巧妙地简化了微服务系统基础设施的开发过程，Spring Cloud 包含上图中所展示的**服务发现注册**、**API 网关**、**配置中心**、**消息总线**、**负载均衡**、**熔断器**、**数据监控**等。

## Spring 5 与响应式编程

随着 Spring 5 的正式发布，我们迎来了响应式编程（Reactive Programming）的全新发展时期。Spring 5 中内嵌了与数据管理相关的响应式数据访问、与系统集成相关的响应式消息通信以及与 Web 服务相关的响应式 Web 框架等多种响应式组件，从而极大地简化了响应式应用程序的开发过程和开发难度。

下图展示了响应式编程的技术栈与传统的 Servlet 技术栈之间的对比：
![Spring Boot 2 VS Reactor](https://spring.io/images/diagram-reactive-1290533f3f01ec9c57baf2cc9ea9fa2f.svg)

从上图可以看到，上图左侧为基于 Spring WebFlux 的技术栈，右侧为基于 Spring MVC 的技术栈。

传统的 Spring MVC 构建在 Java EE 的 Servlet 标准之上，该标准本身就是**阻塞式**和**同步的**，而 Spring WebFlux 基于**响应式流**，因此可以用来**构建异步非阻塞的服务**。

在 Spring 5 中，选取了 Project Reactor 作为响应式流的实现库。由于响应式编程的特性，Spring WebFlux 和 Project Reactor 的运行需要依赖于诸如 Netty 和 Undertow 等支持异步机制的容器。

同时也可以选择使用较新版本的 Tomcat 和 Jetty 作为运行环境，因为它们支持异步 I/O 的 Servlet 3.1。下图更加明显地展示了 Spring MVC 和 Spring WebFlux 之间的区别和联系：
![Spring MVC 和 Spring WebFlux 的关联关系图](https://s0.lgstatic.com/i/image/M00/70/ED/Ciqc1F-8pB6AReQhAADiHs1UMA4354.png)