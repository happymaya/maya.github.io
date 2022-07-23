---
title:  Spring Data Common 之 Repository
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2020-09-05 21:33:00 +0800
categories: [Java, Spring]
tags:  [Spring Security, React]
math: true
mermaid: true
---

Spring Data 对整个数据操作做了很好的封装，其中 Spring Data Common 定义了很多公用的接口和**一些相对数据操作的公共实现**（如**分页排序**、**结果映射**、**Autiting 信息**、**事务**等），而 Spring Data JPA 就是 Spring Data Common 的关系数据库的查询实现。

Spring Data Common 的核心内容——Repository。

## Spring Data Common 的依赖关系

Spring Data Common 的依赖关系如下图：

通过上图的项目依赖，不难发现：
- 数据库连接用的是 JDBC，连接池用的是 HikariCP，强依赖 Hibernate；
- Spring Boot Starter Data JPA 依赖 Spring Data JPA；
- Spring Data JPA 依赖 Spring Data Commons。

## Repository 接口

Repository 是 Spring Data Common 里面的顶级父类接口，操作 DB 的入口类。

### 查看 Resposiory 源码

Common 里面的 Resposiory 源码，如下所示：
```java
package org.springframework.data.repository;
import org.springframework.stereotype.Indexed;

@Indexed
public interface Repository<T, ID> {
}
```

Resposiory 是 Spring Data 里面进行数据库操作顶级的抽象接口，里面什么方法都没有，但是如果任何接口继承它，就能得到一个 Repository，还可以实现 JPA 的一些默认实现方法。

Spring 利用 Respository 作为 DAO 操作的 Type，以及**利用 Java 动态代理机制**就可以实现很多功能，比如为什么接口就能实现 DB 的相关操作？这就是 Spring 框架的高明之处。

Spring 在做动态代理的时候，只要是它的子类或者实现类，再利用 T 类以及 T 类的 主键 ID 类型作为泛型的类型参数，就可以来标记出来、并捕获到要使用的实体类型，就能帮助使用者进行数据库操作。

### Repository 类层次关系

用工具 Intellij Idea，打开类 Repository.class，然后依次【导航】 → 【Hierchy 类型】，会得到如下图所示的结果：


通过该层次结构视图，就会明白基类 Repository 的用意，由此可知，存储库分为以下 4 个大类：
- `ReactiveCrudRepository`，这条线是**响应式编程**，主要**支持当前 NoSQL 方面**的操作，因为这方面大部分操作都是分布式的，所以由此可以看出 Spring Data 想统一数据操作的“野心”，即想提供关于所有 Data 方面的操作。目前 Reactive 主要有**Cassandra**、**MongoDB**、**Redis** 的实现
- `RxJava2CrudRepository`，这条线是为了支持 RxJava 2 做的标准响应式编程的接口；
- `CoroutineCrudRepository`，这条继承关系链是为了支持 Kotlin 语法而实现的；
- `CrudRepository`， 这条继承关系链，是 JPA 相关的操作接口。

Repository 继承关系图如下图所示：

**7 个大 Repository 接口：**
- `Repository(org.springframework.data.repository)`，没有暴露任何方法；
- `CrudRepository(org.springframework.data.repository)`，简单的 Curd 方法；
- `PagingAndSortingRepository(org.springframework.data.repository)`，带分页和排序的方法；
- `QueryByExampleExecutor(org.springframework.data.repository.query)`，简单 Example 查询；
- `JpaRepository(org.springframework.data.jpa.repository)`，JPA 的扩展方法；
- `JpaSpecificationExecutor(org.springframework.data.jpa.repository)`，JpaSpecification 扩展查询；
- `QueryDslPredicateExecutor(org.springframework.data.querydsl)`，QueryDsl 的封装。

**两大 Repository 实现类：**
- `SimpleJpaRepository(org.springframework.data.jpa.repository.support)`，JPA 所有接口的默认实现类；
- `QueryDslJpaRepository(org.springframework.data.jpa.repository.support)`，QueryDsl 的实现类。

### 一个 Repository 的实例
利用 UserRepository 继承 Repository 来实现对 User 的两个查询方法，如下：
```java
public interface UserRepository extends Repository<User, Long> {

    // 根据名称进行查询用户列表
    List<User> findByName(String name);

    // 根据用户的邮箱和名称查询
    List<User> findByEmailAndName(String email, String name);
}
```

### CrudRepository 接口

Repository 有了一定的掌握，它的直接子类 CurdRepository 接口提供了哪些方法，其中一部分如下：
- count(): long 查询总数返回 long 类型；
- void delete(T entity) 根据 entity 进行删除；
- void deleteAll(Iterable<? extends T> entities) 批量删除；
- void deleteAll() 删除所有；原理可以通过刚才的类关系查看，CrudRepository 的实现方法如下：
  ```java
  //SimpleJpaRepository里面的deleteALL方法
  public void deleteAll() {
      for (T element : findAll()) {
          delete(element);
      }
  }
  ```

通过源码我们可以看出 SimpleJpaRepository 里面的 deleteAll 是利用 for 循环调用 delete 方法进行删除操作。CrudRepository 提供的方法：
- void deleteById(ID id)：根据主键删除，查看源码会发现，其是先查询出来再进行删除；
- boolean existsById(ID id)：根据主键判断实体是否存在；
- Iterable<T> findAllById(Iterable ids); 根据主键列表查询实体列表；
- Iterable<T> findAll(); 查询实体的所有列表；
- Optional<T> findById(ID id); 根据主键查询实体，返回 JDK 1.8 的 Optional，这可以避免 null exception；
- <S extends T> S save(S entity); 保存实体方法，参数和返回结果可以是实体的子类；
- saveAll(Iterable<S> entities) : 批量保存，原理和 save方法相同，我们去看实现的话，就是 for 循环调用上面的 save 方法。

上面这些方法是 CrudRepository 对外暴露的常见的 Crud 接口，在对数据库进行 Crud 的时候就会运用到，如打算对 User 实体进行 Curd 操作，来看一下应该怎么写，如下所示：
```java
public interface UserRepository extends CrudRepository<User,Long> {
}
```
通过 UserRepository 继承 CrudRepository，这个时候的 UserRepository 就会有 CrudRepository 里面的所有方法。

这里需要注意一下 save 和 deleteById 的实现逻辑，这两种方法是怎么实现方式如下：
```java
//新增或者保存
public <S extends T> S save(S entity) {
   if (entityInformation.isNew(entity)) {
      em.persist(entity);
      return entity;
   } else {
      return em.merge(entity);
   }
}

//删除
public void deleteById(ID id) {
   Assert.notNull(id, ID_MUST_NOT_BE_NULL);
   delete(findById(id).orElseThrow(() -> new EmptyResultDataAccessException(String.format("No %s entity with id %s exists!", entityInformation.getJavaType(), id), 1)));
}
```

在进行 Update、Delete、Insert 等操作之前。

上面的源码，会通过 findById 先查询一下实体对象的 ID，然后再去对查询出来的实体对象进行保存操作。而如果在 Delete 的时候，查询到的对象不存在，则直接抛异常。

这里特别强调了一下 Delete 和 Save 方法，是因为在实际工作中，看到有的同事画蛇添足：自己在做 Save 的时候先去 Find 一下，其实是没有必要的，Spring JPA 底层都考虑到了。当用任何第三方方法的时候，最好先查一下其源码和逻辑或者 API，然后再写出优雅的代码。

关于 entityInformation.isNew（entity），如果当传递的参数里面没有 ID，则直接 insert；若当传递的参数里面有 ID，则会触发 select 查询。此方法会去看一下数据库里面是否存在此记录，若存在，则 update，否则 insert。

### PagingAndSortingRepository 接口

PagingAndSortingRepository 接口，该接口也是 Repository 接口的子类，主要用于**分页查询**和**排序查询**。

PagingAndSortingRepository 源码发现有两个方法，分别是用于分页和排序的时候使用的，如下所示：
```java
package org.springframework.data.repository;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Sort;

@NoRepositoryBean
public interface PagingAndSortingRepository<T, ID> extends CrudRepository<T, ID> {
	Iterable<T> findAll(Sort sort); （1）
	Page<T> findAll(Pageable pageable); （2）
}
```
其中：
- 第一个方法 findAll 参数是 Sort，是根据排序参数，实现不同的排序规则获取所有的对象的集合；
- 第二个方法 findAll 参数是 Pageable，是根据分页和排序进行查询，并用 Page 对返回结果进行封装。而 Pageable 对象包含 Page 和 Sort 对象。 

通过【Repository 继承关系图】和上的一大堆源码可以看到，PagingAndSortingRepository 继承了 CrudRepository，进而拥有了父类的方法，并且增加了分页和排序等对查询结果进行限制的通用的方法。

PagingAndSortingRepository 和 CrudRepository 都是 Spring Data Common 的标准接口，那么实现类是什么呢？
- 如果采用 JPA，那对应的实现类就是 Spring Data JPA 的 jar 包里面的 SimpleJpaRepository；
- 如果是其他 NoSQL的 实现如 MongoDB，那实现就在 Spring Data MongoDB 的 jar 里面的 MongoRepositoryImpl。

#### PagingAndSortingRepository 使用案例

第一步：定一个 UserRepository 类来继承 PagingAndSortingRepository 接口，实现对 User 的分页和排序操作，实现源码如下：
```java
package com.example.jpa.example1;
import org.springframework.data.repository.PagingAndSortingRepository;
public interface UserRepository extends PagingAndSortingRepository<User,Long> {
}
```


第二步：利用 UserRepository 直接继承 PagingAndSortingRepository 即可，而 Controller 里面就可以有如下用法了：
```java
/**
 * 验证排序和分页查询方法，Pageable的默认实现类：PageRequest
 * @return
 */
@GetMapping(path = "/page")
@ResponseBody
public Page<User> getAllUserByPage() {
   return userRepository.findAll(PageRequest.of(1, 20,Sort.by(new Sort.Order(Sort.Direction.ASC,"name"))));
}

/**
 * 排序查询方法，使用Sort对象
 * @return
 */
@GetMapping(path = "/sort")
@ResponseBody
public Iterable<User> getAllUsersWithSort() {
   return userRepository.findAll(Sort.by(new Sort.Order(Sort.Direction.ASC,"name")));
}
```

### JpaRepository 接口

上面的那些都是 Spring Data 为了兼容 NoSQL 而进行的一些抽象封装，而从 JpaRepository 开始是对关系型数据库进行抽象封装。从类图可以看出来它继承 PagingAndSortingRepository 类，也就继承了其所有方法，并且其实现类也是 SimpleJpaRepository。从类图上还可以看出 JpaRepository 继承和拥有了 QueryByExampleExecutor 的相关方法。


JpaRepository 里面重点新增了批量删除，优化了批量删除的性能，类似于之前 SQL 的 batch 操作，并不是像上面的 deleteAll 来 for 循环删除。其中 flush() 和 saveAndFlush() 提供了手动刷新 session，把对象的值立即更新到数据库里面的机制。


JPA 是 由 Hibernate 实现的，所以有 session 一级缓存的机制，当调用 save() 方法的时候，数据库里面是不会立即变化的。

用 UserRepository 直接继承 JpaRepository，来实现 JPA 的相关方法，如下所示：
```java
public interface UserRepository extends JpaRepository<User,Long> {
}
```

这样 controller 里面就可以直接调用 JpaRepository 及其父接口里面的所有方法了。

### Repository 的实现类 SimpleJpaRepository

关系数据库的所有 Repository 接口的实现类就是 SimpleJpaRepository，如果有些业务场景需要进行扩展了，可以继续继承此类，如 QueryDsl 的扩展（虽然不推荐使用了，但可以参考它的做法，自定义自己的 SimpleJpaRepository），如果能将此类里面的实现方法看透了，基本上 JPA 中的 API 就能掌握大部分内容。

通过 Debug 视图看一下动态代理过程，会发现 UserRepository 的实现类是 Spring 启动的时候，利用 Java 动态代理机制帮我们生成的实现类，而真正的实现类就是 SimpleJpaRepository。

通过上面【类的继承关系图】也可以知道 SimpleJpaRepository 是 Repository 接口、CrudRepository 接口、PagingAndSortingRepository 接口、JpaRepository 接口的实现。其中，SimpleJpaRepository 的部分源码如下：
```java
@Repository
@Transactional(readOnly = true)
public class SimpleJpaRepository<T, ID> implements JpaRepository<T, ID>, JpaSpecificationExecutor<T> {
	private static final String ID_MUST_NOT_BE_NULL = "The given id must not be null!";
	private final JpaEntityInformation<T, ?> entityInformation;
	private final EntityManager em;
	private final PersistenceProvider provider;
	private @Nullable CrudMethodMetadata metadata;
	......

	@Transactional
	public void deleteAllInBatch() {
		em.createQuery(getDeleteAllQueryString()).executeUpdate();
	}
	......
```

通过此类的源码，可以清晰地看出 SimpleJpaRepository 的实现机制，是通过 EntityManger 进行实体的操作，而 JpaEntityInforMation 里面存在实体的相关信息和 Crud 方法的元数据等。

利用 Java 动态代理机制帮我们生成的实现类，那么关于动态代理的实现，可以在 RepositoryFactorySupport 设置一个断点，启动的时候，在断点处就会发现 UserRepository 的接口会被动态代理成 SimpleJapRepository 的实现。

这里需要注意的是每一个 Repository 的子类，都会通过这里的动态代理生成实现类。

## Repository 接口给我的启发

在接触了 Repository 的源码之后，在工作中遇到过一些类似需要抽象接口和写动态代理的情况，所以对于 Repository 的源码，我受到了一些启发：

第一，上面的 7 个大 Repository 接口，在使用的时候可以根据实际场景，来继承不同的接口，从而选择暴露不同的 Spring Data Common 给我们提供的已有接口。这其实利用了 Java 语言的 interface 特性，在这里可以好好理解一下 interface 的妙用。

第二，利用源码也可以很好地理解一下 Spring 中动态代理的作用，可以利用这种思想，在改善 MyBatis 的时候使用。
