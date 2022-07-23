---
title:  Spring Security 中的权限和角色
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2020-06-02 20:33:00 +0800
categories: [Java, Spring]
tags:  [Spring Security]
math: true
mermaid: true
---

认证是实现授权的前提和基础。在执行授权操作时需要明确目标用户，只有明确目标用户才能明确它所具备的角色和权限。

**用户**、**角色**和**权限**也是 Spring Security 中所采用的授权模型。

实现访问授权的基本手段是使用配置方法，配置方法处理过程位于 `WebSecurityConfigurerAdapter` 类中，但使用的是另一个 `configure(HttpSecurity http)` 方法，示例代码如下所示：
```java
protected void configure(HttpSecurity http) throws Exception {
    http
        .authorizeRequests().anyRequest().authenticated()
        .and().formLogin()
        .and().httpBasic();
}

```

## 基于权限进行访问控制
在 Spring Security 的用户对象以及它们之间的关联关系，如下图：

![Spring Security 中的核心用户对象 (2).png](https://cdn.nlark.com/yuque/0/2022/png/12442250/1646712828405-8c72ca87-5cf3-4009-8ca7-f810ba75d04f.png#clientId=u8726bcf3-666c-4&crop=0&crop=0&crop=1&crop=1&from=ui&id=ue1e8bdd9&margin=%5Bobject%20Object%5D&name=Spring%20Security%20%E4%B8%AD%E7%9A%84%E6%A0%B8%E5%BF%83%E7%94%A8%E6%88%B7%E5%AF%B9%E8%B1%A1%20%282%29.png&originHeight=320&originWidth=600&originalType=binary&ratio=1&rotation=0&showTitle=false&size=36386&status=done&style=none&taskId=u1f8d6d46-0ad0-42e1-a95f-3b841a95eed&title=)

上图中的：

- GrantedAuthority 对象，代表的就是一种权限对象
- UserDetails 对象具备一个或多个 GrantedAuthority 对象

通过这种关联关系，就可以对用户的权限做一些限制，如下所示：

![使用权限实现访问控制示意图 (2).png](https://cdn.nlark.com/yuque/0/2022/png/12442250/1646719487154-38f9ae7c-c363-4efe-99df-7a22ab1e1c39.png#clientId=u8726bcf3-666c-4&crop=0&crop=0&crop=1&crop=1&from=ui&id=ubd1789bd&margin=%5Bobject%20Object%5D&name=%E4%BD%BF%E7%94%A8%E6%9D%83%E9%99%90%E5%AE%9E%E7%8E%B0%E8%AE%BF%E9%97%AE%E6%8E%A7%E5%88%B6%E7%A4%BA%E6%84%8F%E5%9B%BE%20%282%29.png&originHeight=377&originWidth=424&originalType=binary&ratio=1&rotation=0&showTitle=false&size=39986&status=done&style=none&taskId=u1c737d4c-7574-4459-9471-ab4c3724fb3&title=)

如果用代码来表示这种关联关系，可以采用如下所示的实现方法：
```java
UserDetails user = User.withUsername("jianxiang")
     .password("123456")
     .authorities("create", "delete")
     .build();
```
可以看到，这里创建了一个名为“jianxiang”的用户，该用户具有“create”和“delete”这两个权限。
Spring Security 提供了一组针对 GrantedAuthority 的配置方法。例如：

- hasAuthority(String)，允许具有**特定权限**的用户进行访问；
- hasAnyAuthority(String)，允许具有**任一权限**的用户进行访问。

可以使用上述两个方法来判断用户是否具备对应的访问权限。

在 WebSecurityConfigurerAdapter 的 configure 方法中添加如下代码：
```java
@Override

protected void configure(HttpSecurity http) throws Exception {
    http.httpBasic();
    http.authorizeRequests().anyRequest().hasAuthority("CREATE");        
}

```
这段代码的作用是对于任何请求，只有权限为“CREATE”才能采用访问。如果将代码成下面的样子：
```java
http.authorizeRequests().anyRequest().hasAnyAuthority("CREATE", "DELETE");
```
此时，只要具备“CREATE”和“DELETE”中任意一种权限的用户都能进行访问。

这两个方法实现起来都比较简单，但局限性也很大：

- 无法基于一些来自环境和业务的参数灵活控制访问规则

针对着局限性，Spring Security 提供了一个 access() 方法，该方法允许传入一个表达式进行更加细粒度的权限控制。

这里，使用 SpEL，它是 Spring Expression Language 的简称，是 Spring 框架提供的一种动态表达式语言。基于 SpEL，只要该表达式的返回值是 true，access() 方法就会允许用户访问。如下示例：
```java
http.authorizeRequests().anyRequest().access("hasAuthority('CREATE')");
```
上面代码与使用 hasAuthority() 方法的效果是完全一致的，但如果是更为复杂的场景，access() 方法的优势就很明显了。可以灵活创建一个表达式，然后通过 access() 方法确定最后的结果，示例代码如下所示：
```java
String expression = "hasAuthority('CREATE') and !hasAuthority('Retrieve')"; 
http.authorizeRequests().anyRequest().access(expression);
```

## 基于角色进行访问控制

**角色可以看成是拥有多个权限的一种数据载体**。

如下图所示，这里分别定义了两个不同的角色“User”和“Admin”，它们拥有不同的权限：
![使用角色实现访问控制示意图 (2).png](https://cdn.nlark.com/yuque/0/2022/png/12442250/1646719869673-ffd4d488-2c5d-4d16-a875-79af49e38935.png#clientId=u8726bcf3-666c-4&crop=0&crop=0&crop=1&crop=1&from=ui&id=uf4808fe6&margin=%5Bobject%20Object%5D&name=%E4%BD%BF%E7%94%A8%E8%A7%92%E8%89%B2%E5%AE%9E%E7%8E%B0%E8%AE%BF%E9%97%AE%E6%8E%A7%E5%88%B6%E7%A4%BA%E6%84%8F%E5%9B%BE%20%282%29.png&originHeight=345&originWidth=599&originalType=binary&ratio=1&rotation=0&showTitle=false&size=41942&status=done&style=none&taskId=ucee0dfc1-ee74-440d-af40-f2cac54ff08&title=)
此时，有可能会认为 Spring Security 应该提供一个独立的数据结构来承载角色的含义。

但事实上，Spring Security ，没有定义类似“GrantedRole”这种专门用来定义用户角色的对象，而是**复用了 GrantedAuthority 对象**。

其中，以“ROLE_”为前缀的 GrantedAuthority 就代表了一种角色，因此可以使用如下方式初始化用户的角色：
```java
UserDetails user = User.withUsername("jianxiang")
      .password("123456")
      .authorities("ROLE_ADMIN")
      .build();
```
上述代码相当于为用户“jianxiang”指定了“ADMIN”这个角色。为了给开发人员提供更好的开发体验，Spring Security 还提供了另一种简化的方法来指定用户的角色，如下所示：
```java
UserDetails user = User.withUsername("jianxiang")
      .password("123456")
      .roles("ADMIN")
      .build();

```

和权限配置一样，Spring Security 也通过使用对应的 hasRole() 和 hasAnyRole() 方法来判断用户是否具有某个角色或某些角色，使用方法如下所示：

```java
http.authorizeRequests().anyRequest().hasRole("ADMIN");
```
当然，针对角色，我们也可以使用 access() 方法完成更为复杂的访问控制。而 Spring Security 还提供了其他很多有用的控制方法供开发人员进行灵活使用。作为总结，下表展示了常见的配置方法及其作用：

| 配置方法 | 作用 |
| --- | --- |
| anonymous() | 允许匿名访问 |
| authenticated() | 允许认证用户访问 |
| denyAll() | 无条件禁止一切访问 |
| hasAnyAuthority(String) | 允许具有任一权限的用户进行访问 |
| hasAnyRole(String) | 允许具有任一角色的用户进行访问 |
| hasAuthority(String) | 允许具有特定权限的用户进行访问 |
| hasIpAddress(String) | 允许来自特定 IP 地址的用户进行访问 |
| hasRole(String) | 允许具有特定角色的用户进行访问 |
| permitAll() | 无条件允许一切访问 |

# 使用配置方法控制访问权限

Spring Security 提供了三种强大的匹配器（Matcher）来实现HTTP 请求与权限控制过程关联起来，分别是：

- **MVC 匹配器**
- **Ant 匹配器**
- **正则表达式匹配器**。

假如如下所示的一个 Controller：
```java
@RestController
public class TestController {
 
    @GetMapping("/hello_user")
    public String helloUser() {
        return "Hello User!";
	}

	@GetMapping("/hello_admin")
    public String helloAdmin() {
        return "Hello Admin!";
    }

 

    @GetMapping("/other")
    public String other() {
        return "Other!";
    }

}

```
同时，创建两个具有不同角色的用户，如下所示：
```java
UserDetails user1 = User.withUsername("jianxiang1") 
    .password("12345") 
    .roles("USER") 
    .build(); 

    
UserDetails user2 = User.withUsername("jianxiang2") 
    .password("12345") 
    .roles("ADMIN") 
    .build();

```
然后，基于这个 Controller 中暴露的各个 HTTP 端点，对三种不同的匹配器总结。

## MVC 匹配器
MVC 匹配器的使用方法比较简单，就是基于 HTTP 端点的访问路径进行匹配，如下所示：
```java
http.authorizeRequests() 
    .mvcMatchers("/hello_user").hasRole("USER") 
    .mvcMatchers("/hello_admin").hasRole("ADMIN");
```
现在，如果使用角色为“USER”的用户“jianxiang1”来访问“/hello_admin”端点，那么将会得到如下所示的响应：
```java
{ 
   "status":403, 
   "error":"Forbidden", 
   "message":"Forbidden", 
   "path":"/hello_admin" 
}

```
显然，MVC 匹配器已经生效了，因为“/hello_admin”端点只有角色为“ADMIN”的用户才能访问。

如果使用拥有“ADMIN”角色的“jianxiang2”来访问这个端点就可以得到正确的响应结果。

**没有被 MVC 匹配器所匹配的端点，其访问不受任何的限制**，效果相当于如下所示的配置：
```java
http.authorizeRequests() 

    .mvcMatchers("/hello_user").hasRole("USER") 

    .mvcMatchers("/hello_admin").hasRole("ADMIN");

    .anyRequest().permitAll();

```
讲到这里，又出现了一个新问题：如果一个 Controller 中存在两个路径完全一样的 HTTP 端点呢？

这种情况是存在的，因为对于 HTTP 端点而言，就算路径一样，只要所使用的 HTTP 方法不同，那就是不同的两个端点。针对这种场景，MVC 匹配器还提供了重载的 mvcMatchers 方法，如下所示：
```java
mvcMatchers(HttpMethod method, String... patterns)

```
这样，就可以把 HTTP 方法作为一个访问的维度进行控制，示例代码如下所示：
```java
http.authorizeRequests() 
    .mvcMatchers(HttpMethod.POST, "/hello").authenticated() 
    .mvcMatchers(HttpMethod.GET, "/hello").permitAll() 
    .anyRequest().denyAll();
```
在上面这段配置代码中，如果一个 HTTP 请求使用了 POST 方法来访问“/hello”端点，那么就需要进行认证。而对于使用 GET 方法来访问“/hello”端点的请求则全面允许访问。最后，其余访问任意路径的所有请求都会被拒绝。

同时，如果想要对某个路径下的所有子路径都指定同样的访问控制，那么只需要在该路径后面添加“*”号即可，示例代码如下所示：
```java
http.authorizeRequests() 
    .mvcMatchers(HttpMethod.GET, "/user/*").authenticated() 

```
通过上述配置方法，如果访问“/user/jianxiang”“/user/jianxiang/status”等路径时，都会匹配到这条规则。

## Ant 匹配器
Ant 匹配器的表现形式和使用方法与前面介绍的 MVC 匹配器非常相似，它也提供了如下所示的三个方法来完成请求与 HTTP 端点地址之间的匹配关系：

- antMatchers(String patterns)
- antMatchers(HttpMethod method)
- antMatchers(HttpMethod method, String patterns)

从方法定义上不难明白，可以组合指定请求的 HTTP 方法以及匹配的模式，例如：
```java
http.authorizeRequests() 
    .antMatchers( "/hello").authenticated();
```
虽然，从使用方式上看，Ant 匹配器和 MVC 匹配器并没有什么区别，但在日常开发过程中，我想推荐你**使用 MVC 匹配器**而不是 Ant 匹配器，原因就在于 Ant 匹配器在匹配路径上有一些风险，主要**体现在对于"/"的处理上**。为了更好地说明，我举一个简单的例子。
基于上面的这行配置，如果你发送一个这样的 HTTP 请求：
```java
http://localhost:8080/hello
```
肯定认为 Ant 匹配器是能够匹配到这个端点的，但结果却是：
```java
{
    "status":401, 
    "error":"Unauthorized", 
    "message":"Unauthorized", 
    "path":"/hello" 

}

```
现在，如果把 HTTP 请求调整为这样，请注意，此时**在请求地址最后添加了一个”/”符号**，那么就会得到正确的访问结果：
```java
http://localhost:8080/hello/
```
显然，Ant 匹配器处理请求地址的方式有点让人感到困惑，而 MVC 匹配器则没有这个问题，无论在请求地址最后是否存在“/”符号，它都能完成正确的匹配。

## 正则表达式匹配器
最后是正则表达式匹配器，同样，它也提供了如下所示的两个配置方法：、

- regexMatchers(HttpMethod method, String regex)
- regexMatchers(String regex)

使用这一匹配器的主要优势在于它能够**基于复杂的正则表达式**对请求地址进行匹配，这是 MVC 匹配器和 Ant 匹配器无法实现的，可以看一下如下所示的这段配置代码：
```java
http.authorizeRequests()
   .mvcMatchers("/email/{email:.*(.+@.+\\.com)}")
   .permitAll()
   .anyRequest()
   .denyAll();
```
可以看到，这段代码就对常见的邮箱地址进行了匹配，只有输入的请求是一个合法的邮箱地址才能允许访问。
