---
title: MySQL 体系结构
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2020-07-04 09:33:22 +0800
categories: [Database, MySQL]
tags:  [MySQL]
math: true
mermaid: true
---

MySQL 数据库的体系结构，如下图所示：

![](https://images.happymaya.cn/assert/db/mysql/mysql-0101.png)

通过上图，MySQL 体系结构由 **Client Connectors 层、MySQL Server 层及存储引擎层组成。**

### Client Connectors 层

负责处理客户端的连接请求，与客户端创建连接。

目前 MySQL 几乎支持所有的连接类型，例如常见的 JDBC、Python、Go 等。

### MySQL Server 层

MySQL Server 层主要包括：

- Connection Pool：负责处理和存储数据库与客户端创建的连接；一个线程负责管理一个连接；包括：

  - 用户认证模块（用户登录身份的认证）
  - 鉴权及安全管理（用户执行操作权限校验）

- Service & utilities：

  管理服务&工具集，包括备份恢复、安全管理、集群管理服务和工具。

- SQL interface：负责接收客户端发送的各种 SQL 语句，比如 DML、DDL 和存储过程等。

- Parser 解析器：Parser 解析器会对 SQL 语句进行语法解析生成解析树。

- Optimizer 查询优化器：查询优化器会根据解析树生成执行计划，并选择合适的索引，然后按照执行计划执行 SQL 语言并与各个存储引擎交互。

- Caches 缓存：包括各个存储引擎的缓存部分，比如：

  - InnoDB 存储的 Buffer Pool；
  - MyISAM 存储引擎的 key buffer 等

  Caches 中也会缓存一些权限，也包括一些 Session 级别的缓存。

### 存储引擎层

存储引擎包括：

- MyISAM；
- InnoDB；
- 支持归档的 Archive ；
- 内存的 Memory 等。

MySQL是插件式的存储引擎，只要正确定义与 MySQL Server 交互的接口，任何引擎都可以访问MySQL，这也是 MySQL 流行的原因之一。

### 物理存储层

存储引擎底部是物理存储层，是文件的物理存储层，包括：

- 二进制日志；
- 数据文件；
- 错误日志；
- 慢查询日志；
- 全日志；
- redo/undo 日志等。



## MySQL 的交互过程

### SQL SELECT

如下图所示：

![](https://images.happymaya.cn/assert/db/mysql/mysql-0102.png)

过程如下：

1. 通过【客户端/服务器】通信协议与 MySQL 建立连接；
2. 查询缓存，是 MySQL 的一个可优化查询的地方，如果开启了 Query Cache 且在查询缓存过程中查询到完全相同的 SQL 语句，则将查询结果直接返回给客户端；如果没有开启Query Cache 或者没有查询到完全相同的 SQL 语句则会由解析器进行语法语义解析，并生成解析树；
3. 预处理器生成新的解析树；
4. 查询优化器生成执行计划；
5. 查询执行引擎执行 SQL 语句，此时查询执行引擎会根据 SQL 语句中表的存储引擎类型，以及对应的 API 接口与底层存储引擎缓存或者物理文件的交互情况，得到查询结果，由MySQL Server 过滤后将查询结果缓存并返回给客户端。若开启了 Query Cache，这时也会将SQL 语句和结果完整地保存到 Query Cache 中，以后若有相同的 SQL 语句执行则直接返回结果。

### SQL INSERT



### SQL UPDATE