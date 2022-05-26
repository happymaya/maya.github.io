---
title: 剖析一个 Spring Web 应用
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2018-03-02 17:32:00 +0800
categories: [Spring]
tags: [SpringBoot, Spring MVC]
---

在典型的 Web 应用程序中，前后端通常采用基于 HTTP 协议完成请求和响应。

HTTP 请求响应过程如下：

1. HTTP 请求；
2. URL 地址映射；
3. HHTP 请求的参数构建；
4. 对象（数据）序列化；
5. 各个服务自身内部的业务逻辑处理；
6. 对象（数据）反序列化；
7. HTTP 响应

基于 Spring MVC 完成上述开发流程所需要的开发步骤如下：
1. 使用 web.xml 定义 Spring 的 DispatcherServlet；
2. 完成启动 Spring MVC 的配置文件；
3. 编写响应 HTTP 请求的 Controller；
4. 将服务部署到 Tomcat Web 服务器

事实上，基于传统的 Spring MVC 框架开发 Web 应用逐渐暴露出一些问题，比较典型的就是**配置工作过于复杂和繁重**，以及**缺少必要的应用程序管理**和**监控机制**。

优化这一套开发过程，有几个点值得去挖掘，比方说**减少不必要的配置工作**、**启动依赖项的自动管理**、**简化部署**并**提供应用监控**等。这些优化点恰巧推动了以 Spring Boot 为代表的新一代开发框架的诞生，基于 Spring Boot 的开发流程见下：
1. 使用 `@SpringBootApplication` 注解创建服务启动类；
2. 编写响应 HTTP 请求的 Controller；
3. 脱离服务器独立运行服务并启动服务监控

作为 Spring 家族新的一员，Spring Boot 提供了令人兴奋的特性，这些特性的核心价值在于确保了开发过程的简单性，具体体现在**编码**、**配置**、**部署**、**监控**等多个方面。

优点如下:
1. 使编码更简单。只需在 Maven 中添加一项依赖并实现一个方法就可以提供微服务架构中所推崇的 RESTful 风格接口；
2. 使配置更简单。它把 Spring 中基于 XML 的功能配置方式转换为 Java Config，同时提供了 .yml 文件来优化原有基于 .properties 和 .xml 文件的配置方案，.yml 文件对配置信息的组织更为直观方便，语义也更为强大。同时，基于 Spring Boot 的自动配置特性，对常见的各种工具和框架均提供了默认的 starter 组件来简化配置；
3. 相较于传统模式下的 war 包，Spring Boot 部署包既包含了业务代码和各种第三方类库，同时也内嵌了 HTTP 容器。这种包结构支持 `java –jar application.jar` 方式的一键启动，不需要部署独立的应用服务器，通过默认内嵌 Tomcat 就可以运行整个应用程序。
4. 基于 Spring Boot 新提供的 Actuator 组件，可以通过 RESTful 接口获取应用程序的当前运行时状态并对这些状态背后的度量指标进行监控和报警。例如：
   1. 可以通过`/env/{name}`端点获取系统环境变量；
   2. 通过`/mapping`端点获取所有 RESTful 服务；
   3. 通过`/dump`端点获取线程工作状态；
   4. 通过`/metrics/{name}`端点获取 JVM 性能指标等


## 剖析一个 Spring Web 应用程序

针对一个基于 Spring Boot 开发的 Web 应用程序，其代码组织方式需要遵循一定的项目结构(我主要使用 Maven 来管理项目工程中的结构和包依赖)，一个典型的 Web 应用程序的项目结构如下：


在上图中，有几个地方需要特别注意，我也在图中做了专门的标注，分别是**包依赖**、**启动类**、**控制器类**以及**配置文件**。

### 包依赖

Spring Boot 提供了一系列 starter 工程来简化各种组件之间的依赖关系。以开发 Web 服务为例，我们需要引入 spring-boot-starter-web 这个工程，而这个工程中并没有具体的代码，只是包含了一些 pom 依赖，如下：
- org.springframework.boot:spring-boot-starter
- org.springframework.boot:spring-boot-starter-tomcat
- org.springframework.boot:spring-boot-starter-validation
- com.fasterxml.jackson.core:jackson-databind
- org.springframework:spring-web
- org.springframework:spring-webmvc

这里包括了传统 Spring MVC 应用程序中会使用到的 spring-web 和 spring-webmvc 组件，因此 Spring Boot 在底层实现上还是基于这两个组件完成对 Web 请求响应流程的构建。

如果使用 Spring Boot 2.2.4 版本，你会发现它所依赖的 Spring 组件都升级到了 5.X 版本。

在应用程序中引入 spring-boot-starter-web 组件就像引入一个普通的 Maven 依赖一样，如下所示。
```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

一旦 spring-boot-starter-web 组件引入完毕，就可以充分利用 Spring Boot 提供的自动配置机制开发 Web 应用程序。

### 启动类

使用 Spring Boot 的最重要的一个步骤是创建一个 Bootstrap 启动类。Bootstrap 类结构简单且比较固化，如下所示：
```java
package cn.happymaya.customer;

@SpringBootApplication
public class CustomerApplication {

	public static void main(String[] args) {
		SpringApplication.run(CustomerApplication.class, args);
	}
}

```

### 控制器类

Bootstrap 类提供了 Spring Boot 应用程序的入口，相当于应用程序已经有了最基本的骨架。接下来我们就可以添加 HTTP 请求的访问入口，表现在 Spring Boot 中也就是一系列的 Controller 类。这里的 Controller 与 Spring MVC 中的 Controller 在概念上是一致的，一个典型的 Controller 类如下所示：
```java
@RestController
@RequestMapping(value="customers")
public class CustomerController {
    
    @Autowired
    private CustomerTicketService customerTicketService; 
	
	@PostMapping(value = "/{accountId}/{orderNumber}")
	public CustomerTicket generateCustomerTicket( @PathVariable("accountId") Long accountId, @PathVariable("orderNumber") String orderNumber) {
		
		CustomerTicket customerTicket = customerTicketService.generateCustomerTicket(accountId, orderNumber);		
		
		return customerTicket;
	}
}
```
请注意，以上代码中包含了
- @RestController；用于指定请求地址的映射关系
- @RequestMapping；等同于指定了 GET 请求的 @RequestMapping 注解
- @GetMapping 这三个注解：是传统 Spring MVC 中所提供的 @Controller 注解的升级版，相当于就是 @Controller 和 @ResponseEntity 注解的结合体，会自动使用 JSON 实现序列化/反序列化操作


### 配置文件

在 src/main/resources 目录下存在一个 application.yml 文件，这就是 Spring Boot 中的主配置文件。例如，可以将如下所示的端口、服务名称以及数据库访问等配置信息添加到这个配置文件中：
```yaml
server:
  port: 8081

spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://119.3.52.175:3306/appointment
    username: root
    password: 1qazxsw2#edc   
```

事实上，Spring Boot 提供了强大的自动配置机制，如果没有特殊的配置需求，完全可以基于 Spring Boot 内置的配置体系完成诸如数据库访问相关配置信息的自动集成。