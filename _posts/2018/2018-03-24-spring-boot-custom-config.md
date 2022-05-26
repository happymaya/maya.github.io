---
title: Spring Boot 创建和管理自定义的配置信息
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2018-03-24 20:32:00 +0800
categories: [Spring]
tags: [SpringBoot]
---

## 在应用程序中嵌入系统配置信息

Spring Boot 通过自动配置机制内置了很多默认的配置信息，在这些配置信息中，有一部分系统配置信息也可以反过来作为配置项应用到我们自己的应用程序中。

例如，获取当前应用程序名称，并作为一个配置项进行管理，很简单，直接通过 ${spring.application.name} 占位符就可以做到这一点，如下所示：
```
myapplication.name: ${spring.application.name}
```

通过 ${} 占位符同样可以引用配置文件中的其他配置项内容，如在下列配置项中，最终`system.description`配置项的值就是`The system springcss is used for health`。

```propertie
system.name=springcss
system.domain=health
system.description=The system ${name} is used for ${domain}.
```
假设使用 Maven 来构建应用程序，那么可以按如下所示的配置项来动态获取与系统构建过程相关的信息：
```yaml
info: 
  app:
    encoding: @project.build.sourceEncoding@
    java:
      source: @java.version@
      target: @java.version@
```

上述配置项的效果与如下所示的静态配置是一样的：
```yaml
info:
  app:
    encoding: UTF-8
    java:
        source: 1.8.0_31
        target: 1.8.0_31
```
根据不同的需求，在应用程序中嵌入系统配置信息是很有用的，特别是在一些面向 DevOps 的应用场景中。


## 创建和使用自定义配置信息

面对纷繁复杂的应用场景，Spring Boot 所提供的内置配置信息并不一定能够完全满足开发的需求，这就需要创建并管理各种自定义的配置信息。

例如，对于一个电商类应用场景，为了鼓励用户完成下单操作，希望每完成一个订单给就给到用户一定数量的积分。从系统扩展性上讲，这个积分应该是可以调整的，所以创建了一个自定义的配置项，如下所示：
```propertie
springcss.order.point = 10
```

应用程序该获取这个配置项的内容呢,通常有两种方法。

### 使用 @Value 注解

使用 @Value 注解来注入配置项内容是一种传统的实现方法。针对前面给出的自定义配置项，可以构建一个 SpringCssConfig 类，如下所示：
```java
@Component
public class SpringCssConfig {
    @Value("${springcss.order.point}")
    private int point;
}
```
在 SpringCssConfig 类中，要做的就是在字段上添加 @Value 注解，并指向配置项的名称即可。

### 使用 @ConfigurationProperties 注解

相较 @Value 注解，更为现代的一种做法是使用 @ConfigurationProperties 注解。在使用该注解时，通常会设置一个“prefix”属性用来指定配置项的前缀，如下所示：
```java
@Component
@ConfigurationProperties(prefix = "springcss.order")
public class SpringCsshConfig {
	privat int point;
	//省略 getter/setter
}
```

相比 @Value 注解只能用于指定具体某一个配置项，@ConfigurationProperties 可以用来批量提取配置内容。只要指定 prefix，就可以把该 prefix 下的所有配置项按照名称自动注入业务代码中。

考虑一种更常见也更复杂的场景：假设用户根据下单操作获取的积分并不是固定的，而是根据每个不同类型的订单会有不同的积分，那么现在的配置项的内容，如果使用 Yaml 格式的话就应该是这样：
```yaml
springcss:
    points:
        orderType[1]: 10
        orderType[2]: 20
        orderType[3]: 30
```
如果把这些配置项全部加载到业务代码中，使用 @ConfigurationProperties 注解同样也很容易实现。直接在配置类 SpringCssConfig 中定义一个 Map 对象，然后通过 Key-Value 对来保存这些配置数据，如下所示：
```java
@Component
@ConfigurationProperties(prefix="springcss.points")
public class SpringCssConfig {
    private Map<String, Integer> orderType = new HashMap<>();
	//省略 getter/setter
}
```
可以看到这里通过创建一个 HashMap 来保存这些 Key-Value 对。类似的，也可以实现常见的一些数据结构的自动嵌入。

### 为自定义配置项添加提示功能

对于管理自定义的配置信息，如何实现当输入某一个配置项的前缀时，IDE 就会自动弹出该前缀下的所有配置信息供你进行选择 ？

当我们在 application.yml 配置文件中添加一个自定义配置项时，会注意到 IDE 会出现一个提示，说明这个配置项无法被 IDE 所识别。

遇到这种提示时，是可以忽略的，因为它不会影响到任何执行效果。但为了达到自动提示效果，就需要生成配置元数据。

生成元数据的方法也很简单，直接通过 IDE 的`Create metadata for 'springcss.order.point'`按钮，就可以选择创建配置元数据文件，这个文件的名称为 `additional-spring-configuration-metadata.json`，文件内容如下所示：
```json
{
    "properties": [
        {
            "name": "springcss.order.point",
            "type": "java.lang.String",
            "description": "A description for 'springcss.order.point'"
        }
    ]
}
```
通过这种方式，IDE 就会自动提示完整的配置项内容。

另外，假设需要为 springcss.order.point 配置项指定一个默认值，可以通过在元数据中添加一个"defaultValue"项来实现，如下所示：
```json
{
    "properties": [
        {
            "name": "springcss.order.point",
            "type": "java.lang.String",
            "description": "A description for 'springcss.order.point'",
            "defaultValue": 10
        }
    ]
}
```
这时候，在 IDE 中设置这个配置项时，就会提出该配置项的默认值为 10。

## 组织和整合配置信息

Profile 可以认为是管理配置信息中的一种有效手段。除此之外，另一种组织和整合配置信息的方法，这种方法同样依赖于 `@ConfigurationProperties` 注解。


### 使用 @PropertySources 注解

在使用 @ConfigurationProperties 注解时，可以和 @PropertySource 注解一起进行使用，从而指定从哪个具体的配置文件中获取配置信息。

例如，在下面这个示例中，通过 @PropertySource 注解指定了 @ConfigurationProperties 注解中所使用的配置信息是从当前类路径下的 application.properties 配置文件中进行读取。
```java
@Component
@ConfigurationProperties(prefix = "springcss.order")
@PropertySource(value = "classpath:application.properties")
public class SpringCssConfig {}
```

既然可以通过 @PropertySource 注解来指定一个配置文件的引用地址，那么显然也可以引入多个配置文件，这时候用到的是 @PropertySources 注解，使用方式如下所示：
```java
@PropertySources({
    @PropertySource("classpath:application.properties "),
    @PropertySource("classpath:redis.properties"),
    @PropertySource("classpath:mq.properties")
})
public class SpringCssConfig {}
```

这里，通过 @PropertySources 注解组合了多个 @PropertySource 注解中所指定的配置文件路径。SpringCssConfig 类可以同时引用所有这些配置文件中的配置项。

另一方面，也可以通过配置 spring.config.location 来改变配置文件的默认加载位置，从而实现对多个配置文件的同时加载。

例如，如下所示的执行脚本会在启动 customerservice-0.0.1-SNAPSHOT.jar 时加载 D 盘下的 application.properties 文件，以及位于当前类路径下 config 目录中的所有配置文件：

```bash
java -jar customerservice-0.0.1-SNAPSHOT.jar --spring.config.location=file:///D:/application.properties, classpath:/config/
```
## 配置文件的加载顺序

通过前面的示例，看到可以把配置文件保存在多个路径，而这些路径在加载配置文件时具有一定的顺序。Spring Boot 在启动时会扫描以下位置的 application.properties 或者 application.yml 文件作为全局配置文件：
```bash
–file:./config/
–file:./
–classpath:/config/
–classpath:/
```
它们的优先级，依次是：
1. –classpath:/config/
2. –classpath:/
3. –file:./config/
4. –file:./

Spring Boot 会全部扫描上图中的这四个位置，扫描规则是高优先级配置内容会覆盖低优先级配置内容。而如果高优先级的配置文件中存在与低优先级配置文件不冲突的属性，则会形成一种互补配置，也就是说会整合所有不冲突的属性。

## 覆写内置的配置类

如果不想使用这些配置 Spring Boot 内置了大量的自动配置，就需要对它们进行覆写。覆写的方法有很多，可以使用配置文件、Groovy 脚本以及 Java 代码。这里，以 Java 代码为例来简单演示覆写配置类的实现方法（参考 Spring Security 框架中，设置用户认证信息所依赖的配置类是 WebSecurityConfigurer 类，pring Security 提供了 WebSecurityConfigurerAdapter 这个适配器类来简化该配置类的使用方式，可以继承 WebSecurityConfigurerAdapter 类并且覆写其中的 configure() 的方法来完成自定义的用户认证配置工作的源码）。