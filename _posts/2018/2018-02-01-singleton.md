# 单例模式分析
在 GoF 中，单例模式最早的定义如下：
> 单例模式（Singleton）允许存在一个和仅存在一个给定类的实例。它提供一种机制让任何实体都可以访问该实例。

对应的 UML 图，如下
![](https://cdn.nlark.com/yuque/0/2022/jpeg/12442250/1658142136970-4fbf7e75-454a-4fc4-b22a-d07dc43be023.jpeg)
上图中，单例模式（Singleton）类声明了一个名为 instance 的静态对象和名为 get­Instance() 的静态方法，静态对象用来存储对象自身的属性和方法，静态方法用来返回其所属类的一个相同实例。
以单例模式经典的懒汉式初始化方式为例，其代码实现如下：
```java
package cn.happymaya.ndp.creational.singleton;

/* final 不允许被继承 */
public class LazyManSingleton {

    /* 在定义实例对象的时候直接初始化 */
    private static LazyManSingleton instance = null;

    /* 私有构造方法，不允许外部 new */
    private LazyManSingleton(){}

    private static LazyManSingleton getInstance() {
        if (null == instance) {
            instance = new LazyManSingleton();
        }
        return instance;
    }

}
```
由此，可以得出单例模式包含三个要点：

- 一个单例类只能有一个实例；
- 单例类必须自行创建这个实例；
- 单例类必须保证全局其他对象都能唯一访问到它。

显然，这三个要点是单例模式需要应对的变化，就是：

- 对象实例数量受到限制的事实；
- 对象实例的构造与销毁；
- 需要保证对象实例成为“线程安全”的某种机制。

从上面那段示例代码还可以看出，单例模式的对象职责有两个：

- 保证一个类只有一个实例；
- 为该实例提供一个全局访问节点。

单例类的默认构造函数和静态对象都是内部调用，之所以将默认构造函数设为私有，是为了防止其他对象使用单例类的 new 运算符。然后，提供一个对外的公共方法来获取唯一的对象实例。在我看来，单例模式就类似于全局变量或全局函数的角色，可以使用它来代替全局变量。

常见场景和解决方案
**单例模式更多是在程序一开始进行初始化时使用的。**接下来，我们就来看看有哪些比较常用的场景和解决方案。

常见单例模式应用和使用的解决方案有：

- 饿汉式初始化
- 懒汉式初始化
- 同步信号
- 双重锁定
- 使用 ThreadLocal

看下 ThreadLocal 的方式，比如，下面这个 AppContext 代码示例：
```java
package cn.happymaya.ndp.creational.singleton;

import java.util.HashMap;
import java.util.Map;

public class AppContext {

    private static final ThreadLocal<AppContext> local = new ThreadLocal<>();

    private Map<String, Object> data = new HashMap<>();
    
    
    public Map<String, Object> getData() {
        return getAppContext().data;
    }
    
    // 批量存储数据
    public void setData(Map<String, Object> data) {
        getAppContext().data.putAll(data);
    }
    
    // 存数据
    public void set(String key, String value) {
        getAppContext().data.put(key, value);
    }
    // 取数据
    private void get(String key) {
        getAppContext().data.get(key);
    }
    
    // 初始化的实现方法
    private static AppContext init() {
        AppContext context = new AppContext();
        local.set(context);
        return context;
    }
    // 做延迟初始化
    public static AppContext getAppContext() {
        AppContext context = local.get();
        if (null == context) {
            context = init();
        }
        return context;
    }
    // 删除实例
    public static void  remove() {
        local.remove();
    }
}
```
上面的代码实现，实际上是懒汉式初始化的扩展，只不过用 ThreadLocal 替换静态对象来存储唯一对象实例。之所会选择 ThreadLocal，就是因为 ThreadLocal 相比传统的线程同步机制更有优势。

在传统的同步机制中，通常会通过对象的锁机制来保证同一时间只有一个线程访问单例类。这时该类是多个线程共享的，我们都知道使用同步机制时，什么时候对类进行读写、什么时候锁定和释放对象是有很烦琐要求的，这对于一般的程序员来说，设计和编写难度相对较大。

**而 ThreadLocal 则会为每一个线程提供一个独立的对象副本，**从而解决了多个线程对数据的访问冲突的问题。正因为每一个线程都拥有自己的对象副本，也就省去了线程之间的同步操作。

所以说，**现在绝大多数单例模式的实现基本上都是采用的 ThreadLocal 这一种实现方式。**

# 为什么使用单例模式

1. **系统某些资源有限。**比如，控制某些共享资源（像数据库或文件这些）的访问权限。资源有限会带来访问冲突的问题，如果不限制实例的数量，有限的资源就会很快耗尽，同时造成大量对象处于等待资源中。再比如，同时读写一个超大的 AI 模型文件，或使用外部进程使服务，如果不适用单例模式，随着用户进程数开启越多，系统原有进程资源就会变得越少，这不仅会导致操作系统处理速度变慢，还会造成用户进行自身处理速度缓慢。
1. **需要表示为全局唯一的对象。**比如，系统要求提供一个唯一的序列号生成器。客户调用类的单个实例只允许使用一个公共访问点，除了该公共访问点，不能通过其他途径访问该实例。在一个系统中要求一个类只有一个实例时才应当使用单例模式。反过来，如果一个类可以有几个实例共存，就需要对单例模式进行改进，使之成为多例模式。

# 优缺点
优点：

1. 对有限资源的合理利用，保护有限的资源，防止资源重复竞抢；
1. 更高内聚的代码组件，能提升代码复用性；
1. 具备全局唯一访问点的权限控制，方便按照统一规则管控权限；
1. 从负载均衡角度考虑，可以轻松地将 Singleton 扩展成两个、三个或更多个实例。由于封装了基数问题，所以在适当的时候可以自由更改实例的数量。

缺点：

1. 作为全局变量使用时，引用的对象越多，代码修改影响的范围也就越大；
1. 作为全局变量，在全局变量中使用状态变量，会造成加/解锁的性能损耗；
1. 即便能扩展多实例，耦合性依然很高，因为屏蔽了不同对象之间的调用关系；
1. 不支持有参数的构造函数。

>  Spring 单例 Bean 与单例模式的区别：
> - 单例模式，是指在一个 JVM 进程中有且仅有一个实例；
> - 单例 Bean，是指在一个 Spring 容器中有且仅有一个实例。

