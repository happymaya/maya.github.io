---
title: Spring 安全体系的整体架构
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2018-03-07 17:32:00 +0800
categories: [Spring]
tags: [SpringBoot, Spring Security]
---

在设计 Web 应用程序时：
1. 一方面，因为缺乏对 Web 安全访问机制的认识，所以系统安全性是一个重要但又容易被忽略的话题；
2. 另一方面，因为系统涉及的技术体系非常复杂，所以系统安全性又是一个非常综合的话题。

在 Spring 家族中，Spring Security 专门提供了一个安全性开发框架。


## Web 应用程序的安全性需求

在软件系统中，把需要访问的内容定义为一种**资源（Resource）**，而**安全性设计的核心目标是对这些资源进行保护**，以此确保外部请求对它们的访问安全受控。

在一个 Web 应用程序中，把对外暴露的 RESTful 端点理解为资源，关于如何对 HTTP 端点这些资源进行安全性访问，业界存在一些常见的技术体系。

而安全领域中非常常见但又容易混淆的两个概念：**认证（Authentication）**和**授权（Authorization）**。

1. 认证，首先需要明确`你是谁`这个问题，也就是说系统能针对每次访问请求判断出访问者是否具有合法的身份标识；
2. 一旦明确了 “你是谁”，就能判断出“你能做什么”，这个步骤就是**授权**。

一般来说，**通用的授权模型**都是基于**权限管理体系**，即对**资源**、**权限**、**角色**和**用户**的进行组合处理的一种方案。

当把认证与授权结合起来后，即**先判断资源访问者的有效身份**，然后**确定其对这个资源进行访问的合法权限**，整个过程就形成了对系统进行安全性管理的一种常见解决方案，如下图所示：
![基于认证和授权机制的资源访问安全性示意图](http://processon.com/chart_image/60f58720637689739c3ea826.png)

上图就是一种通用方案，而在不同的应用场景及技术体系下，系统可以衍生出很多具体的实现策略，比如 Web 应用系统中的认证和授权模型虽然与上图类似，但是在具体设计和实现过程中有其特殊性。

在 Web 应用体系中，
1. **认证**这部分的需求相对比较明确，所以需要构建**一套完整的存储体系来保存和维护用户信息**，并且**确保这些用户信息在处理请求的过程中能够得到合理利用**。
2. **授权**的情况相对来说复杂些，比如：对某个特定的 Web 应用程序而言，面临的第一个问题是如何判断一个 HTTP 请求具备访问自己的权限。解决完这个第一个问题后，就算这个请求具备访问该应用程序的权限，并不意味着它能够访问其所具有的所有 HTTP 端点，比如业务上的某些核心功能还是需要具备较高的权限才能访问，这就涉及需要解决的第二个问题——**如何对访问的权限进行精细化管理**？

如下图所示：
![Web 应用程序访问授权效果示意图](http://processon.com/chart_image/60f588770e3e74539278b400.png)

在上图中，假设该请求具备对 Web 应用程序的访问权限，但不具备访问应用程序中端点 1 的权限，如果想实现这种效果，一般我们的做法是引入角色体系：首先对不同的用户设置不同等级的角色（即角色等级不同对应的访问权限也不同），再把每个请求绑定到某个角色（即该请求具备了访问权限）。

接下来把认证和授权进行结合，梳理出了 Web 应用程序访问场景下的安全性实现方案，如下图所示：
![认证和授权整合示意图](http://processon.com/chart_image/60f589d97d9c087bac5f6bb0.png)

从上图可以看到，用户首先通过请求传递用户凭证完成用户认证，然后根据该用户信息中所具备的角色信息获取访问权限，最终完成对 HTTP 端点的访问授权。

对一个 Web 应用程序进行安全性设计时：
1. 首先需要考虑认证和授权，因为它们是核心考虑点；
2. 在技术实现场景中，只要涉及用户认证，势必会涉及用户密码等敏感信息的加密；
3. 针对用户密码的场景，主要使用**单向散列加密算法**对**敏感信息**进行加密。

关于**单向散列加密算法**，它常用于**生成消息摘要（Message Digest）**，主要特点为：
- **单向不可逆和密文长度固定，同时具备“碰撞”少的优点，即明文的微小差异会导致生成的密文完全不同。**

其中，常见的**单向散列加密实现算法为 MD5（Message Digest 5）**和 **SHA（Secure Hash Algorithm）**。

而在 JDK 自带的 MessageDigest 类中，因为它已经包含了这些算法的默认实现，所以直接调用方法即可。

在日常开发过程中，对于密码进行加密的典型操作时序图如下所示：
![单向散列加密与加盐机制](http://processon.com/chart_image/6288f7710e3e74749fb46652.png)

上图中，引入了**加盐（Salt）机制**，进一步提升了加密数据的安全性。

所谓加盐就是在**初始化明文数据时**，系统自动往明文中添加一些附加数据，然后再进行散列。

目前，**单向散列加密及加盐思想**已被广泛用于**系统登录过程中的密码生成和校验过程中**，比如 Spring Security 框架。


## Spring Security 架构

Spring Security 是 Spring 家族中历史比较悠久的一个框架，在 Spring Boot 出现之前，Spring Security 已经发展了很多年，尽管 Spring Security 的功能非常丰富，相比 Apache Shiro 这种轻量级的安全框架，它的优势就不那么明显了，加之应用程序中集成和配置 Spring Security 框架的过程比较复杂，因此它的发展过程并不是那么顺利。

而正是随着 Spring Boot 的兴起，带动了 Spring Security 的发展。它专门针对 Spring Security 提供了一套完整的自动配置方案，可以零配置使用 Spring Security。


### Spring Security 中的过滤器链

与业务中大多数处理 Web 请求的框架对比后，发现 Spring Security 中采用的是**管道-过滤器（Pipe-Filter）架构模式**，如下图所示：
![管道-过滤器架构模式示意图](http://processon.com/chart_image/6288fc7ce401fd55ba3a5bb9.png)

在上图中可以看到，处理业务逻辑的组件称为过滤器，而处理结果的相邻过滤器之间的连接件称为管道，它们构成了一组过滤器链，即 Spring Security 的核心。

项目一旦启动，**过滤器**链将会实现自动配置，如下图所示：
![Spring Security 中的过滤器链](http://processon.com/chart_image/6289005a5653bb2a362204e8.png)


在上图中，看到了 `BasicAuthenticationFilter`、`UsernamePasswordAuthenticationFilter` 等几个常见的 Filter。

这些类可以**直接或间接实现 Servlet 类中的 Filter 接口**，并完成某一项具体的认证机制。

例如，上图中的 `BasicAuthenticationFilter` 用来认证用户的身份，而 `UsernamePasswordAuthenticationFilter` 用来检查输入的用户名和密码，并根据认证结果来判断是否将结果传递给下一个过滤器。

这里请注意，整个 Spring Security 过滤器链的末端是一个 `FilterSecurityInterceptor`，本质上它也是一个 Filter，但它与其他用于完成认证操作的 Filter 不同，因为它的核心功能是用来实现权限控制，即判定该请求是否能够访问目标 HTTP 端点。因为我们可以把 FilterSecurityInterceptor 对权限控制的粒度划分到方法级别，所以它能够满足前面提到的精细化访问控制。

通过上述分析，知道了在 Spring Security 中，**认证**和**授权**这两个安全性需求主要通过一系列的过滤器进行实现。

基于过滤器链，Spring Security 的核心类结构：

### Spring Security 中的核心类

先以最基础的 `UsernamePasswordAuthenticationFilter` 为例，该类的定义及核心方法 `attemptAuthentication` 如下代码所示：
```java
public class UsernamePasswordAuthenticationFilter extends AbstractAuthenticationProcessingFilter {

    public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) throws AuthenticationException {
        if (postOnly && !request.getMethod().equals("POST")) {
            throw new AuthenticationServiceException("Authentication method not supported: " + request.getMethod());
        }

        String username = obtainUsername(request);
        String password = obtainPassword(request);

        if (username == null) {
            username = "";
        }

        if (password == null) {
            password = "";
        }

        username = username.trim();
        UsernamePasswordAuthenticationToken authRequest = new UsernamePasswordAuthenticationToken(username, password);

        // Allow subclasses to set the "details" property
        setDetails(request, authRequest);
        return this.getAuthenticationManager().authenticate(authRequest);
    }
    …
}
```

围绕上述方法，通过翻阅 Spring Security 源代码，引出了该框架中一系列核心类，并梳理了它们之间的交互结构，如下图所示：
![Spring Security 核心类图](https://images.happymaya.cn/assert/spring-security/spring-security-core-class.png)

上图中的很多类，通过名称就能明白它的含义和作用。

以位于左下角的 SecurityContextHolder 为例，它是一个典型的 Holder 类，存储了应用的安全上下文对象 SecurityContext，包含系统请求中最近使用的认证信息。这里大胆猜想它的内部肯定使用了 ThreadLocal 来确保线程访问的安全性。

而正如 `UsernamePasswordAuthenticationFilter` 中的代码所示，一个 HTTP 请求到达系统后，将通过一系列的 Filter 完成用户认证，然后具体的工作交由 AuthenticationManager 完成，AuthenticationManager 成功验证后会返回填充好的 Authentication 实例。

AuthenticationManager 是一个接口，在其实现 ProviderManager 类时会进一步依赖 AuthenticationProvider 接口完成具体的认证工作。

而在 Spring Security 中存在一大批 AuthenticationProvider 接口的实现类，分别完成各种认证操作。在执行具体的认证工作时，Spring Security 势必会使用用户详细信息，上图位于右边的 UserDetailsService 服务就是用来对用户详细信息实现管理。


PS: 简要描述下在安全访问控制过程中，过滤器机制所发挥的作用是什么？ —— 实现认证和授权这两个安全性需求。过滤器本质上是对整个请求过程进行拦截

