---
title: Spring Security 案例设计和初始化
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2020-06-04 21:33:00 +0800
categories: [Java, Spring]
tags:  [Spring Security]
math: true
mermaid: true
---

本文，通过构建一个简单但完整的小型 Web 应用程序，完成基于 Spring Security 的认证和授权功能保护 Web 应用程序。
​
主要是：当合法用户成功登录系统之后，浏览器会跳转到一个系统主页，并展示一些个人健康档案（HealthRecord）数据。
​
​
## 案例设计
本文案例，有两条独立的代码流程：

- 完成系统业务逻辑处理的代码流程。采用经典的三层架构，即Web 层、服务层和数据访问层，因此会存在 HealthRecordController、HealthRecordService 以及 HealthRecordRepository
- 实现核心功能，自定义的用户认证的代码流程。构建独立的 UserDetailsService、 AuthenticationProvider 以及 User 以及 UserRepository 等组件

以上两条代码流程整合在一起，得到案例的整体设计蓝图，如下图所示：
​

## 系统初始化


要想实现上图中的效果，需要先对系统进行初始化。这部分工作涉及领域对象的定义、数据库初始化脚本的整理以及相关依赖组件的引入。
​

针对领域对象，重点是如下所示的 User 类定义：
```java
@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;
    private String username;
    private String password;


    @Enumerated(EnumType.STRING)
    private PasswordEncoderType passwordEncoderType;

    @OneToMany(mappedBy = "user", fetch = FetchType.EAGER)
	private List<Authority> authorities;
	…

}

```
上面的 User 类中，指定了

- 主键 id
- 用户名 username
- 密码 password 
- 加密算法枚举值 EncryptionAlgorithm，本案例使用了   BCryptPasswordEncoder 和 SCryptPasswordEncoder 两种可用的密码解密器，可以通过该枚举值进行设置
-  Authority 列表，用来指定该 User 所具备的权限信息，Authority 类的定义如下所示：
```java
@Entity
public class Authority {


    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;

    private String name;


    @JoinColumn(name = "user")
    @ManyToOne
	private User user;

	…

}

```


不难看出 User 和 Authority 之间是**一对多**的关系，这点和 Spring Security 内置的用户权限模型是一致的，这里使用了一系列来自 JPA（Java Persistence API，Java 持久化 API）规范的注解来定义领域对象之间的关联关系。
