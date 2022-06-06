---
title:  组合模式，利用树形结构处理对象之间的复杂关系
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2019-06-04 21:13:00 +0800
categories: [Architecture Design, Design Patterns]
tags: [设计模式, 组合模式 , Design Patterns, Composite]
math: true
mermaid: true
---

单纯从字面上来看，很容易将“组合模式”和“组合关系”搞混。

组合模式最初只是用于解决树形结构的场景，更多的是处理对象组织结构之间的问题。

组合关系则是通过将不同对象封装起来完成一个统一功能。

虽然组合模式并不常用，但是学习它的原理能够获得更多复杂结构上的思考。比如：

- MySQL 的索引设计中就是用了 B+ 树算法的组合模式设计，极大地提升了数据查询时的性能。

组合模式的原理很容易理解，但在代码实现上却是反直觉的。

## 原理

组合模式的定义是：**将对象组合成树形结构以表示整个部分的层次结构。组合模式可以让用户统一对待单个对象和对象的组合。**

两个关键点：

- 一个是用树形结构来分层；
- 另一个是通过统一对待来简化操作。

之所以要使用树形结构，其实就是为了能够在某种层次上进行分类，并且能够通过统一的操作来对待复杂的结构。

比如，常见的公司统计多个维度的人员的工资信息，如果一个一个统计单人的工资信息会比较费时费力，但如果将人员工资信息按照组织结构构建成一棵“树”，那么按照一定的分类标示（比如，部门、岗位），就能快速找到相关人员的工资信息，而不是每次都要查找完所有人的数据后再做筛选。

组合场景的 UML 如下图所示：

![组合场景的 UML 图](https://images.happymaya.cn/assert/design-patterns/compostie-uml.png)

组合模式中包含了三个关键角色：

- 抽象组件：定义需要实现的统一操作；
- 组合节点：代表一个可以包含多个节点的复合对象，意味着在它下面还可以有其他组合节点或叶子节点；
- 叶子节点：代表一个原子对象，意味着在它下面不会有其他节点了。

组合模式最常见的结构就是树形结构，通过上面的三个角色就可以很方便地构建树形结构。

可以结合现实中的例子来理解，比如，一个公司中有总经理，在总经理之下有经理、秘书、副经理等，而在经理之下则有组长、开发人员等，其结构图大致如下：

![组合模式最常见的结构就是树形结构](https://images.happymaya.cn/assert/desgin/compostie-tree.png)

除了树形结构以外，组合模式中还有环形结构和双向结构（如下图），其中，环形结构和数据结构中的单向链表很相似，而双向结构其实就是 Spring 中 Bean 常用的结构。

![组合模式中的环形结构和双向结构](https://images.happymaya.cn/assert/design-patterns/cpmpostie-other.png)

组合模式对应的 UML 的代码实现：

```java
/* 抽象组件 */
public abstract class Component {
    public abstract void operation();
}

/* 叶子节点 */
public class Leaf extends Component{
    @Override
    public void operation() {
        /* 叶子节点的操作放在这里 */
    }
}

/* 组合节点 */
public class Node extends Component{

    /* 存放子节点列表 */
    private List<Component> childrenList;

    @Override
    public void operation() {
        for (Component component : childrenList) {
            component.operation();
        }
    }
}
```

这里有一个经常容易被搞混淆的地方，就是 Node 中 operation() 方法下的 for 循环。很多时候，以为这个 for 循环是一个固定实现的代码，但其实这个理解是错的。

实际上，这里的 for 循环想要表达的意思是，遍历组合节点下的所有子节点时才需要用到循环，而不是这里只能用循环。

从上面的分析中，组合模式封装了如下变化：

- 叶子节点和组合节点之间的区别；
- 真实的数据结构（树、环、网等）；
- 遍历真实结构的算法（比如，广度优先，还是深度优先等）；
- 结构所使用的策略（比如，用来汇总数据，还是寻找最佳/最坏的节点等）。

所以说，**组合模式本质上封装了复杂结构的内在变化，让使用者通过一个统一的整体来使用对象之间的结构。**

## 使用场景

组合模式的使用场景有：

1. **处理一个树形结构，**比如，公司人员组织架构、订单信息等；
2. **跨越多个层次结构聚合数据，**比如，统计文件夹下文件总数；
3. **统一处理一个结构中的多个对象，**比如，遍历文件夹下所有 XML 类型文件内容。

以”订单信息"为例。假设一个新的商品订单系统（如下图），如何计算每个订单的总费用呢？

![一个订单树](https://images.happymaya.cn/assert/design-patterns/compostoe-order.png)

从上面的简图可以看到，一个订单中可能通常会包含各类商品、发票等信息。

在现实中，每个商品都会被放到一个快递盒中，然后小的盒子又可以被放入另一个更大的盒子中，以此类推，整个结构看上去像是一棵“倒过来的树”。

过去，计算价格时的做法是：打开所有快递盒，找到每件商品，然后计算总价。但在程序中，会发现这不是使用简单的 for 循环语句（一次打开盒子的动作）就能完成的，此外，还需要知道：对应的商品类别有哪些，这个订单使用了哪些商品，价格是多少，赠送的商品有哪些，要抵消多少价格……

而组合模式就是专门为需要反复计算或统计的场景而生的。

### 生成树形对象功能的具体实现

1. 定义一个抽象组件 AbstractNode，其中定义节点可以做的操作有：判断是否为根节点、获取节点 id、获取节点关联父节点 id、设置 id、设置父 id、增加、删除节点和获取子节点：

   ```java
   package cn.happymaya.ndp.composite;
   
   public abstract class AbstractNode {
   
       public abstract boolean isRoot();
   
       public abstract int getId();
       public abstract int getParentId();
   
       public abstract void setId(int id);
       public abstract void setParentId(int parentId);
   
       public abstract void add(AbstractNode abstractNode);
       public abstract void remove(AbstractNode abstractNode);
   
       public abstract AbstractNode getChild(int i);
   
   }
   
   ```

   

2. 创建组合节点 Node，继承自 AbstractNode 实现定义的 8 种接口方法，其中 List 对象 children 用于存放子节点列表：

   ```java
   package cn.happymaya.ndp.composite;
   
   import java.util.ArrayList;
   import java.util.List;
   
   public class ComponentNode extends AbstractNode{
   
       private List<AbstractNode> children;
       private int id;
       private int pid;
   
       public ComponentNode() {
           children = new ArrayList<AbstractNode>();
       }
   
       @Override
       public boolean isRoot() {
           return -1 == pid;
       }
   
       @Override
       public int getId() {
           return id;
       }
   
       @Override
       public int getParentId() {
           return pid;
       }
   
       @Override
       public void setId(int id) {
           this.id = id;
       }
   
       @Override
       public void setParentId(int parentId) {
           this.pid = parentId;
       }
   
       @Override
       public void add(AbstractNode abstractNode) {
           abstractNode.setParentId(this.pid + children.size());
           abstractNode.setId(abstractNode.getParentId() + 1);
           children.add(abstractNode);
       }
   
       @Override
       public void remove(AbstractNode abstractNode) {
           children.remove(abstractNode);
       }
   
       @Override
       public AbstractNode getChild(int i) {
           return children.get(i);
       }
   
   }
   ```

   

3. 再创建叶子节点 Leaf，同样继承自 AbstractNode，重写 8 种接口方法，不过，因为叶子节点不能新增和删除节点，所以添加和删除方法不支持，并且获取子节点方法也应该为空：

   ```java
   package cn.happymaya.ndp.composite;
   
   public class ComponentLeaf extends AbstractNode{
   
       private int id;
       private int pid;
   
       @Override
       public boolean isRoot() {
           return false;
       }
   
       @Override
       public int getId() {
           return this.id;
       }
   
       @Override
       public int getParentId() {
           return this.pid;
       }
   
       @Override
       public void setId(int id) {
           this.id = id;
       }
   
       @Override
       public void setParentId(int parentId) {
           this.pid = parentId;
       }
   
       @Override
       public void add(AbstractNode abstractNode) {
           throw new UnsupportedOperationException("这个是叶子节点，无法增加...");
       }
   
       @Override
       public void remove(AbstractNode abstractNode) {
           throw new UnsupportedOperationException("这个是叶子节点，无法删除");
       }
   
       @Override
       public AbstractNode getChild(int i) {
           return null;
       }
   }
   
   ```

   

4. 最后，运行一个单元测试。创建一个根节点，再将一个有两个叶子节点的组合节点添加到根节点上，并打印组合节点的 id 值。

   ```java
   public class Demo {
       public static void main(String[] args) {
           AbstractNode rootNode = new ComponentNode();
           rootNode.setId(0);
           rootNode.setParentId(-1);
   
           AbstractNode node = new ComponentNode();
           node.add(new ComponentLeaf());
           node.add(new ComponentLeaf());
   
           rootNode.add(new ComponentLeaf());
           rootNode.add(new ComponentLeaf());
           rootNode.add(node);
   
           System.out.println(node.getId());
       }
   }
   ```

   

5. 通过单元测试代码的运行，实现了建立一个简单的树形结构。从上面所有的代码实现中，定义根节点比较重要，通过根节点能够不断找到相关的节点，而且使用的操作都是相同的。

总结来说，在面向对象编程中，组合模式能够很好地适用于**解决树形结构的应用场景。**

## 使用组合模式的理由

使用组合模式的三个主要原因：

1. **希望一组对象按照某种层级结构进行管理，**比如，管理文件夹和文件，管理订单下的商品等。树形结构天然有一种层次的划分特性，能够让我们自然地理解多个对象间的结构；
2. **需要按照统一的行为来处理复杂结构中的对象，**比如，创建文件，删除文件，移动文件等。在使用文件时，我们其实并不关心文件夹和文件是如何被组织和存储的，只要我们能够正确操作文件即可，这时组合模式就能够很好地帮助我们组织复杂的结构，同时按照定义的统一行为进行操作；
3. **能够快速扩展对象组合。** 比如，订单下的手机商品节点可以自由挂接不同分类的手机（品牌类的，如华为、苹果），并且还可以按照商品的特性（如，价格、图片、商家、功能等）再自由地挂接新的节点组合，而查找时可以从手机开始查找，不断增加节点类型，直到找到合适的手机商品。

## 优缺点

使用组合模式主要有以下三大优点：

1. **清晰定义分层结构。**组合模式在实现树形结构时，能够非常清楚地定义层次，并且能让使用者忽略层次的差异，以方便对整个层次结构进行控制。
2. **简化使用者使用复杂结构数据的代码。**由于组合模式中的对象只有组合节点和叶子节点两种类型，而节点的使用操作是一样的，比如，查找子节点，查找父节点等，那么对于使用者来说，就能使用一致的行为，而不用不关心当前处理的是单个对象还是整个结构，间接简化了使用者的代码。
3. **快速新增节点，提升组合灵活性。** 在组合模式中，新增节点会很方便，而无须对现有代码进行任何修改，符合“开闭原则”；同时还能够在局部节点上按照相关性再进行自由的组合，大大提升了对象结构的灵活性。

组合模式的缺点。

1. **难以限制节点类型。** 比如，在上面的订单例子中，订单树中除了组合商品类的节点外，实际上只要不约束，就可以组合任意类型的节点，因为它们都来自同一个根节点，都属于节点类型，只不过节点里包含的信息各不相同罢了，所以，在使用组合模式时，通常需要在设计时从逻辑层面上进行一定的约束。
2. **要增加很多运行时的检查，增加了代码复杂度**。 一旦对象类型不能做限制后，就必须通过运行时来进行类型检查，而这个实现过程比较复杂，会增加很多额外的代码耦合性，同时还会增加代码的理解难度。
3. **错误的遍历算法可能会影响系统性能。** 组合模式实现树形结构虽然好用，但是一旦使用了错误的遍历算法，就会在数据量剧增的情况下拖慢系统速度，比如，当使用简单多层 for 循环嵌套来查找全量的数据时，算法的时间复杂度可能是 m 次方 O(n^m)，会造成外部服务阻塞等待，这样很可能会直接导致其他服务因为长时间等待而出现超时错误。因此，使用组合模式时一定要谨慎选择遍历算法。