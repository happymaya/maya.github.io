---
title: 构建 RESTful 风格的 Web 服务
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2018-05-10 17:32:00 +0800
categories: [Spring]
tags: [SpringBoot, RESTful, Web]
---

## 创建 RESTful 服务

在当下的分布式系统及微服务架构中，RESTful 风格是一种主流的 Web 服务表现方式。首先来理解什么是 RESTful 服务。

### 理解 RESTful 架构风格

REST（Representational State Transfer，表述性状态转移）本质上只是一种架构风格而不是一种规范。

这种架构风格把位于服务器端的访问入口看作一个资源，每个资源都使用 URI（Universal Resource Identifier，统一资源标识符） 得到一个唯一的地址，且在传输协议上使用标准的 HTTP 方法，比如最常见的 GET、PUT、POST 和 DELETE。

下表是 RESTful 风格的一些具体示例：
|                    URL                    | HTTP 方法 |                  描述                  |
| :---------------------------------------: | :-------: | :------------------------------------: |
|      http://www.example.com/accounts      |    GET    |         获取 Account 对象列表          |
|      http://www.example.com/accounts      |    PUT    |         更新一组 Accoun t对象          |
|      http://www.example.com/accounts      |   POST    |         新增一组 Account 对象          |
|      http://www.example.com/accounts      |  DELETE   |         删除所有 Account 对象          |
| http://www.example.com/accounts/jianxiang |    GET    | 根据账户名 jianxiang 获取 Account 对象 |
| http://www.example.com/accounts/jianxiang |    PUT    | 根据账户名 jianxiang 更新 Account 对象 |
| http://www.example.com/accounts/jianxiang |   POST    |  添加账户名为jianxiang的新Account对象  |
| http://www.example.com/accounts/jianxiang |  DELETE   | 根据账户名 jianxiang 删除 Account 对象 |

另一方面，客户端与服务器端的数据交互涉及**序列化问题**。关于**序列化**完成业务对象在网络环境上的传输的实现方式有很多，常见的有**文本**和**二进制**两大类。

目前 JSON 是一种被广泛采用的序列化方式，因此所有的代码实例我都将 JSON 作为默认的序列化方式。

## 使用基础注解

在原有 Spring Boot 应用程序的基础上，可以通过构建一系列的 Controller 类暴露 RESTful 风格的 HTTP 端点。

。这里的 Controller 与 Spring MVC 中的 Controller 概念上一致，最简单的 Controller 类如下代码所示：
```java
@RestController
public class HelloController {
    
    @GetMapping("/")
    public String index() {
        return "Hello World!";
    }
    
}
```

从以上代码中可以看到，包含了 @RestController 和 @GetMapping 这两个注解。

- @RestController 注解继承自 Spring MVC 中的 @Controller 注解，顾名思义就是一个基于 RESTful 风格的 HTTP 端点，并且会自动使用 JSON 实现 HTTP 请求和响应的序列化/反序列化方式。通过这个特性，在构建 RESTful 服务时，可以使用 @RestController 注解代替 @Controller 注解以简化开发;
- 另外一个 @GetMapping 注解也与 Spring MVC 中的 @RequestMapping 注解类似。

@RequestMapping 注解的定义，该注解所提供的属性都比较容易理解，如下代码所示：
```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Mapping
public @interface RequestMapping {
    String name() default "";

    @AliasFor("path")
    String[] value() default {};

    @AliasFor("value")
    String[] path() default {};

    RequestMethod[] method() default {};

    String[] params() default {};

    String[] headers() default {};

    String[] consumes() default {};

    String[] produces() default {};
}
```

@GetMapping 的注解的定义与 @RequestMapping 非常类似，只是默认使用了 RequestMethod.GET 指定 HTTP 方法，如下代码所示：
```java
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@RequestMapping(method = {RequestMethod.GET})
public @interface GetMapping {
    @AliasFor(
        annotation = RequestMapping.class
    )
    String name() default "";

    @AliasFor(
        annotation = RequestMapping.class
    )
    String[] value() default {};

    @AliasFor(
        annotation = RequestMapping.class
    )
    String[] path() default {};

    @AliasFor(
        annotation = RequestMapping.class
    )
    String[] params() default {};

    @AliasFor(
        annotation = RequestMapping.class
    )
    String[] headers() default {};

    @AliasFor(
        annotation = RequestMapping.class
    )
    String[] consumes() default {};

    @AliasFor(
        annotation = RequestMapping.class
    )
    String[] produces() default {};
}
```

Spring Boot 2 中引入的一批新注解中，除了 `@GetMapping` ，还有 `@PutMapping`、`@PostMapping`、`@DeleteMapping` 等注解，这些注解极大方便了显式指定 HTTP 的请求方法。当然，可以继续使用原先的 @RequestMapping 实现同样的效果。

一个更加具体的示例，以下代码展示了 account-service 中的 AccountController。
```java
@RestController
@RequestMapping(value = "accounts")
public class AccountController {

	@GetMapping(value = "/{accountId}")
	public Account getAccountById(@PathVariable("accountId") Long accountId) {
		
		Account account = new Account();
		account.setId(1L);
		account.setAccountCode("DemoCode");
		account.setAccountName("DemoName");
		return account;
	}
}
```

在该 Controller 中，通过静态的业务代码完成了根据账号编号（accountId）获取用户账户信息的业务流程。

这里用到了两层 Mapping：
1. 第一层的 @RequestMapping 注解在服务层级定义了服务的根路径“/accounts”；
2. 第二层的 @GetMapping 注解则在操作级别定义了 HTTP 请求方法的具体路径及参数信息。

到这里，一个典型的 RESTful 服务已经开发完成了，而后可以通过 java –jar 命令直接运行 Spring Boot 应用程序了。

在启动日志中，发现了以下输出内容（为了显示效果，部分内容做了调整），可以看到自定义的这个 AccountController 已经成功启动并准备接收响应。
```bash
RequestMappingHandlerMapping : Mapped "{[/accounts/{accountId}], methods=[GET]}" onto public cn.happymaya.springcss.account.domain.Account cn.happymaya.account.controller.AccountController.getAccountById (java.lang.Long)
```

而后，可以通过 Postman 访问“http://localhost:8082/accounts/1”端点以得到响应结果。

除此之外，还有一个新的注解：`@PathVariable`，该注解作用于输入的参数，

## 控制请求输入和输出

### 使用注解简化请求输入

Spring Boot 提供了一系列简单有用的注解来简化对请求输入的控制过程，常用的包括 `@PathVariable`、`@RequestParam` 和 `@RequestBody`。

其中 `@PathVariable` 注解用于获取路径参数，即从类似 url/{id} 这种形式的路径中获取 {id} 参数的值。该注解的定义如下代码所示：
```java
@Target({ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface PathVariable {
    @AliasFor("name")
    String value() default "";

    @AliasFor("value")
    String name() default "";

    boolean required() default true;
}
```

通常，使用 `@PathVariable` 注解时，只需要指定一个参数的名称即可。

再看一个示例，如下代码所示：
```java
@GetMapping(value = "/{accountName}")
public Account getAccountByAccountName(@PathVariable("accountName") String accountName) {
	return accountService.getAccountByAccountName(accountName);
}
```

`@RequestParam` 注解的作用与 `@PathVariable` 注解类似，也是用于获取请求中的参数，但是它面向类似 **url?id=XXX** 这种路径形式。

该注解的定义如下代码所示，相较 `@PathVariable` 注解，它只是多了一个设置默认值的 defaultValue 属性。
```java
@Target({ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface RequestParam {
    @AliasFor("name")
    String value() default "";

    @AliasFor("value")
    String name() default "";

    boolean required() default true;

    String defaultValue() default ValueConstants.DEFAULT_NONE;
}
```
在 HTTP 协议中，**content-type 属性**用来指定所传输的内容类型，可以通过 @RequestMapping 注解中的 produces 属性来设置这个属性。


在设置这个属性时，通常会将其设置为`application/json`，如下代码所示：
```java
@RestController
@RequestMapping(value = "accounts", produces="application/json")
public class AccountController {
}
```

`@RequestBody` 注解用来处理 `content-type` 为 `application/json` 类型时的编码内容，通过 `@RequestBody` 注解可以将请求体中的 JSON 字符串绑定到相应的 JavaBean 上。

如下代码所示就是一个使用 @RequestBody 注解来控制输入的场景。
```java
@PutMapping(value = "/")
public void updateAccount(@RequestBody Account account) {}
```
如果使用 `@RequestBody` 注解，我们可以在 Postman 中输入一个 JSON 字符串来构建输入对象，如下所示：
![](https://images.happymaya.cn/assert/spring-boot/postman-json.png)

关于控制请求输入的规则，关键在于按照 RESTful 风格的设计原则设计 HTTP 端点，对于这点业界也存在一些约定。

- 以 Account 这个领域实体为例，如果我们把它视为一种资源，那么 HTTP 端点的根节点命名上通常采用复数形式，即“/accounts”，正如前面的示例代码所示。
- 在设计 RESTful API 时，我们需要基于 HTTP 语义设计对外暴露的端点的详细路径。针对常见的 CRUD 操作，我们展示了 RESTful API 与非 RESTful API 的一些区别。

|   业务操作   |  非 RESTful API  |      RESTful API      |
| :----------: | :--------------: | :-------------------: |
| 获取用户账户 | /account/query/1 |   /accounts/1  GET    |
| 新增用户账户 |   /account/add   |  /accounts      POST  |
| 更新用户账户 |  /account/edit   |  /accounts      PUT   |
| 删除用户账户 | /account/delete  | /accounts      DELETE |

基于以上的控制请求输入的实现方法，可以给出 account-service 中 AccountController 类的完整实现过程，如下代码所示：
```java
@RestController
@RequestMapping(value = "accounts", produces="application/json")
public class AccountController {

	@Autowired
	private AccountService accountService;

	private static final Logger logger = LoggerFactory.getLogger(AccountController.class);

	@GetMapping(value = "/{accountId}")
	public Account getAccountById(@PathVariable("accountId") Long accountId) {
		logger.info("Get account by id: [{}] ", accountId);
		Account account = new Account();
		account.setId(1L);
		account.setAccountCode("DemoCode");
		account.setAccountName("DemoName");
		return account;
	}

	@GetMapping(value = "accountname/{accountName}")
	public Account getAccountByAccountName(@PathVariable("accountName") String accountName) {
		Account account = accountService.getAccountByAccountName(accountName);
		return account;
	}

	@PostMapping(value = "/")
	public void addAccount(@RequestBody Account account) {
		accountService.addAccount(account);
	}

	@PutMapping(value = "/")
	public void updateAccount(@RequestBody Account account) {
		accountService.updateAccount(account);
	}
	
	@DeleteMapping(value = "/")
	public void deleteAccount(@RequestBody Account account) {
		
		accountService.deleteAccount(account);
	}
}
```

### 控制请求的输出

相较输入控制，输出控制就要简单很多，因为 Spring Boot 所提供的 `@RestController` 注解已经屏蔽了底层实现的复杂性，只需要返回一个普通的业务对象即可。`@RestController` 注解相当于是 Spring MVC 中 `@Controller` 和 `@ResponseBody` 这两个注解的组合，它们会自动返回 JSON 数据。

例如 order-service 中的 OrderController 实现过程，如下代码所示：
```java
@RestController
@RequestMapping(value="orders/jpa")
public class JpaOrderController {
  
	@Autowired
    JpaOrderService jpaOrderService;
    	
	@GetMapping(value = "/{orderId}")
    public JpaOrder getOrderById(@PathVariable Long orderId) {
		JpaOrder order = jpaOrderService.getOrderById(orderId);
    	return order;
    }
	
	@GetMapping(value = "orderNumber/{orderNumber}")
    public JpaOrder getOrderByOrderNumber(@PathVariable String orderNumber) {	
    	return jpaOrderService.getOrderByOrderNumberBySpecification(orderNumber);rder;
    }
	
	@PostMapping(value = "")
    public JpaOrder addOrder(@RequestBody JpaOrder order) {	
    	return jpaOrderService.addOrder(order);;
    }
}
```

> 在使用 Spring Boot 构建 Web 服务时，可以使用哪些注解实现对输入参数的有效控制？


> produces 是生成的格式，consumes 是接收的格式，指定生产者所提供的内容格式。