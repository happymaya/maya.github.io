---
title: 使用 JdbcTemplate 访问关系型数据库
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2018-04-20 17:32:00 +0800
categories: [Spring]
tags: [SpringBoot, JdbcTemplate]
---

JDBC 规范是 Java 领域中使用最广泛的数据访问标准，目前市面上主流的数据访问框架都是构建在 JDBC 规范之上。

因为 JDBC 是偏底层的操作规范，所以关于如何使用 JDBC 规范进行关系型数据访问的实现方式有很多（区别在于对 JDBC 规范的封装程度不同），而在 Spring 中，同样提供了 JdbcTemplate 模板工具类实现数据访问，它简化了 JDBC 规范的使用方法。

## 数据模型和 Repository 层设计

以 spring-css 为例子演示。

一个订单中往往涉及一个或多个商品，所以在 spring-css，主要通过一对多的关系来展示数据库设计和实现方面的技巧。为了使描述更简单，把具体的业务字段做了简化。

Order 类的定义如下代码所示：
```java
public class Order{
	
	private Long id;	
	private String orderNumber;
	private String deliveryAddress;
	private List<Goods> goodsList;		
	
    // 省略了 getter/setter
}

```

其中代表商品的 Goods 类定义如下：
```java
public class Goods {
	private Long id;	
	private String goodsCode;
	private String goodsName;
	private Double price;
	
    // 省略了 getter/setter
}

```

从以上代码，不难看出一个订单可以包含多个商品，因此设计关系型数据库表时，我们首先会构建一个中间表来保存 Order 和 Goods 这层一对多关系。

在 spring-css，使用 MySQL 作为关系型数据库，对应的数据库 Schema 定义如下代码所示：
```sql
DROP TABLE IF EXISTS `order`;
DROP TABLE IF EXISTS `goods`;
DROP TABLE IF EXISTS `order_goods`;

create table `order` (
	`id` bigint(20) NOT NULL AUTO_INCREMENT,
	`order_number` varchar(50) not null,
	`delivery_address` varchar(100) not null,
  `create_time` timestamp not null DEFAULT CURRENT_TIMESTAMP,
	PRIMARY KEY (`id`)
);

create table `goods` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `goods_code` varchar(50) not null,
  `goods_name` varchar(50) not null,
  `price` double not null,
  `create_time` timestamp not null DEFAULT CURRENT_TIMESTAMP,
	PRIMARY KEY (`id`)
);

create table `order_goods` (
	`order_id` bigint(20) not null,
	`goods_id` bigint(20) not null,
	foreign key(`order_id`) references `order`(`id`),	
	foreign key(`goods_id`) references `goods`(`id`)
);
```

基于以上数据模型，将完成 order-server 中的 Repository 层组件的设计和实现。


首先，设计一个 OrderRepository 接口，用来抽象数据库访问的入口，如下代码所示：
```java
public interface IOrderRepository {
	
	Order addOrder(Order order);
	
	Order getOrderById(Long orderId);
	
	Order getOrderDetailByOrderNumber(String orderNumber);
}

```

这个接口非常简单，方法都是自解释的。不过请注意，这里的 OrderRepository 并没有继承任何父接口，完全是一个自定义的、独立的 Repository。

针对上述 OrderRepository 中的接口定义，我构建了一系列的实现类:
- OrderRawJdbcRepositoryImpl：使用原生 JDBC 进行数据库访问
- OrderJdbcRepositoryImpl：使用 JdbcTemplate 进行数据库访问
- IOrderJpaRepositoryImpl：使用 Spring Data JPA 进行数据库访问

OrderRawJdbcRepositoryImpl 类中实现方式如下代码所示：
```java
/**
 * 使用原生 JDBC 进行数据库访问
 */
@Repository("orderRawJdbcRepository")
public class OrderRawJdbcRepositoryImpl implements IOrderRepository {

	@Autowired
	private DataSource dataSource;

	@Override
	public Order addOrder(Order order) {
		// 不做实现
		return null;
	}

	@Override
	public Order getOrderById(Long orderId) {

		Connection connection = null;
		PreparedStatement statement = null;
		ResultSet resultSet = null;
		try {
			connection = dataSource.getConnection();
			statement = connection.prepareStatement("select id, order_number, delivery_address from `order` where id=?");
			statement.setLong(1, orderId);
			resultSet = statement.executeQuery();
			Order order = null;
			if (resultSet.next()) {
				order = new Order(resultSet.getLong("id"), resultSet.getString("order_number"),
						resultSet.getString("delivery_address"));
			}
			return order;
		} catch (SQLException e) {
			System.out.print(e);
		} finally {
			if (resultSet != null) {
				try {
					resultSet.close();
				} catch (SQLException e) {
				}
			}
			if (statement != null) {
				try {
					statement.close();
				} catch (SQLException e) {
				}
			}
			if (connection != null) {
				try {
					connection.close();
				} catch (SQLException e) {
				}
			}
		}
		return null;
	}

	@Override
	public Order getOrderDetailByOrderNumber(String orderNumber) {
		// 不做实现
		return null;
	}
}
```

这里，值得注意的是，首先需要在类定义上添加 @Repository 注解，标明这是能够被 Spring 容器自动扫描的 Javabean，再在 @Repository 注解中指定这个 Javabean 的名称为"orderRawJdbcRepository"，方便 Service 层中根据该名称注入 OrderRawJdbcRepository 类。

可以看到，上述代码使用了 JDBC 原生 DataSource、Connection、PreparedStatement、ResultSet 等核心编程对象完成针对“order”表的一次查询。代码流程看起来比较简单，其实也比较烦琐，学到这里，我们可以结合上一课时的内容理解上述代码。

请注意，如果想运行这些代码，千万别忘了在 Spring Boot 的配置文件中添加对 DataSource 的定义。

## 使用 JdbcTemplate 操作数据库

首先引入对它的依赖，如下代码所示：
```xml
<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
```

JdbcTemplate 提供了一系列的 query、update、execute 重载方法应对数据的 CRUD 操作。

### 使用 JdbcTemplate 实现查询

基于 SpringCSS ，最简单的查询操作，对 OrderRawJdbcRepository 中的 getOrderById 方法进行重构。为此，构建了一个新的 OrderJdbcRepositoryImpl 类并同样实现了 OrderRepository 接口，如下代码所示：
```java
package cn.happymaya.order.repository.impl;

import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.sql.Statement;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

import cn.happymaya.order.domain.Goods;
import cn.happymaya.order.domain.Order;
import cn.happymaya.order.repository.IOrderRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.dao.DataAccessException;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.PreparedStatementCreator;
import org.springframework.jdbc.core.ResultSetExtractor;
import org.springframework.jdbc.core.simple.SimpleJdbcInsert;
import org.springframework.jdbc.support.GeneratedKeyHolder;
import org.springframework.jdbc.support.KeyHolder;
import org.springframework.stereotype.Repository;

/**
 * 使用 JdbcTemplate 进行数据库访问
 */
@Repository("orderJdbcRepository")
public class OrderJdbcRepositoryImpl implements IOrderRepository {
	private JdbcTemplate jdbcTemplate;

	@Autowired
	public OrderJdbcRepositoryImpl(JdbcTemplate jdbcTemplate) {
        // 通过构造函数注入了 JdbcTemplate 模板类
		this.jdbcTemplate = jdbcTemplate;
	}
}

```

OrderJdbcRepository 的 getOrderById 方法实现过程如下代码所示：
```java
@Override
public Order getOrderById(Long orderId) {
	return jdbcTemplate.queryForObject("select id, order_number, delivery_address from `order` where id=?", this::mapRowToOrder, orderId);
}
```

这里使用了 JdbcTemplate 的 queryForObject 方法执行查询操作，该方法传入目标 SQL、参数以及一个 RowMapper 对象。其中 RowMapper 定义如下：
```java
package org.springframework.jdbc.core;

import java.sql.ResultSet;
import java.sql.SQLException;
import org.springframework.lang.Nullable;

@FunctionalInterface
public interface RowMapper<T> {
    @Nullable
    T mapRow(ResultSet var1, int var2) throws SQLException;
}

```

从 mapRow 方法定义中，不难看出 RowMapper 的作用就是处理来自 ResultSet 中的每一行数据，并将来自数据库中的数据映射成领域对象。例如，使用 getOrderById 中用到的 mapRowToOrder 方法完成对 Order 对象的映射，如下代码所示：
```java
private Order mapRowToOrder(ResultSet rs, int rowNum) throws SQLException {
	return new Order(rs.getLong("id"), rs.getString("order_number"), rs.getString("delivery_address"));
}
```

getOrderById 方法实际上只是获取了 Order 对象中的订单部分信息，并不包含商品数据。

又设计一个 getOrderDetailByOrderNumber 方法，根据订单编号获取订单以及订单中所包含的所有商品信息，如下代码所示：

```java
	@Override
	public Order getOrderDetailByOrderNumber(String orderNumber) {
		//获取Order基础信息
		Order order = jdbcTemplate.queryForObject(
				"select id, order_number, delivery_address from `order` where order_number=?", this::mapRowToOrder,
				orderNumber);

		if (order == null)
			return order;

		//获取Order与Goods之间的关联关系，找到给Order中的所有GoodsId
		Long orderId = order.getId();
		List<Long> goodsIds = jdbcTemplate.query("select order_id, goods_id from order_goods where order_id=?",
				new ResultSetExtractor<List<Long>>() {
					public List<Long> extractData(ResultSet rs) throws SQLException, DataAccessException {
						List<Long> list = new ArrayList<Long>();
						while (rs.next()) {
							list.add(rs.getLong("goods_id"));
						}
						return list;
					}
				}, orderId);

		//根据GoodsId分别获取Goods信息并填充到Order对象中
		for (Long goodsId : goodsIds) {
			Goods goods = getGoodsById(goodsId);
			order.addGoods(goods);
		}

		return order;
	}
```

## 使用 JdbcTemplate 实现插入

通过 update 方法实现数据的插入和更新。针对 Order 和 Goods 中的关联关系，插入一个 Order 对象需要同时完成两张表的更新，即 order 表和 order_goods 表，因此插入 Order 的实现过程也分成两个阶段，如下代码所示的 addOrderWithJdbcTemplate 方法展示了这一过程：
```java
private Order addOrderDetailWithJdbcTemplate(Order order) {
	//插入Order基础信息
	Long orderId = saveOrderWithJdbcTemplate(order);

	order.setId(orderId);

	//插入Order与Goods的对应关系
	List<Goods> goodsList = order.getGoods();
	for (Goods goods : goodsList) {
		saveGoodsToOrderWithJdbcTemplate(goods, orderId);
	}

	return order;
}
```

这里同样先是插入 Order 的基础信息，然后再遍历 Order 中的 Goods 列表并逐条进行插入。其中的 saveOrderWithJdbcTemplate 方法如下代码所示：
```java
private Long saveOrderWithJdbcTemplate(Order order) {

	PreparedStatementCreator psc = new PreparedStatementCreator() {
		@Override
		public PreparedStatement createPreparedStatement(Connection con) throws SQLException {
			PreparedStatement ps = con.prepareStatement(
						"insert into `order` (order_number, delivery_address) values (?, ?)",
						Statement.RETURN_GENERATED_KEYS);
				s.setString(1, order.getOrderNumber());
			ps.setString(2, order.getDeliveryAddress());
			return ps;
		}
	};

	KeyHolder keyHolder = new GeneratedKeyHolder();
	jdbcTemplate.update(psc, keyHolder);

	return keyHolder.getKey().longValue();
}
```

上述 saveOrderWithJdbcTemplate 的方法比想象中要复杂，主要原因在于我们需要在插入 order 表的同时返回数据库中所生成的自增主键，因此，这里使用了 PreparedStatementCreator 工具类封装 PreparedStatement 对象的构建过程，并在 PreparedStatement 的创建过程中设置了 Statement.RETURN_GENERATED_KEYS 用于返回自增主键。然后我们构建了一个 GeneratedKeyHolder 对象用于保存所返回的自增主键。这是使用 JdbcTemplate 实现带有自增主键数据插入的一种标准做法，你可以参考这一做法并应用到日常开发过程中。

至于用于插入 Order 与 Goods 关联关系的 saveGoodsToOrderWithJdbcTemplate 方法就比较简单了，直接调用 JdbcTemplate 的 update 方法插入数据即可，如下代码所示：
```java
private void saveGoodsToOrderWithJdbcTemplate(Goods goods, long orderId) {
	jdbcTemplate.update("insert into order_goods (order_id, goods_id) " + "values (?, ?)", orderId, goods.getId());
}
```

需要实现插入 Order 的整个流程，先实现 Service 类和 Controller 类，如下代码所示：
```java
@Service
public class OrderService {

    @Autowired
    @Qualifier("orderJdbcRepository")
    private OrderRepository orderRepository;

    public Order addOrder(Order order) {
        return orderRepository.addOrder(order);
    } 
}

@RestController
@RequestMapping(value="orders")
public class OrderController {

    @RequestMapping(value = "", method = RequestMethod.POST)
    public Order addOrder(@RequestBody Order order) {
        Order result = orderService.addOrder(order);
     return result;
    }
}
```

通过 Postman 向http://localhost:8081/orders端点发起 Post 请求后，可以发现 order 表和 order_goods 表中的数据都已经正常插入。

## 使用 SimpleJdbcInsert 简化数据插入过程

虽然通过 JdbcTemplate 的 update 方法可以完成数据的正确插入，不禁发现这个实现过程还是比较复杂，尤其是涉及自增主键的处理时，代码显得有点臃肿。

Spring Boot 针对数据插入场景专门提供了一个 SimpleJdbcInsert 工具类，SimpleJdbcInsert 本质上是在 JdbcTemplate 的基础上添加了一层封装，提供了一组 execute、executeAndReturnKey 以及 executeBatch 重载方法来简化数据插入操作。

可以在 Repository 实现类的构造函数中对 SimpleJdbcInsert 进行初始化，如下代码所示:
```java
/**
 * 使用 JdbcTemplate 进行数据库访问
 */
@Repository("orderJdbcRepository")
public class OrderJdbcRepositoryImpl implements IOrderRepository {

	private JdbcTemplate jdbcTemplate;

	private SimpleJdbcInsert orderInserter;
	private SimpleJdbcInsert orderGoodsInserter;

	@Autowired
	public OrderJdbcRepositoryImpl(JdbcTemplate jdbcTemplate) {
		this.jdbcTemplate = jdbcTemplate;
		this.orderInserter = new SimpleJdbcInsert(jdbcTemplate).withTableName("`order`").usingGeneratedKeyColumns("id");
		this.orderGoodsInserter = new SimpleJdbcInsert(jdbcTemplate).withTableName("order_goods");
	}
}
```

可以看到，这里首先注入了一个 JdbcTemplate 对象，然后我们基于 JdbcTemplate 并针对 order 表和 order_goods 表分别初始化了两个 SimpleJdbcInsert 对象 orderInserter 和 orderGoodsInserter。其中 orderInserter 中还使用了 usingGeneratedKeyColumns 方法设置自增主键列。

基于 SimpleJdbcInsert，完成 Order 对象的插入就非常简单了，实现方式如下所示：
```java
private Long saveOrderWithSimpleJdbcInsert(Order order) {
	Map<String, Object> values = new HashMap<String, Object>();
	values.put("order_number", order.getOrderNumber());
	values.put("delivery_address", order.getDeliveryAddress());

	Long orderId = orderInserter.executeAndReturnKey(values).longValue();
	return orderId;
}
```

通过构建一个 Map 对象，然后把需要添加的字段设置成一个个键值对。通过SimpleJdbcInsert 的 executeAndReturnKey 方法在插入数据的同时直接返回自增主键。同样，完成 order_goods 表的操作只需要几行代码就可以了，如下代码所示：
```java
private void saveGoodsToOrderWithSimpleJdbcInsert(Goods goods, long orderId) {
    Map<String, Object> values = new HashMap<>();
    values.put("order_id", orderId);
    values.put("goods_id", goods.getId());
    orderGoodsInserter.execute(values);
}
```

这里用到了 SimpleJdbcInsert 提供的 execute 方法，可以把这些方法组合起来对 addOrderDetailWithJdbcTemplate 方法进行重构，从而得到如下所示的 addOrderDetailWithSimpleJdbcInsert 方法：
```java
// addOrderDetailWithSimpleJdbcInsert
private Order addOrderDetailWithSimpleJdbcInsert(Order order) {
	//插入Order基础信息
	Long orderId = saveOrderWithSimpleJdbcInsert(order);

	order.setId(orderId);
		
	//插入Order与Goods的对应关系
	List<Goods> goodsList = order.getGoods();
	for (Goods goods : goodsList) {
		saveGoodsToOrderWithSimpleJdbcInsert(goods, orderId);
	}

	return order;
}
```
在使用 JdbcTemplate 时，如果想要返回数据库的自增主键值有哪些实现方法？

JdbcTemplate 模板工具类是一个基于 JDBC 规范实现数据访问的强大工具，是一个优秀的工具类（本质上就是对 JDBC 的封装，也可以算是一种半自动化的 ORM 技术，谈不上是对 Mybatis 的替代方案，只是 Spring 提供的一个工具类）。它对常见的 CRUD 操作做了封装并提供了一大批简化的 API。