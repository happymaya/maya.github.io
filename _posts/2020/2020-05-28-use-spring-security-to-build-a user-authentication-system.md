---
title:  Spring Security 的配置体系
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2020-05-28 22:33:00 +0800
categories: [Java, Spring]
tags:  [Spring Security]
math: true
mermaid: true
---

用户认证主要涉及用户账户体系构建（实现授权管理的前提）。
在 `Spring Security` 中，实现用户认证的方式有很多，本文主要结合框架提供的配置体系进行梳理。
​
在 `Spring Security`中，认证和授权等功能不止一种实现方法，因此 ss 提供了一套完整的配置体系来对这些功能进行灵活设置。使用认证和授权等功能时就依赖于如何合理利用和扩展这套配置体系。
​

例如，针对用户账户存储这个切入点，就可以设计出多种不同的策略

- 将用户名和密码保存在内存中，作为一种轻量级的实现方式，
- 更常见的，将这些认证信息存储在关系型数据库中
- 如果使用了 [LDAP](https://baike.baidu.com/item/%E8%BD%BB%E5%9E%8B%E7%9B%AE%E5%BD%95%E8%AE%BF%E9%97%AE%E5%8D%8F%E8%AE%AE/10493115?fromtitle=LDAP&fromid=2875565&fr=aladdin) 协议，文件系统也是一种不错的存储媒介。
> LDAP 协议，全称是 Lightweight Directory Access Protocol，**「轻量目录访问协**
> **议」**

​

针对上面可选择的实现方式，需要提供一种机制，以便根据自身需求进行灵活设置，这就是配置体系的作用。
​

在上文的示例中，没有进行任何配置也能让 `Spring Security` 发挥作用，这就说明框架内部功能采用了**特定的默认配置。**
​

就用户认证这一场景而言，`Spring Security`内部初始化一个默认用户名“user”，在应用程序启动时自动生产一个密码，通过这种方式自动生产的密码，在每次启动应用时都会发生变化，不适合面向正式的应用。
​

翻阅框架[源代码](https://github.com/spring-projects/spring-security)来进一步理解 `Spring Security` 的默认配置。
​

在 `Spring Security` 中，初始化用户信息依赖的配置类是 `WebSecurityConfigurer` 接口，该接口是一个空接口，继承了更为基础的 `SecurityConfigurer` 接口。
​

在日常开发中，不需要自己实现这个接口，使用 `WebSecurityConfigurerAdapte` 类来简化该配置类的使用。
​

在 `WebSecurityConfigurerAdapter` 中发现了如下所示的 `configure` 方法：
```java
protected void configure(HttpSecurity http) throws Exception {
        http
           .authorizeRequests()
           .anyRequest().authenticated()
           .and().formLogin().and().httpBasic();
}
```
上述代码是 `Spring Security` 中用户认证和访问授权的默认实现，这里用到了多个常见的配置方法。
​

回想[《Spring Security 是一款怎样的安全框架》](https://www.yuque.com/docs/share/0035d9af-2768-417f-a9db-b36a255bbffa?# 《01 | Spring Security 是一款怎样的安全框架？》)中描述的，一旦在代码类路径中引入 `Spring Security` 框架之后，访问任何端点时就会弹出一个登录界面用来完成用户认证。
**​**

**认证是授权的前置流程**，认证结束之后就可以进入到授权环节。
​

结合这些配置方法，简单分析一下默认效果的实现：

1. 通过 `HttpSecurity` 类的 `authorizeRequests()`方法对所有访问 `HTTP` 端点的 `HttpServletReques` 进行限制；
1. `anyRequest().authenticated()` 语句指定了对所有请求都需要执行认证（没有通过认证的用户无法访问任何端点）；
1. `formLogin()` 语句用于指定使用表单登录作为认证方式（会弹出一个登录界面）
1. `httpBasic()` 语句表示可以使用 HTTP 基础认证`（Basic Authentication）`方法来完成认证



在日常开发过程中，通过继承 `WebSecurityConfigurerAdapter` 类并且覆写上述的 `configure() `方法来完成配置工作。
​

在 `Spring Security 中`，存在一批类似于 `WebSecurityConfigurerAdapter` 的配置类。


## 2 实现 HTTP 基础认证和表单登录认证


`httpBasic() 和 formLogin()` 是控制用户认证的实现手段，分别代表了：

-  **HTTP 基础认证**
- **表单登录认证。**

在构建 Web 应用程序时，在 `Spring Security` 提供的认证机制的基础上进行扩展，以满足日常开发需求。


### 2.1 HTTP 基础认证


`HTTP` 基础认证原理较为简单，只需要**通过 **`**HTTP**`**协议消息头携带的用户名和密码**进行登录验证。


在[《Spring Security 是一款怎样的安全框架》](https://www.yuque.com/docs/share/0035d9af-2768-417f-a9db-b36a255bbffa?# 《01 | Spring Security 是一款怎样的安全框架？》)中通过浏览器简单验证用户登录操作，现在，引入` Postman` 对登录的请求和响应进一步分析。
​

在` Postman` 中，之间访问` http://localhost:8080/hello` 端点，得到如下所示的响应：
```json
{
  "timestamp": "2021-02-08T03:45:21.512+00:00",
  "status": 401,
  "error": "Unauthorized",
  "message": "",
  "path": "/hello"
}

```
显然，状态码 401 说明没有访问该端点的权限。同时，在响应中出现了 `WWW-Authenticate`消息头，其值是`Basic realm="Realm"`，这里的 `Realm`表示 Web 服务器中受保护资源的安全域。
​

接下来，执行 `HTTP`基础认证，通过设置认证类型为 `Basic Auth` 并输入对应用户名和密码完成`HTTP` 端点的访问。
​

再次查看 `HTTP` 请求，可以看到 Request Header 中添加了 [ Authorization](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Authorization?fileGuid=xxQTRXtVcqtHK6j8) 标头，格式为：`Authorization: <type> <credentials>`，这里的 type 就是“Basic”，而 credentials 则是这样一个字符串：
```json
dXNlcjo5YjE5MWMwNC1lNWMzLTQ0YzctOGE3ZS0yNWNkMjY3MmVmMzk=
```
这个字符串就是将用户名和密码组合在一起，再经过 `Base64 `编码得到的结果（Base64 是一种编码方式，并没有集成加密机制，本质上传输的还是明文形式）
​

在应用程序中启用 HTTP 基础认证是比较简单的，只需要在 `WebSecurityConfigurerAdapter` 的 `configure` 方法中添加如下配置即可：
```java
protected void configure(HttpSecurity http) throws Exception {
    http.httpBasic();
}

```
HTTP 基础认证比较简单，没有定制的登录页面，所以单独使用场景有限。
​

在使用 `Spring Security` 时，**一般会把 HTTP 基础认证和表单登录认证结合起来**一起使用。
​

### 2.2 表单登录认证 
在 `WebSecurityConfigurerAdapter` 的`configure()` 方法中，一旦配置了 `HttpSecurity` 的 `formLogin()` 方法，就启动了表单登录认证，如下所示：
```java
protected void configure(HttpSecurity http) throws Exception {
    http.formLogin();
}

```
`formLogin()` 方法的执行效果是提供了一个默认的登录界面。
​

对于登录操作而言，登录界面通常是定制化的，同时，也需要对登录的过程和结果进行细化控制。此时，可以通过如下所示的配置内容来修改系统的默认配置：
```java
@Override

protected void configure(HttpSecurity http) throws Exception {
    http
        .formLogin()        
        .loginPage("/login.html") //自定义登录页面
        .loginProcessingUrl("/action") //登录表单提交时的处理地址
        .defaultSuccessUrl("/index"); //登录认证成功后的跳转页面        
}

```


## 3 配置 Spring Security 用户认证体系
​

因为 `Spring Security` 默认提供的用户名是固定的，而密码会随着每次应用程序的启动而变化，所以很不灵活。
​

在` Spring Boot` 中，可以通过在 `application.yml` 配置文件中添加如下所示的配置项来改变这种默认行为：
```yaml
spring:
security:
user:
name: spring
      password: spring_password
```
重启应用，就可以使用上述用户名和密码完成登录。
​

基于配置文件的用户信息存储方案简单直接，但也**缺乏灵活性**，因为无法在系统运行时**动态加载对应的用户名和密码**。
​

因此，在现实中，主要还是通过使用` WebSecurityConfigurerAdapter` 配置类来改变默认的配置行为。
​

通过前面的内容中，可以通过 `WebSecurityConfigurerAdapter` 类的 `configure(HttpSecurity http)` 方法来完成认证。
​

认证过程涉及 `Spring Security` 中用户信息的交互，通过继承 `WebSecurityConfigurerAdapter` 类并且覆写` configure(AuthenticationManagerBuilder auth)` 方法来完成对用户信息的配置工作。
​

请注意,**这是两个不同的**`** configure()**`** 方法**。
​

针对 `WebSecurityConfigurer` 配置类，首先明确配置的内容。实际上，初始化用户信息非常简单，只需要指定用户名（Username）、密码（Password）和角色（Role）这三项数据即可。
​

在 `Spring Security` 中，`AuthenticationManagerBuilder` 工具类提供了**基于内存、JDBC、LDAP 等多种验证方案**。
​

### 3.1 使用基于内存的用户信息存储方案
基于内存的用户信息存储方案，实现方法就是调用 `AuthenticationManagerBuilder` 的 `inMemoryAuthentication` 方法，示例代码如下：
```java
@Override

protected void configure(AuthenticationManagerBuilder builder) throws Exception {
    builder.inMemoryAuthentication()
        .withUser("spring_user")
        .password("password1")
        .roles("USER")
        .and()
        .withUser("spring_admin")
        .password("password2")
        .roles("USER", "ADMIN");
}

```
可以看到，基于内存的用户信息存储方案实现也比较简单，但同样缺乏灵活性，因为用户信息是写死在代码里的。
### 3.2  基于数据库的用户信息存储方案
既然是将用户信息存储在数据库中，势必需要**创建表结构**。
​

可以在 `Spring Security` 的源文件中找到对应 [SQL 语句](org/springframework/security/core/userdetails/jdbc/users.ddl)，如下所示：
```sql
create table users (
  username varchar_ignorecase(50) not null primary key,
  password varchar_ignorecase(500) not null,
  enabled boolean not null);
create table authorities (
  username varchar_ignorecase(50) not null,
  authority varchar_ignorecase(50) not null,
  constraint fk_authorities_users foreign key(username) references users(username));

create unique index ix_auth_username on 
  authorities (username,authority);

```
然后通过注入一个 `DataSource` 对象进行用户数据的查询，如下所示：
```java
@Autowired
DataSource dataSource;

@Override
protected void configure(AuthenticationManagerBuilder auth) throws Exception {  
    auth.jdbcAuthentication().dataSource(dataSource) 
        .usersByUsernameQuery("select username, password, enabled from Users " + "where username=?")
        .authoritiesByUsernameQuery("select username, authority from UserAuthorities " + "where username=?")
        .passwordEncoder(new BCryptPasswordEncoder());
}

```
这里使用了 `AuthenticationManagerBuilder` 的 `jdbcAuthentication` 方法来配置数据库认证方式。
​

内部则使用了 `JdbcUserDetailsManager` 这个工具类。在该类中，定义了各种用于数据库查询的 SQL 语句，以及使用 `JdbcTemplate` 完成数据库访问的具体实现方法。
请注意，这里用到一个 `**passwordEncoder()**`** 方法**，这是 `Spring Security `中提供的一个**密码加解密器**，
