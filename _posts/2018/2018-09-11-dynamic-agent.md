---
title: 动态代理
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2018-09-11 11:34:00 +0800
categories: [Java]
tags: [dynamic agent, 动态代理]
---

动态代理，可以在运行时动态创建一个类，实现一个或多个接口。可以在不修改原有类的基础上动态为通过该类获取的对象添加方法、修改行为。

这个特性被广泛的应用于各种系统程序、框架和库中，比如 Spring、Hibernate、MyBatis、Guice 等。

动态代理是实现面向切面编程 AOP（Aspect Oriented Programming）的基础。切面的栗子有日志、性能监控、权限检查、数据库事务等，它们在程序的很多地方都会用到。代码也都差不多，但与某个具体的业务逻辑关系也不太密切，如果在每个用到的地方都写，代码会很冗余，也难以维护、AOP 将这些切面与主体业务逻辑分开，代码简单优雅很多。

理解动态代理之前，首先要了解**静态代理**。

## 静态代理

代理是一个比较通用的词，作为一个软件模式，它在《设计模式》中被提出，基本概念和日常生活中的概念是类似的。

代理背后一般至少有一个实际对象，并且代理的外部功能和实际对象一般是一样的。

用户与代理打交道，不直接接触实际对象。虽然外部功能和实际对象一样，但代理的价值如下：

1. 节省成本
2. 执行权限检查，代理检查权限后，再调用实际对象；
3. 屏蔽网络差异和复杂性，代理在本地，而实际对象在其他服务器上，调用本地代理时，本地代理请求其他服务器。

代理模式的代码非常

```java
public class SimpleStaticProxyDemo {

    static interface IService {
        void sayHello();
    }

    static class RealService implements IService {
        @Override
        public void sayHello() {
            System.out.println("Hello World");
        }
    }

    static class TraceProxy implements IService {
        private SimpleStaticProxyDemo.IService realService;

        TraceProxy(IService realService){
            this.realService = realService;
        }

        @Override
        public void sayHello() {
            System.out.println("entering sayHello!");
            this.realService.sayHello();
            System.out.println("leaving sayHello!");
        }
    }

    public static void main(String[] args) {
        IService realService = new RealService();
        IService proxyService = new TraceProxy(realService);
        proxyService.sayHello();
    }
}
```

代理和实际的对象一般都有相同的接口，在这个栗子中，共同的接口就是 IService，实际对象就是 RealService，代理 TraceProxy. TraceProxy 内部有一个 IService 的成员变量，指向实际对象，在构造方法中被初始化，对于方法 sayHello 的调用，它转发给了实际对象，在调用前后输出了一些跟踪调试信息，上面代码的输出为：

```bash 
entering sayHello!
Hello World
leaving sayHello!
```

> 设计模式中的适配器模式和装饰器模式，它们与代理模式有些相似，它们的背后都有一个实际对象，都是通过组合的方式执行该对象。不同之处在于：
>
> - 适配器模式时提供了一个不一样的新接口；
> - 装饰器模式是对原接口起到了“装饰”作用，可能是增加了新接口、修改了原有的行为；
> - 代理模式一般不改变接口

上面的代码，想要达到的目的是在实际对象的方法调用前后加一些跟踪语句。为了在不修改原类的情况下达到这个目的，在代码中创建了一个代理类 TraceProxy，由于它的代码在写程序时是固定的，所以称为静态代理。 

输出跟踪调式信息是一个通用的，可以想象，如果每个类都需要，有不希望修改类定义，就需要为每个类创建代理，实现所有接口，这个工作就太繁琐。如果再有其他的切面要求，整个工作可能又要重来一遍。此时，就需**动态代理。**



## Java SDK 动态代理

### 用法

在静态代理中，代理类是直接定义在代码中的；

在动态代理中，代理类是动态生成的，如下：

```java
public class SimpleJDKDynamicProxyDemo {
    static interface IService {
        void sayHello();
    }

    static class RealService implements IService {
        @Override
        public void sayHello() {
            System.out.println("hello");
        }
    }

    static class SimpleInvocationHandler implements InvocationHandler {

        private Object realObject;

        public SimpleInvocationHandler (Object realObject) {
            this.realObject = realObject;
        }

        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            System.out.println("entering " + method.getName() + "!");
            Object result = method.invoke(realObject, args);
            System.out.println("leaving " + method.getName() + "!");
            return result;
        }
    }

    public static void main(String[] args) {
        IService realService = new RealService();
        IService proxyService = (IService) Proxy.newProxyInstance(
                IService.class.getClassLoader(),
                new Class<?>[]{
                        IService.class
                },
                new SimpleInvocationHandler(realService));
        proxyService.sayHello();
    }
}
```

上面代码中，IService 和 RealService 的定义不变，程序的输出也没什么改变，但代理对象 proxyService 的创建方式变量，它使用 java.lang.reflect 包中的 Proxy 类的静态方法 newProxyInstance 来创建代理对象，这个方法的声明如下：

```bash
 public static Object newProxyInstance(ClassLoader loader, Class<?>[] interfaces,InvocationHandler h)throws IllegalArgumentException{}
```

它有三个参数，具体如下：

- loader ，类加载器；
- interface，代理类要实现的接口列表，是一个数组，**元素的类型只能是接口**，不能是不同的类；
- h 的类型为 InvocationHandler，是一个接口，定义在 java.lang.reflect 包中，只定义了一个 invoke()，对代理接口所有方法的调用都会转给该方法；

newProxyInstance 的返回值类型 Object，可以强制转换为 interfaces 数组中的某个接口类型。这里强制转换为 IService 类型，需要注意的是，它不能强制转换为某个类型，比如 RealService，即使它的实际代理的对象类型为 RealService。

SimpleInvocationHandler 实现了 InvocationHandler ，它的构造方法接受一个参数，realObj 标识被代理的对象，invoke 方法处理所有的接口调用，它有三个参数：

1. Proxy，代理对象本身，需要注意的是：它并不是被代理的对象，这个参数一般用处不大；
2. method ，正在被调用的方法；
3. args ，方法的参数。

SimpleInvocationHandler 的 invoke 实现中，调用了 method 的 invoke 方法，传递了实际对象 realObject 作为参数，达到了调用实际对象对应方法的目的，在调用任何方法前后，输出了跟踪调试语句。

需要注意的是，不能将 Proxy（代理类） 作为参数传递给 method.invoke，比如：

```java
Object result = method.invoke(realObject, args);
```

上面的语句会出现死循环，因为 proxy 表示当前代理对象，这又会调用到 SimpleInvocationHanlder 的 invoke 方法。

### 基本原理

上面代码中，创建 proxyService 的代码可以用下面的代码代替：

```java
        Class<?> proxyClass = Proxy.getProxyClass(IService.class.getClassLoader(), new Class<?>[]{IService.class});
        
        Constructor<?> ctor = null;
        try {
            ctor = proxyClass.getConstructor(new Class<?>[]{InvocationHandler.class});
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        }
        InvocationHandler handler = new SimpleInvocationHandler(realService);
        IService proxyService2 = null;
        try {
            proxyService = (IService)ctor.newInstance(handler);
        } catch (InstantiationException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        }

```

分为三步：

1. 通过 Proxy.getProxyClass 创建代理类定义，类定义会被缓存；
2. 获取代理类的构造方法，构造方法有一个 InvocationHandler 类型的参数；
3. 创建 InvocationHandler 对象，创建代理类对象

Proxy.getProxyClass 需要两个参数，一个是 ClassLoader；另一个是接口数组。它会动态生成一个类，类名 以 $Proxy 开头，后面跟了一个数字。



$Proxy 的父类是 Proxy，它有一个构造方法，接受一个 InvocationHanlder 类型的参数，保存了实例变量 h，h 定义在父类 Proxy 中，它实现了接口 IService ，对于每个方法，比如 sayHello，它调用了 InvocationHanlder 的 invoke 方法，对于 Object 中的方法，如 hashCode、equals 和 toString，$Proxy 同样转发给了 InvocationHanlder。

可以看出，这个类定义本身与被代理的对象没有关系，与 InvocationHandler 的具体实现也没有关系，而主要与接口数组有关，给定这个接口数组，它动态创建每个接口的实现代码，实现就是转发给 InvocationHandler ，与被代理对象的关系以及对它的调用由 InvocationHandler 的实现。

对于 Oracle 的 JVM， $Proxy 的定义可以通过下面的命令看到：

```bash
java -Dsun.misc.ProxyGenerator.saveGeneratedFiles=true cn.happymaya.dynamic.agent.SimpleJDKDynamicProxyDemo
```

以上命令会把动态生成的代理类 $Proxy() 保存到文件 $Proxy.class 中，通过反编译工具比如 [D-GUI](http://jd.benow.ca/) 就可以得到源码。

由此可以，代理类的定义，就是获取构造方法，创建代理对象。

###  动态代理的优点

相比于静态代理，动态代理看起来麻烦很多。但是它的好处如下：**使用动态代理，可以编写通用的代理逻辑，用于各种类型的被代理对象，而不需要为每个被代理的类型都创建一个静态代理类。**

```java
public class GeneralProxyDemo {
    static interface IServiceA {
        void sayHello();
    }

    static class ServiceAImpl implements IServiceA {

        @Override
        public void sayHello() {
            System.out.println("hello");
        }
    }

    static interface IServiceB {
        void fly();
    }

    static class ServiceBImpl implements IServiceB {

        @Override
        public void fly() {
            System.out.println("flying");
        }
    }

    static class SimpleInvocationHandler implements InvocationHandler {

        private Object realObject;

        public SimpleInvocationHandler(Object realObject) {
            this.realObject = realObject;
        }

        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            System.out.println("entering " + realObject.getClass().getSimpleName() + "::" + method.getName());
            Object result = method.invoke(realObject, args);
            System.out.println("leaving " + realObject.getClass().getSimpleName() + "::" + method.getName());
            return result;
        }
    }

    private static <T> T getProxy(Class<T> intf, T realObject){
        return (T)Proxy.newProxyInstance(intf.getClassLoader(),new Class<?>[]{intf}, new SimpleInvocationHandler(realObject));
    }

    public static void main(String[] args) {

        System.err.println("########################################");
        System.err.println("##                                    ##");
        System.err.println("##        General dynamic proxy       ##");
        System.err.println("##                                    ##");
        System.err.println("########################################");

        IServiceA a = new ServiceAImpl();
        IServiceA aProxy = getProxy(IServiceA.class, a);
        aProxy.sayHello();;

        IServiceB b = new ServiceBImpl();
        IServiceB bProxy = getProxy(IServiceB.class, b);
        bProxy.fly();;
    }
}
```

在这个栗子中，有两个接口 IServiceA 和 IServiceB ，它们对应的实现类是 ServiceAImpl 和 ServiceBImpl ，虽然它们的接口和实现不同，但利用动态代理，它们可以调用通用的方法 getProxy 获取代理对象，共享同样的代理逻辑 SimpleInvocationHandler，即在每个方法调用前后输出一条跟踪调试语句：程序输出为：

```bash
entering ServiceAImpl::sayHello
hello
leaving ServiceAImpl::sayHello
entering ServiceBImpl::fly
flying
leaving ServiceBImpl::fly
```

## CGLIB 动态代理

Java SDK 动态代理的缺点是：只能为接口创建代理，返回的代理对象也只能转换到某个接口类型。

如果一个类没有接口，或者希望代理非接口中定义的方法，这就没有办法了。

有一个三方类库 [CGLIB(Code Generation Library)](https://github.com/cglib/cglib)，可以做到上面一点，Spring、Hibernate 等都使用该类库。

```java
public class SimpleCGLIBDemo {
    static class RealService {
        public void sayHello() {
            System.out.println("hello");
        }
    }

    static class SimpleInterceptor implements MethodInterceptor {

        @Override
        public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
            System.out.println("entering " + method.getName() + "!");
            Object result = methodProxy.invokeSuper(o, objects);
            System.out.println("leaving " + method.getName() + "!");
            return result;
        }
    }

    private static <T> T getProxy(Class<T> cls){
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(cls);
        enhancer.setCallback(new SimpleInterceptor());
        return (T) enhancer.create();
    }

    public static void main(String[] args) {
        RealService proxyService = getProxy(RealService.class);
        proxyService.sayHello();
    }
}
```

RealService 表示被代理的类，它没有接口。getProxy() 为一个类生成代理对象，这个代理对象可以安全地转换为被代理类的类型，它使用了 CGLIB 的 Enhancer 类。Enhancer 类的 setSuperClass 设置被代理的类，setCallBack 设置被代理类的 public 非 final 方法被调用时的处理类。Enhancer 支持多种类型，这里使用的类实现了 MethodInterceptor 接口，它与 Java SDK 中的 InvocationHandler 有点类似，方法名变成了 intercept，多了一个 MethodProxy 类型的参数。



与前面的 InvocationHandler 不同，SimpleInterceptor 中没有被代理的对象，它通过 MethodProxy 的 invokeSuper 方法调用被代理类的方法：

```java
Object result = methodProxy.invokeSuper(o, objects);
```

不过，它不能这样调用被代理类的方法：

```java
Object result = methodProxy.invoke(object, args);
```

object 是代理对象，调用这个方法还会调用到 SimpleInterceptor 的 intercept 方法，造成死循环。

在 main 方法中，也没有创建被代理的对象，创建的对象直接就是代理对象。

CGLIB 的实现机制与 Java SDK 不同，是通过继承实现的。也是动态创建了一个类，但这个类的父类是被代理的类，代理类重写了父类的所有 public 非 final 方法，改为调用 Callback 中的相关方法。上面代码中，调用 SimpleInterceptor 的 intercept 方法。

## Java SDK 代理与 CGLIB 代理的比较

**Java SDK 代理面向的是一组接口。**它为这些接口动态创建了一个实现类。接口的具体实现逻辑是通过自定义的 InvocationHandler 实现的，这个实现是自定义的，也就是说，**其背后都不一定有真正被代理的对象，也可能有多个实际对象，根据情况动态选择。CGLIB 代理面向的是一个具体的类，**它动态创建了一个新类，继承了该类，重写了其方法。



从代理的角度看，**Java SDK 代理的是对象，**需要先有一个实际对象，自定义的 InvocationHandler 引用该对象，然后创建一个代理类和代理对象，客户端访问的是代理对象，代理对象最后再调用实际对象的方法；**CGLIB 代理的是类**，创建的对象只有一个。



如果目的都是为一个类的方法增强功能，Java SDK 要求该类要求必须有接口，且只能处理接口中的方法，CGLIB 没有这个限制。

## 动态代理的应用：AOP

利用 CGLIB 动态代理，可以实现一个极简单的 AOP 框架，从而学习了解 AOP 的基本思想和技术。

### 用法

1. 添加一个新的注解`@Aspect`，其定义为：

   ```java
   @Retention(RUNTIME)
   @Target(TYPE)
   public @interface Aspect {
       Class<?>[] value();
   }
   
   ```

2. 约定，切面类可以声明三个方法 before/after/exception，在主体类的方法 **调用前/调用后/出现异常时** 分别调用这三个方法，这三个方法的声明符合如下参数：

   ```java
   public static void before(Object object, Method method, Object[] args) {}
   
   public static void after(Object object, Method method, Object[] args, Object result) {}
   
   public static void exception(Object object, Method method, Object[] args, Throwable e) {}
   ```

3. `@Aspect`主要用于注解切面类，它有一个参数，可以指定要增强的类，比如：

   ```java
   @Aspect({ServiceA.class,ServiceB.class})
   public class ServiceLogAspect {
   
       public static void before(Object object, Method method, Object[] args) {
           System.out.println(
                   "entering "
                           + method.getDeclaringClass().getSimpleName()
                           + "::"
                           + method.getName()
                           + ", args: "
                           + Arrays.toString(args)
           );
       }
   
       public static void after(Object object, Method method, Object[] args, Object result) {
           System.out.println(
                   "leaving "
                           + method.getDeclaringClass().getSimpleName()
                           + "::"
                           + method.getName()
                           + ", result: "
                           + result
           );
       }
   }
   
   ```

   ServiceLogAspect 就是一个切面，它负责 ServiceA 和 ServiceB 的日志切面，即为这两个增加日志功能。

   目的是在类 ServiceA 和 ServiceB 所有方法的执行前后增加一些日志，而 ExceptionAspect 的目的是在类 ServiceB 的方法执行出现异常时收到通知并输出一些信息。

   **它们都没有修改类 ServiceA 和 ServiceB 本身，本身做的事是比较通用的，与 ServiceA 和 ServiceB 的具体逻辑关系也不密切，但又想改变 ServiceA/ServiceB 的行为，这就是 AOP 的思维。**

4. 异常切面，负责类的异常切面

   ```java
   @Aspect({ServiceB.class})
   public class ExceptionAspect {
       public static void exception(Object object, Method method, Object[] args, Throwable e) {
           System.err.println("exception when calling: " + method.getName() + "," + Arrays.toString(args));
       }
   }
   ```

   object、method 和 args 与 CGLIB MethodInterceptor 中的 invoke 参数一样，after 中的 result 表示方法执行的结果，exception 中的 e 表示发生的异常类型。 

   ExceptionAspect 只实现 exception 方法，在异常发生的时候，会输出一些信息。

5. 只是声明一个切面类是不起作用的，还需要和 DI 容器结合起来，实现一个新的容器 CGLibContaier ，有一个方法如下：

   通过该方法获取 ServiceA 和 ServiceB ，它们的行为就会被改变。

   通过 CGLibContaier 获取 ServiceA，会自动应用 ServiceLogAspect，比如：

   ```java
   ServiceA a = CGLibContaier.getInstance(ServiceA.class);
   ```

   输出为：

   ```bash
   entering ServiceA::callB, args: {}
   entering ServiceB::action, args: {}
   I'm B
   leaving ServiceB::action, result: null
   leaving ServiceA::callB, result:null
   ```

   







