---
title: 索引设计
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2020-07-11 17:53:22 +0800
categories: [Database, MySQL]
tags:  [MySQL, Index Design]
math: true
mermaid: true
---

## 索引的定义

> 数据库索引是一种数据结构，以额外写入和存储空间为代价来提高数据库表上数据检索操作的速度。
>
> 通俗来说，索引类似于书的目录，根据其中记录的页码可以快速找到所需的内容。
>
> ——《维基百科》

MySQL 官方对索引（Index）的定义是**存储引擎用于快速查找记录的一种数据结构：**

- 索引是物理数据页，数据库页大小（Page Size）决定了一个页可以存储多少个索引行，以及需要多少页来存储指定大小的索引；
- 索引可以加快检索速度，但同时也降低索引列插入、删除、更新的速度，索引维护需要代价。

索引设计的理论知识有：

- 二分查找；
- 哈希表；
- B+Tree.

## 二分查找法

**二分查找法也叫作折半查找法，它是在有序数组中查找指定数据的搜索算法。**

> 维基百科对二分查找法的定义：
>
> 1. Set $L$ to $0$ and $R$ to $n -1$ .
>
> 2. If $L > R$, the search terminates as unsuccessful .
>
>    3. Set $m$ (the position of the middle element) to the ==floor== of $\frac{L+R}{2}$ ，which is the greatest integer less than or equal to $\frac{L+R}{2}$
>
> 4. If $A_m < T$  , set $L$ to $m+1$ and go to step 2.
>5. If $A_m > T$ , set $R$ to $m -1$ and go to step 2.
>    6. Now $A_m > T$, the search is done; return m.
>

### 优点

- 等值查询；
- 范围查询性能优秀。

### 缺点

更新数据、新增数据、删除数据维护成本高。

### 栗子

> 有序数组 [1-71] 有 17 个值，即在有序数组 $[A_0-A_{16}] $ 中希望找到 $Target(7)$ 所在的位置，首选确定下标 $L$ 为 0，下标 $R$ 为 16，下标 m 为 floor [$\frac{L+R}{2}$]，即向下取整数。

#### 第一次查询

 $下标\ L=0, R = 16, m = floor[\frac{0+16}{2}] = 8$ 

获得 A8 的值为 14，因为 A8(14) >Target(7) 则设置 R=m-1=7，如下图所示。

![](https://images.happymaya.cn/assert/db/mysql/mysql-0601.png)

#### 第二次查询

$下标\ L=0, R = 7, m = floor[\frac{0+7}{2}] = 3$ 

获得 $A_3$ 的值为 6，$A_3(6) < Target(7)$  则设置下标 $L=m+1=4$ ，如下图所示：

![](https://images.happymaya.cn/assert/db/mysql/mysql-0602.png)

#### 第三次查询

$下标\ L=4, R = 7, m = floor[\frac{4+7}{2}] = 5$ ，

获得 $A_5$ 的值为 8，$A_5(8) = Target(7)$，则设置下标 $R = m - 1 = 4$，如下图所示：

![](https://images.happymaya.cn/assert/db/mysql/mysql-0603.png)

#### 第四次查询

$下标\ L=4, R = 4, m = floor[\frac{4+2}{2}] = 4$ ，

获得 $A_4$ 的值为 7，$A_4(7) = Target(7)$，查询结束，如下图所示：

![](https://images.happymaya.cn/assert/db/mysql/mysql-0604.png)

此次查询经过 4 次二分查找后找到目标数据 7，如果在查询过程中出现下标 $L > R $ 的情况，则表示目标元素不在有序数组内，结束查询。

二分查找是索引实现的理论基础，



## 索引原理

数据库查询是数据库的核心功能，而索引又是作为加速查询的重要技术手段。

对于索引数据结构的选择，本质是贴合当前数据读写的硬件环境，选择一个优秀的数据结构。进行数据存储和遍历。

在数据库中，大部分索引都是通过 ==B+Tree== 来实现的。当然也涉及其他数据结构，在 MySQL 中除了 ==B+Tree== 索引外，还需要关注==Hash 索引==。

### Hash 索引

哈希表是数据库中哈希索引的基础，根据键值 `<key,value>` 存储数据的结构。

简单说：

- 哈希表是使用哈希函数，将索引列计算到桶或槽的数组，
- 实际存储是根据哈希函数，将 key 换算成确定的存储位置，并将 value 存放到该数组位置上；
- 访问时，只需要输入待查找的 key，即可通过哈希函数计算，得出确定的存储位置并读取数据。

如下图所示：

![](https://images.happymaya.cn/assert/db/mysql/mysql-0605.png)

姓名作为 key，通过哈希函数对姓名字段数据进行计算，得到哈希码并存放到桶或槽的数组中，同时存放指向真实数据行的指针作为 value，形成哈希表。

#### 哈希索引的实现

- 数据库中哈希索引是基于哈希表实现，
- 对于==哈希索引列==的数据通过 Hash 算法计算，得到对应==索引列的哈希码==形成==哈希表==，由==哈希码及哈希码指向的真实数据行的指针==组成了==哈希索引==；
- 哈希索引的应用场景是只在对==哈希索引列的等值查询==才有效。

如下图所示：

![](https://images.happymaya.cn/assert/db/mysql/mysql-0606.png)

根据表中的 name 字段构建 Hash 索引，通过 Hash 算法对每一行 name 字段的数据进行计算，得出 Hash 码。由 Hash 码及 Hash 码指向真实数据行的指针组成了哈希索引。

#### 优点

因为哈希索引只存储哈希值和行指针，不存储实际字段值，所以其结构紧凑，查询速度也非常快，在无哈希冲突的场景下访问哈希索引一次即可命中。

#### 缺点

- 哈希索引只适用于等值查询，包括 =、IN()、<=> （安全等于， select null <=> null 和 select null=null 是不一样的结果) ；
- 不支持范围查询；
- 另外，哈希索引的性能跟哈希冲突数量成反比，哈希冲突越多其维护代价越大性能越低。 

#### Hash 碰撞处理

Hash 碰撞是指，**不同索引列值计算出相同的哈希码，**

如上图所示， 表中 name 字段为 John Smith 和 Sandra Dee 两个不同值根据 Hash 算法计算出来的哈希码都是 152，这就表示出现了 Hash 碰撞。 

对于 Hash 碰撞通用的处理方法是使用**链表**：

- 将 Hash 冲突碰撞的元素形成一个链表，发生冲突时在链表上进行二次遍历找到数据。 

Hash 碰撞跟选择的 Hash 算法有关系，为了减少 Hash 碰撞的概率：

- 优先选择避免 Hash 冲突的 Hash 算法，例如：
  - 使用 Percona Server 的函数 FNV64() ，其哈希值为 64 位，出现 Hash 冲突的概率要比 CRC32 小很多；

- 其次是考虑性能，优先**选择数字类型的 Hash 算法，因为字符串类型的 Hash 算法不仅浪费空间而且不方便进行比较。** 

常见的 CRC32、SHA1 和 MD5 Hash 函数生成的返回值如下图所示：

![](https://images.happymaya.cn/assert/db/mysql/mysql-0607.png)

因此，Hash 算法优先级：==**`FNV64 > CRC32 （大数据量下 Hash 冲突概率较大）> MD5 > SHA1。**==

#### MySQL 如何使用 Hash

在 MySQL 中使用 Hash 索引主要是分为：

- Memory 存储引擎，原生支持的 Hash 索引 ；
- InnoDB 自适应哈希索引；
- NDB 集群的哈希索引 3 类。

![](https://images.happymaya.cn/assert/db/mysql/mysql-0609.png)

如上图所示，Memory 存储引擎创建表时即可原生显式创建并使用 Hash 索引。

InnoDB 存储引擎，虽然不能原生显示创建 Hash 索引，但是可以伪造哈希索引来加速定值查询的性能。例如：为超长文本（如网站 URL）进行 Hash 计算后的字段 url_hash 创建 B+Tree 索引，获得 Hash 索引的功能。

关于哈希索引，InnoDB 提供了 InnoDB 自适应哈希索引的强大功能，

**InnoDB 自适应哈希索引是为了提升查询效率，InnoDB 存储引擎会监控表上各个索引页的查询，当 InnoDB 注意到某些索引值访问非常频繁时，会在内存中基于 B+Tree 索引再创建一个哈希索引，使得内存中的 B+Tree 索引具备哈希索引的功能，即能够快速定值访问频繁访问的索引页。**

当满足下面三个条件时，InnoDB 为整个 block 构建 AHI 记录项：

- 分析使用自适应哈希索引（AHI）可以成功查询的次数是否超过 block 上记录数的1/16；
- btr_search_info::n_hash_potential大于或等于BTR_SEARCH_BUILD_LIMIT (100)，表示为 SQL 查询能够连续 100 次成功使用 AHI；
- 尚未为当前 block 构造索引或者当前 block 上已经构建了 AHI 索引且 block->n_hash_helps 大于 page 上记录数的两倍或者当前 block上 推荐的前缀索引列发生了变化 。

为什么要为 B+Tree 索引页二次创建自适应哈希索引呢：

- 因为 B+Tree 索引的查询效率取决于 B+Tree 的高度，在数据库系统中通常 B+Tree 的高度为 3～4 层，所以访问数据需要做 3～4 次的查询。
- 而 Hash 索引访问通常一次查找就能定位数据（无 Hash 碰撞的情况），其等值查询场景 Hash 索引的查询效率要优于 B+Tree。

 自适应哈希索引的建立使得 InnoDB 存储引擎能自动根据索引页访问的频率和模式自动地为某些热点页建立哈希索引来加速访问。另外 InnoDB 自适应哈希索引的功能，用户只能选择开启或关闭功能，无法进行人工干涉。



功能开启后可以通过 `Show Engine Innodb Status` 看到当前自适应哈希索引的使用情况：

```bash
Hash table size 276707， node heap has 0 buffer(s)
0.00 Hash searches/s， 0.00 non-Hash searches/s
// 可以看到 Hash table 的大小，使用情况及每秒使用 AHI 和非 AHI 搜索的情况。
```

#### B+ 索引

在数据库中大部分索引都是通过 B+Tree 来实现的。

对于 B+Tree 具体的定义可以参考《数据结构》等相关书籍。 

在 MySQL 数据库中讨论索引时，如果没有明确指定类型，则默认是指使用 B+Tree 数据结构进行存储，其说法等价于 B+Tree、B-Tree、BTREE（看到创建索引语句为 BTREE 也不要惊讶，等同于 B+Tree）。

 如下图所示为一个简单的、标准的 B+tree，每个节点有 K 个键值和 K+1 个指针。

![](https://images.happymaya.cn/assert/db/mysql/mysql-0611.png)

对于 MySQL 存储引擎而言，其实际使用的 B+Tree 索引是为了满足：

- 数据读写性能；
- 适配磁盘访问模式优化后的数据结构，每一个叶子节点都包含指向下一个叶子节点的指针。

 在 MySQL 中，索引是在存储引擎层而非服务器层实现的，所以不同存储引擎层支持的索引类型可以不同。例如：

- 虽然 MyISAM 和 InnoDB 的索引都是使用 B+Tree 实现的，但是其实际数据存储结构有不少差异；

- 下图中 B+Tree 示例一共 2 层，图中每个页面都已经被随机编号（编号可以认定为页面号），其中页面号为 20 的页面是 B+Tree 的根页面（根页面通常是存放在内存中的），根页面存储了 <key+pageno>，pageno 是指向具体叶子节点的页面号。其他页面都是叶子节点，存放了具体的数据 <key+data>

   ![](https://images.happymaya.cn/assert/db/mysql/mysql-0612.png)      



**B+Tree 索引能够快速访问数据，就是因为存储引擎可以不再需要通过全表扫描来获取数据，而是从索引的根结点（通常在内存中）开始进行二分查找，根节点的槽中都存放了指向子节点的指针，存储引擎根据这些指针能够快速遍历数据。**

例如：通过页面号为 20 的根节点可以快速得知 Key<10 的数据在 pageno 33 的页面，key在 [10,16) 范围的数据在 pageno 56 的页面。

 叶子节点存放的 <key+data> ，对于真正要存放哪些数据还得取决于该 B+Tree 是聚簇索引（Clustered Index）还是辅助索引（Secondary Index）。

#### 聚簇索引和辅助索引

##### 聚簇索引

- **聚簇索引是一种数据存储方式，它表示表中的数据按照主键顺序存储，是索引组织表；**

- InnoDB 的聚簇索引是按照主键顺序构建 B+Tree，B+Tree 的叶子节点就是行记录，数据行和主键值紧凑地存储在一起。 这也意味着 InnoDB 的主键索引就是数据表本身，它按主键顺序存放了整张表的数据；
- 聚簇索引占用的空间就是整个表数据量的大小；
-  InnoDB 只能创建一个聚簇索引（假想下如果能支持多个聚簇索引，那就意味着一张表按不同排序规则冗余存储多份全表数据了）；
- 相比索引组织表，还有一种堆表类型，堆表是根据数据写入的顺序直接存储在磁盘上的。对于堆表而言，其主键和辅助索引唯一的区别就是键值是否唯一，两者都是根据索引列排序构建 B+Tree 的，在每个叶子节点加上指向堆表的行指针（row data pointer） 。堆表在各类数据库中也被广泛使用，MyISAM 存储引擎的表就是堆表。

#####  辅助索引

- InnoDB 辅助索引（也叫作二级索引）只是根据索引列构建 B+Tree，但在 B+Tree 的每一行都存了主键信息，加速回表操作。；
- 二级索引会比聚簇索引小很多， 通常创建辅助索引就是为了提升查询效率。
- 可以创建多个辅助索引。

## 索引类型

MySQL 中索引是在存储引擎层而非服务器层实现的，所以不同存储引擎层支持的索引类型可以不同。

在 MySQL 中不同存储引擎间支持的常见索引类型有：

- 哈希索引（Memory/InnoDB adaptive Hash index/NDB）；
- B+Tree 索引（MyISAM/InnoDB）；
- 全文索引（MyISAM/InnoDB）；
- 空间索引（MyISAM R-Tree）；
- 分形树索引（TokuDB Fractal Tree Index）。

如下表所示：

| 索引类型    | 存储引擎                              |
| ----------- | ------------------------------------- |
| 哈希索引    | Memory/InnoDB Adaptive Hash Index/NDB |
| B+Tree 索引 | MylSAM/InnoDB                         |
| 全文索引    | MylSAM/InnoDB                         |
| 空间索引    | MylSAM R-Tree                         |
| 分形树索引  | TokuDB Fractal Tree Index             |

在 MySQL InnoDB 中索引通常可以分为两大类：

- 主键索引（即聚簇索引）；
- 辅助索引（非聚簇索引） 。

对于没有指定主键的表，InnoDB 会自己选择合适字段为主键，其选择顺序如下：

- 显式主键；
- 第一个唯一索引（要求唯一索引所有列都非 NULL）；
- 内置的 6 字节 ROWID。

优先使⽤ UNSIGNED 自增列显示创建主键！



根据索引列个数和功能描述不同索引也可以分为：

- 联合索引，指在多个字段联合组建索引的。
- 覆盖索引，当通过索引即可查询到所有记录，不需要回表到聚簇索引时，这类索引也叫作覆盖索引；
- 主键查询是天然的覆盖索引，联合索引可以是覆盖索引。

看 SQL 语句是否使用到覆盖索引：

- 通常在查看执行计划时， Extra 列为 Using index 则表示优化器使用了覆盖索引。

优先考虑使用覆盖索引，这是因为如果 SQL 需要查询辅助索引中不包含的数据列时，就需要先通过辅助索引查找到主键值，然后再回表通过主键查询到其他数据列（即回表查询），需要查询两次。而覆盖索引能从索引中直接获取查询需要的所有数据，从⽽避免回表进行二次查找，节省IO，效率较⾼。例如：

```sql
--- 如果 uid 不是主键，那可以将索引添加为 index(uid，email)，以获得查询性能提升。
SELECT email，uid FROM user_email WHERE uid=xx;
```

## 索引使用技巧

主要有：

- ==谓词==，就是条件表达式，通俗讲就是过滤字段，如下面的 SQL 语句：

  ```sql
  select * from city where city="Beijing" and last_update="2019-08-01";
  ```

  可以拆解为：

  - 简单谓词：city 和 last_update
  - 组合谓词：city and last_update

- ==过滤因子==，谓词之后就可以计算谓词的过滤因子：

  - **直接描述了谓词的选择性**
  - 表示满足谓词条件的记录行数所占比例
  - 过滤因子越小意味着能过滤越多数据，需要在这类谓词字段上创建索引；
  - 过滤因子的计算算法，就是**满足谓词条件的记录行数除以表总行数**：
    - $简单谓词的过滤因子 = \frac{谓词结果集的数量}{表总行数}$
    - $组合谓词的过滤因子 = 谓词 \ 1 \ 的过滤因子 * 谓词 \ 2 \ 的过滤因子$ 

- ==基数（Cardinality）==，**是某个键值去重后的行数， 索引列不重复记录数量的预估值，MySQL 优化器会依赖于它。**

- ==选择率==，$ \frac{count(distinct city)}{count(*)}$，越接近 1 ，越适合创建索引，例如主键和唯一键的选择率都是 1。

- ==回表==，指无法通过索引扫描访问所有数据，需要回到主表进行数据扫描并返回。

### 栗子

借助谓词、过滤因子、基数、选择率以及回表可以创建高效索引。

用一个例子来看下，如何快速根据 SQL 语句计算谓词、过滤因子、基数和选择率。

根据 SQL 语句可以快速得到谓词信息：

- 简单谓词 city 和 last_update；
- 组合谓词 city and last_update。

**计算每个谓词信息的过滤因子，过滤因子越小表示选择性越强，字段越适合创建索引。**例如：

- $city \ 的过滤因子 = \frac{谓词 city 结果集的数量 }{表总行数}= \frac{select \  count(*) \ from \ city \ where \ city \ =  \ 'BeiJing' } {select \ count(*) \ from  \ city  } = 0.2$；

- $last\_update \ 的过滤因子 = \frac{谓词 last\_update \ 结果集的数量}{表总行数}  =  \frac{select \ count(*) \ from \ city \ where \ last\_update \ = \ ‘2019-08-01’}{select \ count(*) \ from \ city \ } = \ 0.1$；
- $组合谓词 \ = \ city \ 过滤因子 * last\_update \ 过滤因子 = 0.2 * 0.1 = 0.02 $，组合谓词的过滤因子为 2%，即只有表总行数的 2% 匹配过滤条件，可以考虑创建组合索引  (city，last_update)。

除谓词信息、过滤因子外，字段基数和选择率信息可以帮助了解字段数据的分布情况。

### 统计信息

MySQL InnoDB 的==统计信息==参考==基数 Cardinality==的信息。

Cardinality 能快速告知字段的选择性，高选择性字段有利于创建索引。

优化器在选择执行计划时会依赖该信息，通常这类信息也叫作统计信息，数据库中对于统计信息的采集是在存储引擎层进行的。

执行下面命令会看到 Cardinality，同时也会触发 MySQL 数据库对 Cardinaltiy 值的统计：

```sql
show index from table_name;
```

除此之外，还有三种更新策略：

- 触发统计：Cardinality 统计信息更新发生在 INSERT 和 UPDATE 时，InnoDB 存储引擎内部更新的 Cardinality 信息的策略为：表中超过1/16的数据发生变化：`stat_modified_counter > 2000 000 000 （20亿）`；
- 采样统计（sample）：为了减少统计信息更新造成的资源消耗，数据库对Cardinality 通过采样来完成统计信息更新，每次随机获取 innodb_stats_persistent_sample_pages 页的数量进行 Cardinality 统计；
- 手动统计：`alter table table_name engine=innodb` 或 `analyze table table_name`，当发现优化器选择错误的执行计划或没有走理想的索引时，执行 SQL 语句来手动统计信息有时是一种有效的方法。

由于采样统计的信息是随机获取 8 个（8 是由 ==`innodb_stats_transient_sample_pages`== 参数指定）页面数进行分析，这就意味着下一次随机的 8 个页面可能是其他页面，其采集页面的 Carinality 也不同。因此当表数据无变化时也会出现 Cardinality 发生变化的情况，如下图所示：

![](https://images.happymaya.cn/assert/db/mysql/mysql-0614.png)

关于统计信息的采集，涉及如下主要参数：

- ==information_schema_stats_expiry:86400==，Cardinality 存放过期时间，设置为 0 表示实时获取统计信息，严重影响性能，建议设置默认值并通过手动刷新统计信息；
- ==innodb_stats_auto_recalc:ON==，是否自动更新统计信息，默认即可；
- ==innodb_stats_include_delete_marked :OFF==，计算持久化统计信息时 InnoDB 是否包含 Delete-marked 记录，默认即可；
- ==innodb_stats_method:nulls_equal==，用来判断如何对待索引中出现的 NULL 值记录，默认为 nulls_equal，表示将 NULL 值记录视为相等的记录；
- ==innodb_stats_on_metadata==， 默认值 OFF，执行 SQL 语句 ANALYZE TABLE、SHOW TABLE STATUS、SHOQ INDEX，以及访问 INFORMATION_SCHEMA 架构下表 tables和statistics 时，是否重新计算索引的 Cardinality 值；
- ==innodb_stats_persistent:ON==，表示通过 ANALYZE TABLE 计算得到的 Cardinality值存放到磁盘上；
- ==innodb_stats_persistent_sample_pages:20==，表示 ANALYZE TABLE 更新Cardinality 值时采样页的数量；
- ==innodb_stats_transient_sample_pages:8==，表示每次统计 Cardinality 时采样页的数量，默认为 8。

### 索引使用细节

1. 首先是创建索引后，如何确认 SQL 语句是否走索引了？

2. 创建索引后通过查看执行 SQL 语句的执行计划，即可知道 SQL 语句是否走索引；

3. 执行计划重点关注跟索引相关的关键项有：

   1. type

   2. ==possible_keys==，表示查询可能使用的索引

   3. ==key==，表示真正实际使用的索引

   4. ==key_len==，表示使用索引字段的长度

   5. ref

   6. ==Extra== ，Extra 显示 use index 时就表示该索引是覆盖索引，通常性能排序的结果是 usd index > use where > use filsort，如下图所示：

      ![](https://images.happymaya.cn/assert/db/mysql/mysql-0615.png)



#### key_len

当索引选择组合索引时，通过计算 key_len 来了解有效索引长度对索引优化也是非常重要的。

**key_len 表示得到结果集所使用的选择索引的长度[字节数]，不包括 order by，也就是说如果 order by 也使用了索引则 key_len 不计算在内。**

key_len 计算规则从两个方面考虑：

- 索引字段的数据类型，根据索引字段的定义可以分为变长和定长两种数据类型：

  - 定长数据类型，比如 char、int、datetime，需要有是否为空的标记，这个标记需要占用 1 个字节；
  - 变长数据类型，比如 Varchar，除了是否为空的标记外，还需要有长度信息，需要占用 2 个字节；（备注：当字段定义为非空的时候，是否为空的标记将不占用字节）。

- 表、字段使用的字符集：

  不同的字符集计算的 key_len 不一样，例如，GBK 编码的是一个占用 2 个字节大小的字符，UTF8 编码的是一个占用 3 个字节大小的字符。

  举例说明：在四类字段上创建索引后的 key_len 的计算：

  - Varchr(10) 变长字段且允许 NULL:10*(Character Set：utf8=3，gbk=2，latin1=1)+1（标记是否为 NULL 需要 1 个字节）+ 2（变长字段存储长度信息需要 2 个字节）；*
  - *Varchr(10) 变长字段且不允许 NULL:10*(Character Set：utf8=3，gbk=2，latin1=1)+2（变长字段存储长度信息需要2个字节），非空不再需要占用字节来标记是否为空；
  - Char(10) 固定字段且允许 NULL:10*(Character Set：utf8=3，gbk=2，latin1=1)+1（标记是否为 NULL 需要 1 个字节）；*
  - *Char(10) 固定字段且不允许 NULL:10*(Character Set：utf8=3，gbk=2，latin1=1)，非空不再需要占用字节来标记是否为空。



### 最左前缀匹配原则

通过 key_len 计算也帮助了解索引的最左前缀匹配原则。

最左前缀匹配原则，是指在使用 B+Tree 联合索引进行数据检索时，MySQL 优化器会读取谓词（过滤条件），并按照联合索引字段创建顺序一直向右匹配，直到遇到范围查询或非等值查询后停止匹配，此字段之后的索引列不会被使用，这时计算 key_len 可以分析出联合索引实际使用了哪些索引列。

### 设计性能索引

![](https://images.happymaya.cn/assert/db/mysql/mysql-0616.png)

创建一个 test 表。 在 a、b、c 上创建索引，执行表中的 SQL 语句，快速定位语句孰好孰坏。

首先分析 key_len， 因为 a、b、c 不允许 NULL 的 varchar(50)，那么，每个字段的 key_len 为 50×4+2=202，整个联合索引的 key_len 为 202×3=606。

| 序号 | SQL 语句                                                    | 是否索引                                                     | 结果                                                         |
| ---- | ----------------------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| sql1 | select * from test where a=? and b=? and c=?;               | key: idx_a_b_c <br/>key_len: 606 <br/>Extra: Using index     | 可以使用覆盖索引，性能好                                     |
| sql2 | select * from test where a=? and b=? and c=? order by c;    | key: idx_a_b_c <br/>key_len: 606 <br/>Extra: Using index     | 可以使用覆盖索引同时可以避免排序，性能好                     |
| sql3 | select * from test where b=? and c=?;                       | key: idx_a_b_c <br/>key_len: 606 <br/>Extra: Using where, Using index | 可以使用覆盖索引，但是需要根据 where 字句进行过滤            |
| sql4 | select * from test where a=? order by c;                    | key: idx_a_b_c <br/>key_len: 606 <br/>Extra: Using index, Using filesort | 可以使用部分索引 a，但无法避免排序，性能差                   |
| sql5 | select * from test order by a, b, c;                        | key: idx_a_b_c <br/>key_len: 606 <br/>Extra: Using index     | 完全使用覆盖索引，同时可以避免排序，性能好                   |
| sql6 | select * from test order by a, b, c desc;                   | key: idx_a_b_c <br/>key_len: 606 <br/>Extra: filesort        | 可以使用覆盖索引，但无法避免排序，这是因为 MySQL InnoDB 创建索引时默认asc升序，索引无法自动倒序排序； |
| sql7 | select * from test where a in (?,?) and b in (?,?) and c=?; | key: idx_a_b_c <br/>key_len: 606 <br/>Extra: Using where, Using index | 可以使用覆盖索引，但是需要根据 where 子句进行过滤（非定值查询） |

在实际设计高性能索引时，可以结合前面的内容，按照如下步骤进行分析。

1. 定位由于索引不合适或缺少索引而导致的慢查询：

   - 通常在业务建库建表时就需要提交业务运行相关的 SQL 给 DBA 审核；
   - 也可以借助Arkcontrol Arkit 来自动化审核。比如，慢查询日志分析，抓出运行慢的 SQL 进行分析；
   - 也可以借助第三方工具例如 Arkcontrol 慢查询分析系统进行慢查询采集和分析；
   - 在分析慢查询时进行参数最差输入，同时，对 SQL 语句的谓词进行过滤因子、基数、选择率和 SQL 查询回表情况的分析。

2. 设计索引:

   - 设计索引的目标是让查询语句运行得足够快，同时让表、索引维护也足够快；
   - 例如，使用业务不相关自增字段为主键，减缓页分裂、页合并等索引维护成本，加速性能；
   - 也可以使用第三方工具进行索引设计，例如 Arkcontrol SQL 优化助手，会给出设计索引的建议。

3. 创建索引策略：

   - 优先为搜索列、排序列、分组列创建索引，必要时加入查询列创建覆盖索引；
   - 计算字段列基数和选择率，选择率越接近于 1 越适合创建索引；
   - 索引选用较小的数据类型（整型优于字符型），字符串可以考虑前缀索引；
   - 不要建立过多索引，优先基于现有索引调整顺序；参与比较的字段类型保持匹配并创建索引。

4. 调优索引：

   - 分析执行计划；

   - 更新统计信息（Analyze Table）；

   - Hint 优化，方便调优（FORCE INDEX、USE INDEX、IGNORE INDEX、STRAIGHT_JOIN）；

   - 检查连接字段数据类型、字符集；

   - 避免使用类型转换；

   - 关注 optimizer_switch，重点关注索引优化特性 ：

     - MRR（Multi-Range Read）

       MRR 优化是为了减少磁盘随机访问，将随机 IO 转化为顺序 IO 的数据访问，其方式是将查询得到辅助索引的键值放到内存中进行排序，通常是按照主键或 RowID 进行排序，当需要回表时直接根据主键或 RowID 排序顺序访问实际的数据文件，加速 SQL 查询。

     - ICP（Index Condition Pushdown）

       ICP 优化也是对索引查询的优化特性，MySQL 根据索引查询到数据后会优先应用 where 条件进行数据过滤，即无法使用索引过滤的 where 子句，其过滤由之前 Server 层的数据过滤下推到了存储引擎层，可以减少上层对记录的检索，提高数据库的整体性能。

### 创建索引的规范

MySQL 创建索引规范如下：

- 命名规范， 内部统一。

- 索引维护的成本，单张表的索引数量不超过 5 个，单个索引中的字段数不超过 5 个。

- 表必需有主键，推荐使⽤ UNSIGNED 自增列作为主键。表不设置主键时 InnoDB 会默认设置隐藏的主键列，不便于表定位数据同时也会增大 MySQL 运维成本（例如主从复制效率严重受损、pt 工具无法使用或正确使用）。

- 唯一键由 3 个以下字段组成，并且在字段都是整形时，可使用唯一键作为主键。其他情况下，建议使用自增列或发号器作主键。 

- 禁止冗余索引、禁止重复索引，索引维护需要成本，新增索引时优先考虑基于现有索引进行 rebuild，例如 (a,b,c)和 (a,b)，后者为冗余索引可以考虑删除。重复索引也是如此，例如索引(a)和索引(a,主键ID) 两者重复，增加运维成本并占用磁盘空间，按需删除冗余索引。

- 联表查询时，JOIN 列的数据类型必须相同，并且要建⽴索引。 

- 不在低基数列上建⽴索引，例如“性别”。 在低基数列上创建的索引查询相比全表扫描不一定有性能优势，特别是当存在回表成本时。

- 选择区分度（选择率）大的列建立索引。组合索引中，区分度（选择率）大的字段放在最前面。 

- 对过长的 Varchar 段建立索引。建议优先考虑前缀索引，或添加 CRC32 或 MD5 伪列并建⽴索引。 

- 合理创建联合索引，(a,b,c) 相当于 (a) 、(a,b) 、(a,b,c)。 

- 合理使用覆盖索引减少IO，避免排序。