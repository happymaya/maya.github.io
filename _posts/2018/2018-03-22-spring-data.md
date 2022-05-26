---
title: Spring Data 对数据访问过程的统一抽象
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2018-03-02 17:32:00 +0800
categories: [Spring]
tags: [SpringBoot, Spring Data]
---

事实上，JdbcTemplate 是相对偏底层的一个工具类，作为系统开发最重要的基础功能之一，数据访问层组件的开发方式在 Spring Boot 中也得到了进一步简化，并充分发挥了 Spring 家族中另一个重要成员 Spring Data 的作用。

Spring Data 是 Spring 家族中专门用于数据访问的开源框架，其核心理念是**对所有存储媒介支持资源配置从而实现数据访问**。

众所周知，数据访问需要完成领域对象与存储数据之间的映射，并对外提供访问入口，Spring Data 基于 Repository 架构模式抽象出一套实现该模式的统一数据访问方式。

Spring Data 对数据访问过程的抽象主要体现在两个方面：
1. 提供了一套 Repository 接口定义及实现；
2. 实现了各种多样化的查询支持。

## Repository 接口及实现

Repository 接口是 Spring Data 中对数据访问的最高层抽象，接口定义如下所示：
```java
public interface Repository<T, ID> {}
```
在以上代码中，看到 Repository 接口只是一个空接口，通过泛型指定了领域实体对象的类型和 ID。

在 Spring Data 中，存在一大批 Repository 接口的子接口和实现类，该接口的部分类层结构如下所示：
![Repository 接口的部分类层结构图](https://images.happymaya.cn/assert/spring-boot/spring-data-repository.png)

可以看到 CrudRepository 接口是对 Repository 接口的最常见扩展，添加了对领域实体的 CRUD 操作功能，具体定义如下代码所示：
```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by FernFlower decompiler)
//

package org.springframework.data.repository;

import java.util.Optional;

@NoRepositoryBean
public interface CrudRepository<T, ID> extends Repository<T, ID> {
    <S extends T> S save(S var1);

    <S extends T> Iterable<S> saveAll(Iterable<S> var1);

    Optional<T> findById(ID var1);

    boolean existsById(ID var1);

    Iterable<T> findAll();

    Iterable<T> findAllById(Iterable<ID> var1);

    long count();

    void deleteById(ID var1);

    void delete(T var1);

    void deleteAll(Iterable<? extends T> var1);

    void deleteAll();
}

```

这些方法都是自解释的，可以看到 CrudRepository 接口提供了**保存单个实**体、**保存集合**、**根据 id 查找实体**、**根据 id 判断实体是否存在**、**查询所有实体**、**查询实体数量**、**根据 id 删除实体** 、**删除一个实体的集合**以及**删除所有实体**等常见操作，其中几个方法的实现过程。

在实现过程中，首先需要关注最基础的 save 方法。通过查看 CrudRepository 的类层结构，找到它的一个实现类 SimpleJpaRepository，这个类显然是基于 JPA 规范所实现的针对关系型数据库的数据访问类。

save 方法如下代码所示：
```java
private final JpaEntityInformation<T, ?> entityInformation;
private final EntityManager em;

@Transactional
public <S extends T> S save(S entity) {
    if (this.entityInformation.isNew(entity)) {
        this.em.persist(entity);
        return entity;
    } else {
        return this.em.merge(entity);
    }
}
```
显然，上述 save 方法依赖于 JPA 规范中的 EntityManager，当它发现所传入的实体为一个新对象时，就会调用 EntityManager 的 persist 方法，反之使用该对象进行 merge。

接着看一下用于根据 id 查询实体的 findById 方法，如下代码所示：
```java
public Optional<T> findById(ID id) {
    Assert.notNull(id, "The given id must not be null!");
    Class<T> domainType = this.getDomainClass();
    if (this.metadata == null) {
        return Optional.ofNullable(this.em.find(domainType, id));
    } else {
        LockModeType type = this.metadata.getLockModeType();
        Map<String, Object> hints = this.getQueryHints().withFetchGraphs(this.em).asMap();
        return Optional.ofNullable(type == null ? this.em.find(domainType, id, hints) : this.em.find(domainType, id, type, hints));
    }
}
```

在执行查询过程中，findById 方法会根据领域实体的类型调用 EntityManager 的 find 方法来查找目标对象。需要注意的是，这里也会用到一些元数据 Metadata，以及涉及改变正常 SQL 执行效果的 Hint 机制的使用。

## 多样化查询支持

在日常开发过程中，数据查询的操作次数要远高于数据新增、数据删除和数据修改，因此在 Spring Data 中，除了对领域对象提供默认的 CRUD 操作外，我们还需要对查询场景高度抽象。而在现实的业务场景中，最典型的查询操作是 @Query 注解和方法名衍生查询机制。


### @Query 注解

可以通过 @Query 注解直接在代码中嵌入查询语句和条件，从而提供类似 ORM 框架所具有的强大功能。

下面就是使用 @Query 注解进行查询的典型例子：
```java
@Repository
public interface AccountRepository extends JpaRepository<Account, Long>  {

	@Query("select a from Account a where a.accountName = ?1")
	Account findAccountByAccountName(String accountName);
}
```

注意到这里的 @Query 注解使用的是类似 SQL 语句的语法，它能自动完成领域对象 Account 与数据库数据之间的相互映射。因我们使用的是 JpaRepository，所以这种类似 SQL 语句的语法实际上是一种 JPA 查询语言，也就是所谓的 JPQL（Java Persistence Query Language）。

JPQL 的基本语法如下所示：
```text
SELECT 子句 FROM 子句 
[WHERE 子句] 
[GROUP BY 子句]
[HAVING 子句] 
[ORDER BY 子句]
```

JPQL 语句是不是和原生的 SQL 语句非常类似，唯一的区别就是 JPQL FROM 语句后面跟的是对象，而原生 SQL 语句中对应的是数据表中的字段。

@Query 注解定义，这个注解位于 `org.springframework.data.jpa.repository` 包中，如下所示：
```java
package org.springframework.data.jpa.repository;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;
import org.springframework.data.annotation.QueryAnnotation;

@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD, ElementType.ANNOTATION_TYPE})
@QueryAnnotation
@Documented
public @interface Query {
    String value() default "";

    String countQuery() default "";

    String countProjection() default "";

    boolean nativeQuery() default false;

    String name() default "";

    String countName() default "";
}

```
@Query 注解中最常用的就是 value 属性，在前面示例中 JPQL 语句有使用到 。

如果将 nativeQuery 设置为 true，那么 value 属性则需要指定具体的原生 SQL 语句。

请注意，在 Spring Data 中存在一批 @Query 注解，分别针对不同的持久化媒介。例如 MongoDB 中存在一个 @Query 注解，但该注解位于 org.springframework.data.mongodb.repository 包中，定义如下：
```java
package org.springframework.data.mongodb.repository;

public @interface Query {
    String value() default "";
    String fields() default "";
    boolean count() default false;
    boolean exists() default false;
    boolean delete() default false;
}
```

与面向 JPA 的 @Query 注解不同的是，MongoDB 中 @Query 注解的 value 值是一串 JSON 字符串，用于指定需要查询的对象条件。

### 方法名衍生查询

方法名衍生查询也是 Spring Data 的查询特色之一，通过在方法命名上直接使用查询字段和参数，Spring Data 就能自动识别相应的查询条件并组装对应的查询语句。典型的示例如下所示：
```java
@Repository
public interface AccountRepository extends JpaRepository<Account, Long>  {

    List<Account> findByFirstNameAndLastName(String firstName, String lastName);
}
```
在上面的例子中，通过 findByFirstNameAndLastname 这样符合普通语义的方法名，并在参数列表中按照方法名中参数的顺序和名称（即第一个参数是 fistName，第二个参数 lastName）传入相应的参数，Spring Data 就能自动组装 SQL 语句从而实现衍生查询。很神奇！

而想要使用方法名实现衍生查询，需要对 Repository 中定义的方法名进行一定约束。

首先需要指定一些查询关键字，常见的关键字如下表所示：
![方法名衍生查询中查询关键字列表](https://images.happymaya.cn/assert/spring-boot/spring-data-repository.png)

有了这些查询关键字后，在方法命名上还需要指定查询字段和一些限制性条件。例如，在前面的示例中，只是基于“fistName”和“lastName”这两个字段做查询。

事实上，可以查询的内容非常多，下表列出了更多的方法名衍生查询示例，如下：
![方法名衍生查询中查询关键字列表](https://images.happymaya.cn/assert/spring-boot/spring-data-method-name-example.png)

在 Spring Data 中，方法名衍生查询的功能非常强大，上表中罗列的这些也只是全部功能中的一小部分而已。

如果在一个 Repository 中同时指定了 @Query 注解和方法名衍生查询，那么 Spring Data 会具体执行哪一个呢？要想回答这个问题，就需要对查询策略有一定的了解。

在 Spring Data 中，查询策略定义在 QueryLookupStrategy 中，如下代码所示：
```java
public interface QueryLookupStrategy {

    public static enum Key {
        CREATE, USE_DECLARED_QUERY, CREATE_IF_NOT_FOUND;

        public static Key create(String xml) {
            if (!StringUtils.hasText(xml)) {
                return null;
            }
            return valueOf(xml.toUpperCase(Locale.US).replace("-", "_"));
        }
    }
    RepositoryQuery resolveQuery(Method method, RepositoryMetadata metadata, ProjectionFactory factory, NamedQueries namedQueries);
}

```

从以上代码中，看到 QueryLookupStrategy 分为三种，即 **CREATE**、**USE_DECLARED_QUERY **和 **CREATE_IF_NOT_FOUND**。

这里的 CREATE 策略指的是直接根据方法名创建的查询策略，也就是使用前面介绍的方法名衍生查询。

而 USE_DECLARED_QUERY 指的是声明方式，主要使用 @Query 注解，如果没有 @Query 注解系统就会抛出异常。

而最后一种 CREATE_IF_NOT_FOUND 可以理解为是 @Query 注解和方法名衍生查询两者的兼容版。请注意，Spring Data 默认使用的是 CREATE_IF_NOT_FOUND 策略，也就是说系统会先查找 @Query 注解，如果查到没有，会再去找与方法名相匹配的查询。

## Spring Data 中的组件

Spring Data 支持对多种数据存储媒介进行数据访问，表现为提供了一系列默认的 Repository，包括针对关系型数据库的 JPA/JDBC Repository，针对 MongoDB、Neo4j、Redis 等 NoSQL 对应的 Repository，支持 Hadoop 的大数据访问的 Repository，甚至包括 Spring Batch 和 Spring Integration 在内的系统集成的 Repository。

在 Spring Data 的官方网站https://spring.io/projects/spring-data 中，列出了其提供的所有组件，可以在这里看到。

根据官网介绍，Spring Data 中的组件可以分成四大类：
- 核心模块（Main modules）；
- 社区模块（Community modules）；
- 关联模块（Related modules）和正在孵化的模块（Modules in Incubation）。

例如，前面介绍的 Respository 和多样化查询功能就在核心模块 Spring Data Commons 组件中。

这里，特别想注意的是正在孵化的模块，它目前只包含一个组件，即 Spring Data R2DBC。 R2DBC 是 Reactive Relational Database Connectivity 的简写，代表响应式关系型数据库连接，相当于是响应式数据访问领域的 JDBC 规范。

ps：不是 Mybatis 的替代品、另一种实现策略，是 Java 领域的 ORM 标准规范 ！！！

> Spring Data 时，针对查询操作可以使用哪些高效的实现方法 ?