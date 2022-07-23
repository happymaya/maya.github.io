---
title:  利用单元测试和集成测试
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2020-09-01 23:33:00 +0800
categories: [Java, Spring]
tags:  [Spring Security, React]
math: true
mermaid: true
---

如果测试用例非常完备，是可以提升团队体效率的

## Spring Data JPA 单元测试

实际工作中我们免不了要和 Repository 打交道，那么这层的测试用例应该怎么写呢？怎么才能提高开发效率呢？关于 JPA 的 Repository，下面我们分成两个部分来介绍：了解基本语法；分析最佳实践。

Spring Data JPA Repository 的测试用例
测试用例写法步骤如下。

第一步：引入 test 的依赖，gradle 的语法如下所示。
```java
testImplementation 'com.h2database:h2'
testImplementation 'org.springframework.boot:spring-boot-starter-test'
```

第二步：利用项目里面的实体和 Repository，假设项目里面有 Address 和 AddressRepository，代码如下所示。

第三步：新建 RepsitoryTest，@DataJpaTest 即可
```java
@DataJpaTest
public class AddressRepositoryTest {

    @Autowired
    private AddressRepository addressRepository;

    //测试一下保存和查询
    @Test
    public  void testSave() {
        Address address = Address.builder().city("shanghai").build();
        addressRepository.save(address);
        List<Address> address1 = addressRepository.findAll();
        address1.stream().forEach(address2 -> System.out.println(address2);
    }
}
```
通过上面的测试用例可以看到，直接添加了 @DataJpaTest 注解，然后利用 Spring 的注解 @Autowired，引入了 spring context 里面管理的 AddressRepository 实例。换句话说，我们在这里面使用了集成测试，即直接连接的数据库来完成操作。

第四步：直接运行上面的测试用例，可以得到如下图所示的结果。
通过测试结果，可以发现：
- 测试方法默认都会开启一个事务，测试完了之后就会进行回滚；
- 执行了 insert 和 select 两种操作；
- 如果开启了 Session Metrics 的日志的话，也可以观察出来其发生了一次 connection。

通过这个案例，可以知道 Repository 的测试用例写起来还是比较简单的，其中主要利用了 @DataJpaTest 的注解


## Repository 的测试