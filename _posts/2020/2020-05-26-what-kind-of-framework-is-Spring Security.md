---
title:  初识 Spring Security 
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2020-05-26 21:13:11 +0800
categories: [Java, Spring]
tags:  [Spring Security]
math: true
mermaid: true
---

Spring Security 是 Spring 家族中历史悠久的框架，具备完整强大的功能体系。本文围绕它的功能体系进行总结梳理。

其实，在 Sping Boot 出现之前，Spring Security 就已经诞生多年了。但 Spring Security 发展一直不顺利。主要问题在于：

- 继承、配置 Spring Secutiy 框架过程比较复杂。


但是随着 Spring Boot 的兴起，基于 Spring Boot 所提供的针对 Spring Security 的自动配置方案，可以零配置使用 Spring Security 。
​
如果想要在 Spring Boot 中使用 Spring Security ，只需要在 Maven 工程的 pom 文件中添加如下依赖：
```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-security</artifactId>
</dependency>
```
下面，通过构建一个简单的 HTTP 端点，来演示下 Spring Security 的初步使用。
```java
@RestController
public class DemoController {

    @GetMapping("/hello")
    public String hello() {
        return "Hello World!";
    }
}

```

启动这个 Spring Boot 应用程序，然后通过浏览器访问 "/hello" 端点。可能希望得到的是 "Hello World!" 这个输出结果，但事实上，浏览器跳转到一个如下所示的登录页面：
​

![image.png](https://cdn.nlark.com/yuque/0/2021/png/12442250/1628949521145-3f8986c8-7d50-4d0d-b6ac-2ef615d60a5f.png#crop=0&crop=0&crop=1&crop=1&height=174&id=PJd5G&margin=%5Bobject%20Object%5D&name=image.png&originHeight=694&originWidth=1663&originalType=binary&ratio=1&rotation=0&showTitle=false&size=64575&status=done&style=none&title=&width=416)


会弹出这个登录页面原因在于添加了 Spring Security 依赖之后，Spring Security 为应用程序**自动嵌入了用户认证机制**。


接下来，就围绕着这个登录场景，分析下如何获取登录所需的用户名和密码。注意到 Spring Boot 的控制台启动日志中，出现了如图下所示的一行日志：
![image.png](https://cdn.nlark.com/yuque/0/2021/png/12442250/1628949382586-2bdd2d9a-2a67-4c55-9287-c2ace375419d.png#crop=0&crop=0&crop=1&crop=1&height=231&id=SL54q&margin=%5Bobject%20Object%5D&name=image.png&originHeight=924&originWidth=2244&originalType=binary&ratio=1&rotation=0&showTitle=false&size=355962&status=done&style=none&title=&width=561)


这行日志就是 Spring Security 生产的一个密码，而用户名则是系统默认的 "user"。通过输入正确的用户名和密码，浏览器就会输出 "Hello World！"这个响应结果。


上面就是 Spring Security 提供的认证功能，也是 Spring Security 众多功能中的一项基础功能。接下来就深挖一下 Spring Security 中的完整功能体系。


## 2. Spring Security 功能体系


Spring Secturity 提供的是一整套完整的安全性解决方案。面向不同的业务需求和应用场景，Spring Security 分别提供了对应的安全性能。下面分别从**单体应用、微服务架构以及响应式系统**这三个维度对这些功能展开探讨和总结。


### 2.1 Spring Security 与单体应用


在软件系统中，把需要访问的内容称为**资源（Resource）, **而安全性设计的目的是对这些资源进行保护，确保它们的访问是安全性的。例如：在一个 Web 应用程序中，对外暴露的 HTTP 端点就可以被理解为是资源。
​

对于资源的安全性访问，业界存在一些常见的技术体系，这些技术体系中都存在既容易理解又容易混淆的两个概念：

- **认证（Authentication）**
- **授权（Authorization）**

​

认证和授权结合起来，**构成了对系统中的资源进行安全性管理的最常见的解决方案 **：

1. 判断资源访问者的有效身份
1. 确定其是否有对这个资源进行访问的合法权限

这个流程如下图所示：
![基于认证和授权机制的资源访问安全性示意图.png](https://cdn.nlark.com/yuque/0/2021/png/12442250/1628950663193-be8c2579-a3ac-49df-891e-1c0e8fcb8911.png#crop=0&crop=0&crop=1&crop=1&height=236&id=Y6MIe&margin=%5Bobject%20Object%5D&name=%E5%9F%BA%E4%BA%8E%E8%AE%A4%E8%AF%81%E5%92%8C%E6%8E%88%E6%9D%83%E6%9C%BA%E5%88%B6%E7%9A%84%E8%B5%84%E6%BA%90%E8%AE%BF%E9%97%AE%E5%AE%89%E5%85%A8%E6%80%A7%E7%A4%BA%E6%84%8F%E5%9B%BE.png&originHeight=942&originWidth=1424&originalType=binary&ratio=1&rotation=0&showTitle=true&size=72188&status=done&style=none&title=%E5%9F%BA%E4%BA%8E%E8%AE%A4%E8%AF%81%E5%92%8C%E6%8E%88%E6%9D%83%E6%9C%BA%E5%88%B6%E7%9A%84%E8%B5%84%E6%BA%90%E8%AE%BF%E9%97%AE%E5%AE%89%E5%85%A8%E6%80%A7%E7%A4%BA%E6%84%8F%E5%9B%BE&width=356 "基于认证和授权机制的资源访问安全性示意图")
​

上图代表一种通用方案，该方案面对不同应用场景和技术体系可以衍生出很多具体的实现策略。
​

Web 应用系统中的认证和授权模型与上图相似。但在具体设计和实现过程中也有其特殊性。
​


- 针对认证。这部分的需求相对比较明显，有以下两个要点
   - 构建一套完整的存储体系，保存和维护用户信息，
   - 确保在处理请求过程中合理利用这些用户信息。
- 针对授权。情况相对认证较复杂。对于某一特定 Web 应用程序而言，面临两个主要问题：
   - 判断一个 HTTP 请求是否具备访问权限
   - 对访问进行精细化管理（虽然一个请求具备访问该应用程序的权限，但是不意味着能够访问所有 HTTP 端点，某些核心功能需要具备较高的权限才能访问，而有些则不需要）

如下图所示：
![Web 应用程序访问授权效果示意图.png](https://cdn.nlark.com/yuque/0/2021/png/12442250/1628951048023-a9c8d51f-466e-4688-bdf7-a83f12289066.png#crop=0&crop=0&crop=1&crop=1&height=323&id=h1snU&margin=%5Bobject%20Object%5D&name=Web%20%E5%BA%94%E7%94%A8%E7%A8%8B%E5%BA%8F%E8%AE%BF%E9%97%AE%E6%8E%88%E6%9D%83%E6%95%88%E6%9E%9C%E7%A4%BA%E6%84%8F%E5%9B%BE.png&originHeight=1292&originWidth=2020&originalType=binary&ratio=1&rotation=0&showTitle=true&size=132449&status=done&style=none&title=Web%20%E5%BA%94%E7%94%A8%E7%A8%8B%E5%BA%8F%E8%AE%BF%E9%97%AE%E6%8E%88%E6%9D%83%E6%95%88%E6%9E%9C%E7%A4%BA%E6%84%8F%E5%9B%BE&width=505 "Web 应用程序访问授权效果示意图")
​

在上图中，假设该请求具备对应用程序中端点2、3、4 的访问权限，但是不具备访问端点 1 的权限。想要达到这个效果，一般的做法**引入角色体系**

- 针对不同的用户设置不同等级的角色
- 角色等级对应的访问权限等级
- 每一个请求绑定到某一个角色，也就具备了某一个权限的访问



接下来，把认证和授权结合起来，梳理出 Web 应用程序访问场景下的安全性实现方案，如下图所示：
![单体服务下的认证和授权整合示意图 (1).png](https://cdn.nlark.com/yuque/0/2021/png/12442250/1628951456284-1310bf7b-80fa-45a4-93ca-437d961d98e7.png#crop=0&crop=0&crop=1&crop=1&height=1069&id=gRzlm&margin=%5Bobject%20Object%5D&name=%E5%8D%95%E4%BD%93%E6%9C%8D%E5%8A%A1%E4%B8%8B%E7%9A%84%E8%AE%A4%E8%AF%81%E5%92%8C%E6%8E%88%E6%9D%83%E6%95%B4%E5%90%88%E7%A4%BA%E6%84%8F%E5%9B%BE%20%281%29.png&originHeight=1069&originWidth=1908&originalType=binary&ratio=1&rotation=0&showTitle=true&size=102114&status=done&style=none&title=%E5%8D%95%E4%BD%93%E6%9C%8D%E5%8A%A1%E4%B8%8B%E7%9A%84%E8%AE%A4%E8%AF%81%E5%92%8C%E6%8E%88%E6%9D%83%E6%95%B4%E5%90%88%E7%A4%BA%E6%84%8F%E5%9B%BE&width=1908 "单体服务下的认证和授权整合示意图")
结合示意图可以看到：

- 通过请求传递用户凭证完成用户认证
- 根据该用户信息中具备的角色信息获取访问权限，
- 最终完成对 HTTP 端点的访问授权



围绕认证和授权，还需要一系列的额外功能确保整个流程得以实现。这些功能包括：

- 用于密码保护的加解密机制
- 用于实现方法级的安全访问
- 支持跨域等

这些功能在以后再总结。


### 2.2 Spring Security 与微服务架构
​

微服务架构的情况要比单体应用复杂的很多，因为涉及了服务与服务之间的调用关系。
​

上述中资源，对应到微服务架构中：

- 服务提供者充当的角色是资源服务器
- 服务消费者充当的角色是客户端

因此，服务既可以是客户端，也可以是资源服务器。
​

同样将认证和授权结合起来，微服务访问场景下的安全实现方案，如下图所示：
![微服务架构下的认证和授权整合示意图.png](https://cdn.nlark.com/yuque/0/2021/png/12442250/1628951870879-863ae1ae-97d0-4ad0-adc2-e9d26335e5dd.png#crop=0&crop=0&crop=1&crop=1&height=413&id=TkNwk&margin=%5Bobject%20Object%5D&name=%E5%BE%AE%E6%9C%8D%E5%8A%A1%E6%9E%B6%E6%9E%84%E4%B8%8B%E7%9A%84%E8%AE%A4%E8%AF%81%E5%92%8C%E6%8E%88%E6%9D%83%E6%95%B4%E5%90%88%E7%A4%BA%E6%84%8F%E5%9B%BE.png&originHeight=1651&originWidth=3113&originalType=binary&ratio=1&rotation=0&showTitle=false&size=311056&status=done&style=none&title=&width=778)
微服务架构下的认证和授权整合示意图


可以看到，与单体应用相比，微服务架构把认证和授权的过程进行集中化管理，所以在上图中出现了一个授权中心

1. 授权中心获取客户端请求中带有的身份凭证信息
1. 基于凭证信息生成一个 Token（包含了权限范围和有效期）
1. 客户端获取 Token 后，基于该 Token 发起对微服务的访问。
1. 资源服务器对客户端获取的 Token 进行认证
1. 资源服务器根据 Token 的权限范围和有效期从授权中心获取该请求能够访问的特定资源

在微服务系统中，对外的资源表现形式同样可以理解为一个个 HTTP 端点。
​

上图中关键点就在于构建用于生成和验证 Token 的授权中心，为此需要引入OAuth2 协议。
​

OAuth2 协议为客户端程序和资源服务器之间设置了一个授权层，确保 Token 能够在各个微服务中进行有效传递，如下图所示：
![OAuth2 协议在服务访问场景中的应用.png](https://cdn.nlark.com/yuque/0/2021/png/12442250/1638198782343-6e9b8e40-416b-4501-9551-49ec4fa530e0.png#clientId=u74fba880-e6d1-4&crop=0&crop=0&crop=1&crop=1&from=ui&id=ub744ede9&margin=%5Bobject%20Object%5D&name=OAuth2%20%E5%8D%8F%E8%AE%AE%E5%9C%A8%E6%9C%8D%E5%8A%A1%E8%AE%BF%E9%97%AE%E5%9C%BA%E6%99%AF%E4%B8%AD%E7%9A%84%E5%BA%94%E7%94%A8.png&originHeight=346&originWidth=953&originalType=binary&ratio=1&rotation=0&showTitle=false&size=26759&status=done&style=none&taskId=u3ee8ba0e-808c-4d6a-8920-8589e4b3526&title=)


OAuth2 是一个相对复杂的协议。

- 综合应用摘要认证、签名认证、HTTPS 等安全性手段，提
- 供 Token 生成和校验以及公私钥管理等功能
- 权限粒度控制

​

应当避免自行实现这类复杂协议，倾向于借助于特定工具，以免重复造轮子，Spring Security 就提供了实现这一协议的完整解决方案，可以使用该框架完成适用于微服务系统中的认证授权机制。
​

> PS ：单体架构只需要用户登陆以及对用户调用某个接口时进行鉴权即可。微服务的话，多了其它服务(应用而非用户)来调用接口，需要判断这个上游服务有没有权限来调用这个接口



### 2.2 Spring Security 与响应式系统


随着 Spring 5 的发布，迎来了响应式编程（Reactive Programming）的全新发展时期。响应式编程是 Spring 5 最核心的新功能，也是 Spring 家族目前重点推广的技术体系。Spring 5 的响应式编程模型以 Project Reactor 库为基础，后者则实现了响应式流规范。
事实上，Spring Boot 从 2.x 版本开始也全面依赖 Spring 5。同样，在 Spring Security 中，用户账户体系的建立、用户认证和授权、方法级别的安全访问、OAuth2 协议等传统开发模式下具备的安全性功能都具备对应的响应式版本。














