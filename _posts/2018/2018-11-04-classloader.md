---
title: 类加载机制
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2018-11-04 11:34:00 +0800
categories: [Java, base]
tags: [reflect, 反射]
---


类加载器（ClassLoader）就是加载其他类的类，负责将字节码文件加载到内存，创建 Class 对象。与反射、注解和动态代理一样，在大部分的应用编程中，需要自己实现 ClassLoader。

不过，理解类加载的机制和过程，有助于更好地理解反射、注解和动态代理。比如在反射中的 Class 的静态方法 Class.forName。



ClassLoader，一般是系统提供的，不需要自己实现，不过，通过创建自定义的 ClassLoader，可以实现
一些强大灵活的功能，比如：

1. **热部署。**在不重启 Java 程序的情况下，动态替换类的实现，比如 Java Web 开发中的 JSP 技术就利用
   自定义的 ClassLoader 实现修改 JSP 代码即生效，OSGI(Open Service Gateway Initiative) 框架使用自定义
   ClassLoader 实现动态更新；
2. **应用的模块化和相互隔离。**不同的 ClassLoader 可以加载相同的类但互相隔离、互不影响。Web 应用服务器如 Tomcati 利用这一点在一个程序中管理多个 Web 应用程序，每个 Web 应用使用自己的 ClassLoader，这些 Web 应用互不干扰。OSGI 和J ava9 利用这一点实现了一个动态模块化架构，每个模块有自己的ClassLoader，不同模块可以互不干扰。
3. **从不同地方灵活加载。**系统默认的 ClassLoader 一般从本地的 .class 文件或 jar 文件中加载字节码文件，通过自定义的ClassLoader,我们可以从共享的Web服务器、数据库、缓存服务器等其他地方加载字节码文件。

理解自定义 ClassLoader 有助于理解这些系统程序和框架，如 Tomat、JSP、OSGI，在业务需要的时候，也可以借助自定义 ClassLoader 实现动态灵活的功能。



## 类加载机制和过程

运行 Java 程序，就是执行 java 这个命令，指定包含 main 方法的完整类名，以及一个 classpath，即类路
径。类路径可以有多个：

- 对于直接的 class 文件，路径是 class 文件的根目录；
- 对于jar包，路径是 jar 包的完整名称（包括路径和jar包名）。



Java 运行时，会根据类的完全限定名寻找并加载类，寻找的方式基本就是在系统类和指定的类路径中寻找，如果是 class 文件的根目录，则直接查看是否有对应的子目录及文件；如果是 jar 文件，则首先在内存中解压文件，然后再查看是否有对应的类。

负责加载类的类就是类加载器，它的输入是完全限定的类名，输出是 Class 对象。类加载器不是只有一个，一般程序运行时，都会有三个（适用于Java9 之前，Java9 引入了模块化，基本概念是类似的，但有一些变化）。

1. **启动类加载器（Bootstrap ClassLoader）**：该加载器是 Java 虚拟机实现的一部分，不是 Java 语言实现的，一般是 C++ 实现的，它负责加载 Java 的基础类，主要是 `<JAVA_HOME-/lib/rt.jar`，日常用的 Java 类库比如String、ArrayList 等都位于该包内。

2. **扩展类加载器（Extension ClassLoader）**：这个加载器的实现类是`sun.misc.Launcher$ExtClassLoader`，它负责加载 Java 的一些扩展类，一般是`<JAVA_HOME>Iib/ext`目录中的jar包。

3. **应用程序类加载器(Application ClassLoader)**：这个加载器的实现类是
   `sun.misc.LauncherSAppClassLoader`，它负责加载应用程序的类，包括自己写的和引入的第三方法类库，即所有在类路径中指定的类。

   ![类加载](https://images.happymaya.cn/assert/java/jvm/classloader.png)

这三个类加载器有一定的关系，可认为是父子关系，Application ClassLoader 的父亲是 Extension ClassLoader，Extension 的父亲是 Bootstrap ClassLoader.。

**注意不是父子继承关系，而是父子委派关系。**

子 ClassLoader 有一个变量 parent 指向父 ClassLoader，在子 ClassLoader 加载类时，一般会首先通过父 ClassLoader 加载，具体来说，在加载一个类时，基本过程是：

1. 判断是否已经加载过了，加载过了，直接返回 Class 对象，一个类只会被一个 ClassLoader 加载一
   次；
2. 如果没有被加载，先让父 ClassLoader 去加载，如果加载成功，返回得到的 Class 对象；
3. 在父 ClassLoader 没有加载成功的前提下，自己尝试加载类。

这个过程一般被称为"**双亲委派**"模型，即优先让父 ClassLoader 去加载。为什么要先让父 ClassLoader 去加载呢？因为这样可以**避免 Java 类库被覆盖**的问题。

比如，自己自定义了一个类 java.lang.String，通过双亲委派，java.lang.String 只会被 Bootstrap ClassLoader 加载，避免自定义的 String 覆盖 Java 类库的定义。

不过需要注意的是，“双亲委派” 只是一般模型，但也有一些例外，比如：

1. **自定义的加载顺序：**尽管不被建议，自定义的 ClassLoader 可以不遵从“双亲委派这个约定，不过，即使不遵从，以 java 开头的类也不能被自定义类加载器加载，这是由 **Java 的安全机制**保证的，以避免混乱；
2. **网状加载顺序：**在 **OSGI 框架**和 **Java9 模块化系统**中，类加载器之间的关系是一个网，每个模块有一个类加载器，不同模块之间可能有依赖关系，在一个模块加载一个类时，可能是从自己模块加载，也可能是委派给其他模块的类加载器加载。
3. **父加载器委派给子加载器加载：**典型的例子有 JNDI 服务（Java Naming and Directory Interface）,
   它是 Java 企业级应用中的一项服务。

一个程序运行时，会创建一个 Application ClassLoader，在程序中用到 ClassLoader 的地方，如果没有指定，一般用的都是这个 ClassLoader，所以，这个 ClassLoader 也被称为系统类加载器（System ClassLoader）。



## ClassLoader

类 ClassLoader 是一个抽象类。

Application ClassLoader  的具体实现类是：sun.misc.Launcher$AppClassLoader；

Extension ClassLoader 的具体实现类是：sun.misc.Launcher$ExtClassLoader;

Bootstrap ClassLoader 不是由 Java 实现的，没有对应的类。



每个 Class 对象都有一个方法，可以获取实际加载它的 ClassLoader，方法是：

```java
public ClassLoader getClassLoader() {
    ClassLoader cl = getClassLoader0();
    if (cl == null) {
        return null;
    }
    SecurityManager sm = System.getSecurityManager();
    if (sm != null) {
        ClassLoader.checkClassLoaderPermission(cl, Reflection.getCallerClass());
    }
    return cl;
}
```

有一个方法，可以获取它的父 ClassLoader：

```java
public final ClassLoader getParent()
```

如果 ClassLoader 是 Bootstrap ClassLoader，返回值为 null，比如：

```java
public class ClassLoaderDemo {
    public static void main(String[] args) {
        ClassLoader cl = ClassLoaderDemo.class.getClassLoader();
        while (cl != null) {
            System.out.println(cl.getClass().getName());
            cl = cl.getParent();
        }
        System.out.println(String.class.getClassLoader());
    }
}
```

输出结果为：

```bash
sun.misc.Launcher$AppClassLoader
sun.misc.Launcher$ExtClassLoader
null
```

ClassLoader 有一个静态方法，可以获取默认的系统类加载器：

```java
public static ClassLoader getSystemClassLoader()
```

有一个主要方法，用于加载类：

```
public final ClassLoader getClassLoader()
```

比如：

```java
public class ClassLoaderDemo {
    public static void main(String[] args) {
        try {
            Class<?> cls = cl.loadClass("java.util.ArrayList");
            ClassLoader actualLoader = cls.getClassLoader();
            System.out.println(actualLoader);
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
    }
}
// 输出结果为：null
```

由于委派机制，Class 的 getClassLoader 方法返回的不一定是调用 load-Class 的 ClassLoader，比如，上面代码中，java.util.ArrayList 实际由 BootStrap ClassLoader加载，所以返回值就是 null。



在反射中，Class 的两个静态方法 forName：

```java
public static Class<?> forName(String className);
public static Class<?> forName(String name, boolean initialize, ClassLoader loader)
```

- 第一个方法，使用**系统类加载器加载**，
- 第二个方法，指定 ClassLoader，参数 initialize 表示加载后是否执行类的初始化代码（如static语句块），没有指定默认为true。

ClassLoader 的 loadClass 方法 与 Class 的 forName方法 都可以加载类，它们基本是一样的，不过，ClassLoader 的 loadClass 不会执行类的初始化代码，例子：

```java
public class CLInitDemo {
    public static class Hello {
        static {
            System.out.println("hello");
        }
    }

    public static void main(String[] args) {
        ClassLoader cl = ClassLoader.getSystemClassLoader();
        String className = CLInitDemo.class.getName() + "$Hello";
        try {
            Class<?> cls = cl.loadClass(className);
//            Class<?> cla = Class.forName(className);
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
    }

}
```

使用 ClassLoader 加载静态内部类 Hello，Hello 有一个 static 语句快，输出 ”hello“，运行该程序，类被加载了，但没有任何输出，也就是 static 语句块没有被执行。

如果将 loadClass 的语句换为：

```java
Class<?> cla = Class.forName(className);
```

则 static 语句块会被执行，屏幕将输出 ”hello“。



ClassLoader 的 loadClass 代码是：

```java
public Class<?> loadClass(String name) throws ClassNotFoundException {
    return loadClass(name, false);
}
```

它调用了另一个 loadClass 方法，主要代码为：

```java
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // First, check if the class has already been loaded
            // 首先，检查类是否已经被加载了
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                // 没有加载，先委派父 ClassLoader 或 BootStrap ClassLoader 去加载
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                        // 委派父 ClassLoader，resolve 参数固定为 false
                        c = parent.loadClass(name, false);
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                    // 没有找到，捕获异常，以便尝试自己加载
                }

                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    // 自己加载，findClass 才是当前 ClassLoader 的真正加载方法
                    long t1 = System.nanoTime();
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                // 链接，执行 static 语句块
                resolveClass(c);
            }
            return c;
        }
    }
```

参数 resolve 类似  Class.forName 中的参数 initialize：

- 其默认值为 false，即使通过自定义 ClassLoader 重写 loadClass；
- 设置 resolve 为 true，它调用父 ClassLoader 的时候，传递的也是固定的 false。

findClass 是一个 protected 方法，类 ClassLoader 的默认实现就是抛出 ClassNotFoundException，子类应该重写该方法，实现自己的加载逻辑。

## 类加载的栗子：可配置的策略

很多应用使用面向接口的编程，接口具体的实现类可能有很多，适用于不同的场合，具体使用哪个实现类在配置
文件中配置，通过更改配置，不用改变代码，就可以改变程序的行为，在设计模式中，这是一种**策略模式**。

1. 定义一个服务接口 IService：

   ```java
   public class ServiceB implements IService{
       @Override
       public void action() {
           System.out.println("service B action");
       }
   }
   ```

   

2. 查看配置文件，根据配置的实现类，自己加载，使用反射创建实例对象

   ```java
   public class ConfigurableStrategyDemo {
       public static IService createService() {
           try {
               Properties properties = new Properties();
               String filename = "D:\\happymaya\\notes-java\\dynamic-agent\\src\\main\\resources\\config.properties";
               properties.load(new FileInputStream(filename));
               String className = properties.getProperty("service");
               Class<?> cls = Class.forName(className);
               return (IService)cls.newInstance();
           } catch (Exception e) {
               throw new RuntimeException(e);
           }
       }
   
       public static void main(String[] args) {
           IService service = createService();
           service.action();
       }
   }
   ```

   config.properties 的内容为：

   ```java
   service=cn.happymaya.dynamic.classloader.ServiceB
   ```

   ServiceB 的内容为：

   ```java
   public class ServiceB implements IService{
       @Override
       public void action() {
           System.out.println("service B action");
       }
   }
   ```

3. 输出结果为：

   ```java
   service B action
   ```

   

## 自定义 ClassLoader

Java 类加载机制的强大之处在于，可以创建自定义的 ClassLoader，**自定义 Class-Loader 是 Tomcat**
**实现应用隔离、支持SP、OSGI实现动态模块化的基础**。

自定义 ClassLoader，一般而言，继承类 ClassLoader，而后重写 findClass 就可以了。怎么实现 findClass 呢？使用自己的逻辑寻找 class 文件字节码的字节形式，找到后，使用如下方法转换为 Class 对象：

```java
// name 表示 类名
// b 存放字节码数据的字节数组
// off 有效数据的开始
// len 有效数据的长度
protected final class<?>defineclass(String name,byte[]b,int off,int len)
```

栗子：

```java
public class MyClassLoader extends ClassLoader {
    private static final String BASE_DIR = "";

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        String fileName = name.replaceAll("\\.", "/");
        fileName = BASE_DIR + fileName + ".class";
        try {
            byte[] bytes = BinaryFileUtils.readFileToByteArray(fileName);
            return defineClass(name, bytes, 0, bytes.length);
        } catch (IOException e) {
            e.printStackTrace();
        }
        return super.findClass(name);
    }
}
```

MyClassLoader 从 BASE_DIR 下的路径中加载类，使用自定义实现的 readFileToByteArray 方法读取文件，并转换为 byte 数组。MyClassLoader 没有指定父 Class-Loader，默认是系统类加载器，即
ClassLoader.getSystemClassLoader() 的返回值。

不过，Class-Loader有一个可重写的构造方法，可以指定父 ClassLoader ：

```java
protected ClassLoader(ClassLoader parent) {
    this(checkCreateClassLoader(), parent);
}
```

将 BASE_DIR 加到 classpath ，确实可以，这只是基本用法，实际中，还可以从 **Web 服务器**、**数据库**或**缓存服务器**获取 bytes 数组，这就不是系统类加载器能做到的了。

不过，不把 BASE_DIR 放到 classpath 中，而是使用 MyClassLoader 加载，还有一个很大的好处，那就是
可以创建多个MyClassLoader，对同一个类，每个 MyClassLoader 都可以加载一次，得到同一个类的不同
Class对象，比如：

```java
public static void main(String[] args) throws ClassNotFoundException {
    String className = "cn.happymaya.dynamic.classloader.ServiceB";
    
    MyClassLoader classLoader1 = new MyClassLoader();
    Class<?> class1 = classLoader1.loadClass(className);
    
    MyClassLoader classLoader2 = new MyClassLoader();
    Class<?> class2 = classLoader2.loadClass(className);
    
    if (class2 == class1) {
        System.out.println("different classes");
    }
}
```

classLoader1 和 classLoader2 是两个不同的 ClassLoader。

class1 和class2 对应的类名一样，但它们是不同的对象。

它们的作用有亮点，如下：

1. **实现隔离。**一个复杂的程序，内部可能按模块组织，不同模块可能使用同一个类，但使用的是不同版本，如果使用同一个类加载器，它们是无法共存的，不同模块使用不同的类加载器就可以实现隔离，Tomcat 使用它隔离不同的 Web 应用，OSGI 使用它隔离不同模块。
2. **实现热部署。**使用同一个 ClassLoader，类只会被加载一次，加载后，即使 class 文件已经变了，再次加载，得到的也还是原来的 Class 对象，而使用 MyClassLoader，则可以先创建一个新的 ClassLoader，再用它加载 Class，得到的 Class 对象就是新的，从而实现动态更新。

## 利用自定义的 ClassLoader 热部署

**热部署**，就是在不重启应用的情况下，当类的定义（即字节码文件）修改后，能够替换该 Class 创建的对象。

1. 使用面向接口编程方式，先定义一个接口 HelloService：

   ```java
   public interface IHelloService {
       public void sayHello();
   }
   ```

2. 实现类 HelloServiceImpl：

   ```java
   public class HelloServiceImpl implements IHelloService{
       @Override
       public void sayHello() {
           System.out.println("say Hello....");
       }
   }
   ```

   

3. 演示类是 HotDeployDemo，主要定义了以下静态变量：

   ```java
   public class HotDeployDemo {
       public static final String CLASS_NAME = "cn.happymaya.dynamic.classloader.HelloServiceImpl";
       public static final String FILE_NAME = "data/byte/" + CLASS_NAME.replaceAll("\\.","/") + ".class";
       private static volatile IHelloService helloService;
   }
   ```

   - CLASS_NAME，表示实现类名称；
   - FILE_NAME，表示具体的class文件路径；
   - helloService 是 IHelloService 实例。

4. 当 CLASS._NAME 代表的类字节码改变后，希望重新创建helloService,反映最新的代码，怎么做
   呢？先看用户端获取 IHelloService 的方法：

   ```java
   public static IHelloService getHelloService() {
       if (helloService != null) {
           return helloService;
       }
       synchronized (HotDeployDemo.class) {
           if (helloService == null) {
               helloService = createHelloService();
           }
           return helloService;
       }
   }
   ```

   这只是一个单例模式，createHelloService() 的代码为：

   ```java
   private static IHelloService createHelloService() {
       try {
           MyClassLoader classLoader = new MyClassLoader();
           Class<?> cls = classLoader.loadClass(CLASS_NAME);
           if (cls != null) {
               return (IHelloService) cls.newInstance();
           }
       } catch (Exception e) {
           e.printStackTrace();
       }
       return null;
   }
   ```

   它使用 MyClassLoader 加载类，并利用反射创建实例，它假定实现类有一个 public 无参构造方法。

5. 在调用 HelloService 的方法时，客户端总是先通过 getHelloService 获取实例对象。模拟一个客户端线程，它不停地获取 IHelloService 对象，并调用其方法，然后睡眠 1 秒钟，其代码为：

   ```java
       public static void client() {
           Thread t = new Thread() {
               @Override
               public void run() {
                   try {
                       while (true) {
                           IHelloService helloService = getHelloService();
                           helloService.sayHello();
                           Thread.sleep(1000);
                       }
                   } catch (InterruptedException e) {
                       throw new RuntimeException(e);
                   }
               }
           };
           t.start();
       }
   ```

   

6. 怎么晓得类的 class 文件发送变化，并重新创建 helloService 对象，使用一个单独的线程模拟这个过程：

   ```java
   public static void monitor() {
       Thread t = new Thread() {
           private long lastModified = new File(FILE_NAME).lastModified();
           
           @Override
           public void run() {
               try {
                   while (true) {
                       Thread.sleep(1000);
                       long now = new File(FILE_NAME).lastModified();
                       if (now != lastModified) {
                           lastModified = now;
                           reloadHelloService();
                       }
                   }
               } catch(InterruptedException e){
                   throw new RuntimeException(e);
               }
           }
       };
       t.start();
   }
   ```

   

7. 使用文件的最后修改时间来跟踪文件是否发生了变化，当文件修改后，调用 reloadHelloService() 来重新加载，代码为：

   ```java
   public static void reloadHelloService() {
       helloService = createHelloService();
   jie}
   ```

   

   它就是利用 MyClassLoader 重新创建 HelloService，创建后，赋值给 helloService，这样，下次 getHelloService()  获取到的就是最新的。

8. 在主程序中启动 client 和 monitor 线程，代码为：

   ```java
   public static void main(String[] args) {
       monitor();
       client();
   }
   ```

   在运行过程中，替换 Hellolmpl.class，可以看到行为会变化，目录下准备了两个不同的实现类：HelloImpl_origin.class7 和 HelloImpl_revised.class，在运行过程中替换，会看到输出不一样。

   

   使用 cp 命令修改 Hellolmpl.class，如果其内容与 Hellolmpl_origin.class 一样，输出为"hello"；如果与
   HelloImpl_revised.class 一样，输出为"hello revised"。



## 总结

Java 9 引入了模块的概念。在模块化系统中，类加载的过程有一些变化，扩展类的目录被删除掉了，原来的扩展类加载器没有了，增加了一个平台类加载器(Platform Class Loader)，角色类似于扩展类加载器，它分担了一部分启动类加载器的职责，另外，加载的顺序也有一些变化。之后再说！