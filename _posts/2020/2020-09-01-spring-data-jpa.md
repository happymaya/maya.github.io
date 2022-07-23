---
title:  Spring Data JPA
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2020-09-01 23:33:00 +0800
categories: [Java, Spring]
tags:  [Spring Security, React]
math: true
mermaid: true
---

## Spring Boot 和 Spring Data JPA

**第一步：利用 IDEA 和 SpringBoot 2.3.3 快速创建一个演示项目**
![Spring Initializr](https://images.happymaya.cn/assert/spring-data-jpa/springboot-jpa-1.png)

选择 Spring Boot 的依赖：
1. Lombok：帮我们创建一个简单的 Entity 的 POJO，主要用来省去创建 GET 和 SET 方法；
2. Spring Web：MVC 必备组件；
3. Spring Data JPA：重头戏，这是本课时的重点内容；
4. H2 Database：内存数据库；
5. Spring Boot Actuator：监控我们项目状态使用。

然后通过下图操作界面选择上面的依赖，如下图所示：
![Spring Initializr](https://images.happymaya.cn/assert/spring-data-jpa/springboot-jpa-2.png)


**第二步：通过 IDEA 的图形化界面，一路单击 Next 按钮，然后单击 Finsh 按钮，得到一个工程：**


**第三步：新增 3 个类来完成对 User 的 CURD**

第一个类：新增 User.java，它是一个实体类，用来做 User 表的映射的，如下所示：
```java
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;
    private String name;
    private String email;

}
```

第二个类：新增 UserRepository，它是 DAO 层，用来操作实体 User 进行增删改成操作，如下所示：
```java
public interface UserRepository extends JpaRepository<User, Long> {
}
```

第三个类：新增 UserController，它是 Controller，用来创建 Rest 的 API 的接口的，如下所示：
```java
package cn.happymaya.jpaguide.controller;

import cn.happymaya.jpaguide.dao.UserRepository;
import cn.happymaya.jpaguide.domain.User;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.*;

import java.awt.*;

@RestController
@RequestMapping(path = "/api/v1")
public class UserController {

    private UserRepository userRepository;

    public UserController(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    /**
     * 保存用户
     * @param user 用户对象
     * @return 返回用户对象
     */
    @PostMapping(path = "user", consumes = {MediaType.APPLICATION_ATOM_XML_VALUE})
    public User addNewUser(@RequestBody User user) {
        return userRepository.save(user);
    }

    /**
     * 根据分页信息查询用户
     * @param request 请求数据
     * @return 返回用户信息
     */
    @GetMapping(path = "users")
    @ResponseBody
    public Page<User> getAllUsers(Pageable request) {
        return userRepository.findAll(request);
    }

}
```
最终，项目结构变成如下图所示的模样：
![Spring Initializr](https://images.happymaya.cn/assert/spring-data-jpa/springboot-jpa-3.png)
上图中，appliaction.properties 里面的内容是空的，到现在三步搞定，其他什么都不需要配置，直接点击 JpaApplication 这个类，就可启动项目了。

**第四步：调用项目里面的 User 相关的 API 接口测试一下**

新增一个 JpaApplication.http 文件，内容如下：
```http
POST /api/v1/user HTTP/1.1
Host: 127.0.0.1:8080
Content-Type: application/json
Cache-Control: no-cache
{"name":"jack","email":"123@126.com"}
#######
GET http://127.0.0.1:8080/api/v1/users?size=3&page=0
###
```


运行之后，结果如下：
```json
POST http://127.0.0.1:8080/api/v1/user
HTTP/1.1 200 
Content-Type: application/json
Transfer-Encoding: chunked
Date: Sat, 22 Aug 2020 02:48:43 GMT
Keep-Alive: timeout=60
Connection: keep-alive
{
  "id": 4,
  "name": "jack",
  "email": "123@126.com"
}
Response code: 200; Time: 30ms; Content length: 44 bytes
GET http://127.0.0.1:8080/api/v1/users?size=3&page=0
HTTP/1.1 200 
Content-Type: application/json
Transfer-Encoding: chunked
Date: Sat, 22 Aug 2020 02:50:20 GMT
Keep-Alive: timeout=60
Connection: keep-alive
{
  "content": [
    {
      "id": 1,
      "name": "jack",
      "email": "123@126.com"
    },
    {
      "id": 2,
      "name": "jack",
      "email": "123@126.com"
    },
    {
      "id": 3,
      "name": "jack",
      "email": "123@126.com"
    }
  ],
  "pageable": {
    "sort": {
      "sorted": false,
      "unsorted": true,
      "empty": true
    },
    "offset": 0,
    "pageNumber": 0,
    "pageSize": 3,
    "unpaged": false,
    "paged": true
  },
  "totalPages": 2,
  "last": false,
  "totalElements": 4,
  "size": 3,
  "number": 0,
  "numberOfElements": 3,
  "sort": {
    "sorted": false,
    "unsorted": true,
    "empty": true
  },
  "first": true,
  "empty": false

}

Response code: 200; Time: 59ms; Content length: 449 bytes

```

通过以上案例，Spring Data JPA 可以做数据的 CRUD 操作。
## JPA 整合 MySQL 数据库

### 切换 MySQL 数据源
上面的例子，采用的是默认 H2 数据源的方式，现在调整一下上面的代码，以 MySQL 作为数据源。

第一处改动，application.properties 内容如下：

第二处改动，删除 H2 数据源，新增 MySQL 数据库驱动：

调整完毕之后，重启这个项目，以同样的方式测试上面的两个接口依然 OK。

其实这个时候可以发现一件事情，那就是没有手动去创建任何表，JPA 自动帮我创建了数据库的 DDL，并新增了 User 表，所以用 JPA 之后创建表的工作就不会那么复杂了，只需要把实体写好就可以了。

### Spring Data JPA 测试用例的写法

关注 Repository 的测试用例的写法，Controller 和 Service 等更复杂的测试不考虑。

第一步，在 Test 目录里增加 UserRepositoryTest 类：
```java
@DataJpaTest
public class UserRepositoryTest {
    @Autowired
    private UserRepository userRepository;

    @Test
    public void testSaveUser() {
        User user = userRepository.save(User.builder().name("jackxx").email("123456@126.com").build());
        Assert.assertNotNull(user);
        List<User> users= userRepository.findAll();
        System.out.println(users);
        Assert.assertNotNull(users);
    }
}
```

## 整体认识 JPA

### 主流 ORM 框架比对
下表是市场上比较流行的 ORM 框架，这里罗列了 MyBatis、Hibernate、Spring Data JPA 等，并对比了下它们的优缺点：
|     ORM框架     |                             优点                             |                             缺点                             |
| :-------------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
|     MyBatis     | MyBatis 本是 Apache 的一个开源项目 iBatis；<br/>2010年由 Apache Software Foundation 迁移到了Google Code，改名为MyBatis<br/>其着力于 POJO 与 SQL 之间的映射关系，可以进行更为细致的SQL编写操作使用起来十分灵活、上手简单、容易掌握，所以深受开发者的喜欢<br/>目前市场占有率最高，比较适合互联应用公司的API场景 |         工作量比较大<br/>需要各种配置文件和 SQL 语句         |
|    Hibernate    | Hibernate 是一个开放源代码的对象关系映射框架，它对 JDBC 进行了非常<br/>轻量级的对象封装，使得Jva程序员可以随心所欲地使用对象编程思维来操纵数据库<br/>对象有自己的生命周期，着力点为对象与对象之间关系<br/>有自己的 HQL 查询语言，所以数据库移植性很好<br/>Hibernate 是完备的ORM框架，是符合 JPA 规范的，有自己的缓存机制 |            上手比较难比较适合企业级的应用系统开发            |
| Spring Data JPA | 可以理解为JPA规范的再次封装抽象，底层还是使用了 Hibernate 的JPA<br/>技术实现，引用JPQL（Java Persistence Query Language)查询语言，属于Spring的整个生态体系的一部分<br/>由于 Spring Boot 和 Spring Cloud 在市场上比铰流行，Spring Data JPA也逐渐进入我们的视野，他们有机的整体，使用起来比较方便，加快了开发的效率，使开发者不需要关心和配置更多的东西，完全可以沉浸在Spring的<br/>完整生态标准的实现下上手简单、开发效率高，又对对象的支持比较好,有很大的灵活性，市场的认可度越来越高 | 入门简单<br/>上手比较快<br/>但想要精通<br/>就需要了解很多知识 |
|     OpenJPA     | 这是Apache组织提供的开源项目，它实现了EJB3.0中的JPA标准<br/>为开发者提供功能强大、使用简单的持久化数据管理框架 | 功能、性能、普及性<br/>等方面需要加大力度<br/>所以使用的人不是特<br/>别多 |
|    QueryDSL     | QueryDSL 是个对ORM框架扩展的操作，提供了一种通用的Util和方法用来构建不同查询参数的<br/>API,提供统一的抽象；目前QueryDSL支持的框架包括JPA、JDO、SQL、Java Collections、<br/>RDF、Lucene、Hibernate等 |                                                              |

## Java Persistence API 和开源实现
JPA 是 JDK 5.0 新增的协议，通过相关持久层注解（@Entity 里面的各种注解）来描述对象和关系型数据里面的表映射关系，并将 Java 项目运行期的实体对象，通过一种Session持久化到数据库中。

想象一下，一旦协议有了，大家都遵守了此种协议进行开发，那么周边开源产品就会大量出现，比如 Spring Data Rest 就基于这套标准，进而对 Entity 的操作再进行封装，从而可以得到更加全面的 Rest 协议的 API接口。

再比如 `JSON API（https://jsonapi.org/）协议`，就是雅虎的大牛基于 JPA 的协议的基础，封装制定的一套 RESTful 风格、JSON 格式的 API 协议，那么一旦 JSON API 协议成了标准，就会有很多周边开源产品出现。比如很多 JSON API 的客户端、现在比较流行的 Ember 前端框架，就是基于 Entity 这套 JPA 服务端的协议，在前端解析 API 协议，从而可以对普通 JSON 和 JSON API 的操作进行再封装。

所以规范是一件很有意思的事情，突然之间世界大变样，很多东西都一样了，我们的思路就需要转换了。

### JPA 的分类

一套 API 标准定义了一套接口，在 `javax.persistence` 的包下面，用来操作实体对象，执行 CRUD 操作，而实现的框架（Hibernate）替代完成所有的事情，从烦琐的 JDBC 和 SQL 代码中解脱出来，更加聚焦自己的业务代码，并且使架构师架构出来的代码更加可控。

定义了一套基于对象的 `SQL：Java Persistence Query Language（JPQL）`，像 Hibernate 一样，通过写面向对象（JPQL）而非面向数据库的查询语言（SQL）查询数据，避免了程序与数据库 SQL 语句耦合严重，比较适合跨数据源的场景（一会儿 MySQL，一会儿 Oracle 等）。

`ORM（Object/Relational Metadata）`对象注解映射关系，JPA 直接通过注解的方式来表示 Java 的实体对象及元数据对象和数据表之间的映射关系，框架将实体对象与 Session 进行关联，通过操作 Session 中不通实体的状态，从而实现数据库的操作，并实现持久化到数据库表中的操作，与 DB 实现同步。

[JPA 协议地址](https://github.com/eclipse-ee4j/jpa-api)

### JPA 的开源实现

JPA 的宗旨是为 POJO 提供持久化标准规范，可以集成在 Spring 的全家桶使用，也可以直接写独立 application 使用。

任何用到 DB 操作的场景，都可以使用，极大地方便开发和测试，所以 JPA 的理念已经深入人心了。

`Spring Data JPA`、`Hibernate 3.2+`、`TopLink 10.1.3` 以及 `OpenJPA`、`QueryDSL` 都是实现 JPA 协议的框架，他们之间的关系结构如下图所示：
![Spring Data JPA、Hibernate 3.2+、TopLink 10.1.3 以及 OpenJPA、QueryDSL 之间的关系结构](http://assets.processon.com/chart_image/6289ca960e3e74749fb59363.png)

俗话说得好：“未来已经来临，只是尚未流行”，大神资深开发用 Spring Data JPA，编程极客者用 JPA；而普通 Java 开发者，不想去挑战的 Java“搬砖者”用 Mybatis。

## Spring Data 认识

### 1. Spring Data 介绍

Spring Data 项目是从 2010 年开发发展起来的，Spring Data 利用一个大家熟悉的、一致的、基于**注解**的数据访问编程模型，做一些公共操作的封装，它可以轻松地让开发者使用数据库访问技术，包括关系数据库、非关系数据库（NoSQL）。同时又有不同的数据框架的实现，保留了每个底层数据存储结构的特殊特性。

Spring Data Common 是 Spring Data 所有模块的公共部分，该项目提供了基于 Spring 的共享基础设施，它提供了基于 repository 接口以 DB 操作的一些封装，以及一个坚持在 Java 实体类上标注元数据的模型。

Spring Data 不仅对传统的数据库访问技术如 JDBC、Hibernate、JDO、TopLick、JPA、MyBatis 做了很好的支持和扩展、抽象、提供方便的操作方法，还对 MongoDb、KeyValue、Redis、LDAP、Cassandra 等非关系数据的 NoSQL 做了不同的实现版本，方便我们开发者触类旁通。

其实这种接口型的编程模式可以很好地学习 Java 的封装思想，实现对操作的进一步抽象，也可以把这种思想运用在自己公司写的 Framework 上面。

### 2. Spring Data 的子项目有哪些
![Spring Data 的子项目](http://assets.processon.com/chart_image/616d8e2b0e3e747d1c874202.png)

主要项目（Main Modules）：
- Spring Data Commons，相当于定义了一套抽象的接口，下一课时我们会具体介绍
- Spring Data Gemfire
- Spring Data JPA，关注的重点，对 Spring Data Common 的接口的 JPA 协议的实现
- Spring Data KeyValue
- Spring Data LDA
- Spring Data MongoD
- Spring Data RES
- Spring Data Redi
- Spring Data for Apache Cassandr
- Spring Data for Apache Solr

社区支持的项目（Community Modules）：
- Spring Data Aerospik
- Spring Data Couchbas
- Spring Data DynamoD
- Spring Data Elasticsearc
- Spring Data Hazelcas
- Spring Data Jes
- Spring Data Neo4
- Spring Data Vault

其他（Related Modules）：
- Spring Data JDBC Extension
- Spring for Apache Hadoo
- Spring Content

除了上面这些，还有很多开源社区版本，比如 Spring Data、MyBatis 等