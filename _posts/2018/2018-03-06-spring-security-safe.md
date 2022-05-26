---
title: 基于 Spring Security 确保请求安全访问
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2018-03-06 17:32:00 +0800
categories: [Spring]
tags: [SpringBoot, Spring Security]
---

在日常开发过程中，需要对 Web 应用中的不同 HTTP 端点进行不同粒度的权限控制，并且希望这种控制方法足够灵活。而借助 Spring Security 框架，就可以对其进行简单实现。

## 对 HTTP 端点进行访问授权管理

在一个 Web 应用中，权限管理的对象是通过 Controller 层暴露的一个个 HTTP 端点，而这些 HTTP 端点就是需要授权访问的资源。

开发人员使用 Spring Security 中提供的一系列丰富技术组件，即可通过简单的设置对权限进行灵活管理。

### 使用配置方法

实现访问授权的第一种方法是使用配置方法，关于配置方法的处理过程也是位于 `WebSecurityConfigurerAdapter` 类中，但使用的是 `configure(HttpSecurity http)` 方法，如下代码所示：
```java
protected void configure(HttpSecurity http) throws Exception {
    http.authorizeRequests()
        .anyRequest()
        .authenticated()
        .and()
        .formLogin()
        .and()
        .httpBasic();
}
```

上述代码就是 Spring Security 中作用于访问授权的默认实现方法，这里用到了多个常见的配置方法。

访问任何端点时，一旦在代码类路径中引入了 Spring Security 框架，就会弹出一个登录界面从而完成用户认证。因为认证是授权的前置流程，认证结束后就可以进入授权环节。

结合这些配置方法的名称，实现这种默认的授权效果的具体步骤：
1. 通过 HttpSecurity 类的 authorizeRequests() 方法，我们可以对所有访问 HTTP 端点的 HttpServletRequest 进行限制；
2. anyRequest().authenticated() 语句指定了所有请求都需要执行认证，也就是说没有通过认证的用户无法访问任何端点；
3. formLogin() 语句指定了用户需要使用表单进行登录，即会弹出一个登录界面；
4. httpBasic() 语句使用 HTTP 协议中的 Basic Authentication 方法完成认证。


当然，Spring Security 中还提供了很多其他有用的配置方法灵活使用，如下表，一起来看下:

| 配置方法                | 作用                               |
| ----------------------- | ---------------------------------- |
| anonymous()             | 允许匿名访问                       |
| authenticated()         | 允许认证用户访问                   |
| denyAll()               | 无条件禁止一切访问                 |
| hasAnyAuthority(String) | 允许具有任意权限的用户进行访问     |
| hasAnyRole(String)      | 允许具有任意角色的用户进行访问     |
| hasAuthority(String)    | 允许具有特定权限的用户进行访问     |
| haslpAddress(String)    | 允许来自特定 IP 地址的用户进行访问 |
| hasRole(String)         | 允许具有特定角色的用户进行访问     |
| permitAll()             | 无条件允许一切访问                 |
| rememberMe()            | 允许 remember-me 的用户进行访问    |


**基于上表中的配置方法，就可以通过 HttpSecurity 实现自定义的授权策略。**

比方说，希望针对`/orders`根路径下的所有端点进行访问控制，且只允许认证通过的用户访问，那么可以创建一个继承了 `WebSecurityConfigurerAdapter` 类的 `SpringCssSecurityConfig`，并覆写其中的 `configure(HttpSecurity http)` 方法来实现，如下代码所示：
```java
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    public void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
            .antMatchers("/orders/**")
            .authenticated();
    }
}
```

**请注意：虽然上表中的这些配置方法非常有用，但是由于我们无法基于一些来自环境和业务的参数灵活控制访问规则，也就存在一定的局限性。**

为此，Spring Security 还提供了一个 access() 方法，该方法允许传入一个表达式进行更细粒度的权限控制，这里，将引入 Spring 框架提供的一种动态表达式语言—— `SpEL（Spring Expression Language 的简称）`。

只要 SpEL 表达式的返回值为 true，access() 方法就允许用户访问，如下代码所示：
```java
@Override
public void configure(HttpSecurity http) throws Exception {
    http.authorizeRequests()
        .antMatchers("/orders")
        .access("hasRole('ROLE_USER')");
}
```
上述代码中，假设访问`/orders`端点的请求必须具备`ROLE_USER`角色，通过 access 方法中的 **hasRole 方法**，即可灵活地实现这个需求。

当然，除了使用 **hasRole** 外，还可以使用 **authentication、isAnonymous、isAuthenticated、permitAll** 等表达式进行实现。因这些表达式的作用与前面介绍的配置方法一致。

### 使用注解
除了使用配置方法，Spring Security 提供了 `@PreAuthorize` 注解实现类似的效果，该注解定义如下代码所示：
```java
@Target({ ElementType.METHOD, ElementType.TYPE })
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface PreAuthorize {
    //通过 SpEL 表达式设置访问控制
    String value();
}
```

可以看到 `@PreAuthorize` 的原理与 access() 方法一样，即通过传入一个 SpEL 表达式设置访问控制，如下所示代码就是一个典型的使用示例：
```java
@RestController
@RequestMapping(value="orders")
public class OrderController {

    @PostMapping(value = "/")
    @PreAuthorize("hasRole(ROLE_ADMIN)")
    public void addOrder(@RequestBody Order order) {
        …
    }
}
```

从这个示例中可以看到，在`/orders/`这个 HTTP 端点上，添加了一个 `@PreAuthorize` 注解用来限制只有角色为`ROLE_ADMIN`的用户才能访问该端点。

其实，Spring Security 中用于授权的注解还有 `@PostAuthorize`，它与 `@PreAuthorize` 注解是一组，主要用于请求结束之后检查权限。因这种情况比较少见，

## 实现多维度访问授权方案

HTTP 端点是 Web 应用程序的一种资源，而每个 Web 应用程序对于自身资源的保护粒度因服务而异。对于一般的 HTTP 端点，用户可能通过认证就可以访问；对于一些重要的 HTTP 端点，用户在已认证的基础上还会有一些附加要求。

对资源进行保护的三种粒度级别：
1. **用户级别**： 该级别是**最基本的资源保护级别**，只要是认证用户就可能访问服务内的各种资源。
2. **用户 + 角色级别**： 该级别在认证用户级别的基础上，还**要求用户属于某一个或多个特定角色**。
3. **用户 + 角色 + 操作级别**： 该级别在**认证用户 + 角色级别**的基础上，**对某些 HTTP 操作方法做了访问限制**。

基于配置方法和注解，可以轻松实现上述三种访问授权方案。


### 使用用户级别保护服务访问
因为 CustomerController 是 spring-css 案例中的核心入口，所以我们它的所有端点都应该受到保护。于是，在 customer-service 中，创建了一个 SpringCssSecurityConfig 类继承 WebSecurityConfigurerAdapter，如下代码所示：
```java
@Configuration
public class SpringCssSecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    public void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
            .anyRequest()
            .authenticated();
    }
}
```
位于 `configure()` 方法中的 `.anyRequest().authenticated()` 语句指定了访问 customer-service 下的所有端点的任何请求都需要进行验证。因此，当使用普通的 HTTP 请求访问 CustomerController 中的任何 URL（例如 http://localhost:8083/customers/1），将会得到如下图代码所示的错误信息，该错误信息明确指出资源的访问需要进行认证。
```json
{
    "error": "access_denied",
    "error_description": "Full authentication is required to access to this resource"
}
```

覆写 WebSecurityConfigurerAdapter 的 config(AuthenticationManagerBuilder auth) 方法时提供了一个用户名`springcss_user`，现在就用这个用户名来添加用户认证信息并再次访问该端点。显然，因为此时传入的是有效的用户信息，所以可以满足认证要求。

### 使用用户 + 角色级别保护服务访问

对于某些安全性要求比较高的 HTTP 端点，通常需要限定访问的角色。

例如，customer-service 服务中涉及客户工单管理等核心业务，认为不应该给所有的认证用户开放资源访问入口，而应该限定只有角色为`ADMIN`的管理员才开放。这时，就可以使用认证用户 + 角色保护服务的访问控制机制，具体的示例代码如下所示：
```java
@Configuration
public class SpringCssSecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    public void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
            .antMatchers("/customers/**")
            .hasRole("ADMIN")
            .anyRequest()
            .authenticated();
    }
}
```
在上述代码中可以看到，使用了`HttpSecurity`类中的`antMatchers("/customer/")`和`hasRole("ADMIN")`方法为访问`/customers/`的请求限定了角色，只有`ADMIN` 角色的认证用户才能访问以`/customers/`为根地址的所有 URL。

如果使用了**认证用户 + 角色的方式保护服务访问**，使用角色为`USER`的认证用户`springcss_user`访问 customer-service 时就会出现如下所示的“access_denied”错误信息：
```json
{
    "error": "access_denied",
    "error_description": "Access is denied"
}
```


### 使用用户 + 角色+操作级别保护服务访问

在认证用户+角色的基础上，我们需要再对具体的 HTTP 操作进行限制。

在 customer-service 中，我们认为所有对客服工单的删除操作都很危险，因此可以使用 http.antMatchers(HttpMethod.DELETE, "/customers/**") 方法对删除操作进行保护，示例代码如下：
```java
@Configuration
public class SpringCssSecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    public void configure(HttpSecurity http) throws Exception{
        http.authorizeRequests()
            .antMatchers(HttpMethod.DELETE, "/customers/**")
            .hasRole("ADMIN")
            .anyRequest()
            .authenticated();
    }
}
```
上述代码的效果在于对`/customers`端点执行删除操作时，需要使用具有`ADMIN`角色的`springcss_admin`用户，执行其他操作时不需要。因为如果使用`springcss_user`账户执行删除操作，还是会出现`access_denied`错误信息。