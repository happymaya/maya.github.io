---
title:  Spring Data ElasticSearch 在 Spring Data 中的用法
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2020-09-10 21:33:00 +0800
categories: [Java, Spring]
tags:  [Spring Security, React]
math: true
mermaid: true
---

## Spring Data ElasticSearch

Spring Data 和 Elasticsearch 结合的时候，唯一需要注意的是**版本之间的兼容性问题**。

Elasticsearch 和 Spring Boot 是同时向前发展的，而 Elasticsearch 的大版本之间还存在一定的 API 兼容性问题，所以必须要知道这些版本之间的关系，如下:
| Spring Data Release Train | Spring Data Elasticsearch | Elasticsearch | Spring Boot |
| ------------------------- | ------------------------- | ------------- | ----------- |
| 2020.0.0[1]               | 4.1.x[1]                  | 7.9.3         | 2.4.x[1]    |
| Neumann                   | 4.0.x                     | 7.6.2         | 2.3.x       |
| Moore                     | 3.2.x                     | 6.8.12        | 2.2.x       |
| Lovelace                  | 3.1.x                     | 6.2.2         | 2.1.x       |
| Kay[2]                    | 3.0.x[2]                  | 5.5.0         | 2.0.x[2]    |
| Ingalls[2]                | 2.1.x[2]                  | 2.4.0         | 1.5.x[2]    |

1. 利用 Helm Chart 安装一个 Elasticsearch 集群 7.9.3 版本;
2. 在 gradle.build 里面配置 Spring Data ElasticSearch 依赖的 Jar 包;
3. 建一个目录，结构如下图所示，方便我们测试;
4. 在 application.properties 里面新增 es 的连接地址，连接本地的 Elasticsearch;
```properties
spring.data.elasticsearch.client.reactive.endpoints=127.0.0.1:9200
```
5. 新增一个 ElasticSearchConfiguration 的配置文件，主要是为了开启扫描的包；
6. 新增一个 Topic 的 Document，它类似 JPA 里面的实体，用来保存和读取 Topic 的数据，代码如下所示；
7. 新建一个 Elasticsearch 的 Repository，用来对 Elasticsearch 索引的增删改查，代码如下所示；
8. 新建一个 Controller，对 Topic 索引进行查询和添加；
9. 发送一个添加和查询的请求测试一下；
10. Elasticsearch Repository 的测试用例写法，如下面的代码和注释所示。

## ESRepository 和 JPARepository 同时存在

怎么区分不同的 Repository 用什么呢：
1. 将对 Elasticsearch 的实体、Repository 和对 JPA 操作的实体、Repository 放到不同的文件里面；
2. 新增 JpaConfiguration，用来指定 Jpa 相关的 Repository 目录；
3. 新增 User 实体，用来操作用户基本信息;
4. 新增 UserRepository，用来进行 DB 操作；
5. 写测试用例进行测试;

Spring Data 对 JPA 等 SQL 型的传统数据库的支持是非常好的，同时对 NoSQL 型的非关系类数据库的支持也比较友好，大大降低了操作不同数据源的难度，可以有效提升我们的开发效率。