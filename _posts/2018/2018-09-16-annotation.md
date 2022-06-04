---
title: 注解
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2018-09-16 11:34:00 +0800
categories: [Java]
tags: [annotation, 注解]
---

在 Java中，注解就是给程序添加一些信息，用字符`@`开头，这些信息用于修饰它后面紧挨着的其他代码元素，比如：**类、接口、字段、方法、方法中的参数、构造方法**等。注解可以被**编译器**、**程序运行时**和**其他工具**使用，用于**增强或修改程序行为**等。

## 内置注解

### @Override

Java 内置了一些常用注解：`@Override`、`@Deprecated`、`@SuppressWarnings`。

@Override 修饰一个方法，表示该方法不是当前类首先声明的，而是在某个父类或实现的接口中声明
的，当前类“重写”了该方法，比如：

```java
public class BaseDemo {
    static class Base {
        public void action(){};
    }

    static class child extends Base {
        @Override
        public void action(){
            System.out.println("child action");
        }

        @Override
        public String toString() {
            return "child";
        }
    }
}
```



- Child 的 action 方法重写了父类 Base 中的 action 方法；
- toString() 方法重写了 Object 类中的 toString 方法。

虽然这个注解不写也不会改变这些方法是“重写”的本质，但是它可以减少一些编程错误。如果方法有 @Override
注解，但没有任何父类或实现的接口声明该方法，则编译器会报错，强制修复该问题。

### @Deprecated

@Deprecated 可以修饰的范围很广，包括类、方法、字段、参数等，它表示对应的代码已经过时，不应该使用它。

不过，它是一种警告，而不是强制性的，在 IDE 如 Eclipse 中，会给 Deprecated 元素加一条删除线以示警告。比如，Date中很多方法就过时了：

```java
@Deprecated
public Date(int year,int month,int date)
@Deprecated
public int getYear()
```

在声明元素为 @Deprecated 时，应该用Java文档注释的方式同时说明替代方案，就像Date中的API文档
那样，在调用@Deprecated方法时，应该先考虑其建议的替代方案。

从 Java 9 开始，@Deprecated 多了两个属性：

- since，since 是一个字符串，表示是从哪个版本开始过时的；

- forRemoval，forRemoval 是一个 boolean 值，表示将来是否会删除。比如，Java9 中 Integer的一个构造方法就从版本 9 开始过时了，其代码为：

  ```java
  @Deprecated(since="9")
  public Integer(int value){ 
      this.valuevalue;
  }
  ```

  

### @SuppressWarning

@SuppressWarnings 表示压制 Java 的编译警告，它有一个必填参数，表示压制哪种类型的警告，它也
可以修饰大部分代码元素，在更大范围的修饰也会对内部元素起效，比如，在类上的注解会影响到方法，
在方法上的注解会影响到代码行。对于Date方法的调用，可以这样压制警告：

```java
@SuppressWarnings({"deprecation", "unused"})
public static void main(String[] args) {
    Date date = new Date(2017, 4 ,12);
    int year = date.getYear();
}
```

Java 提供的内置注解比较少，在日常开发中使用的注解基本都是自定义的。不过，一般也不是自己定义的，而是由各种框架和库定义的，主要还是根据它们的文档直接使用。

## 框架和库的注解

### Jackson

Jackson 是一个通用的序列化库，可以使用它提供的注解对序列化进行定制，比如：

- 使用`@JsonIgnore`和`@JsonIgnoreProperties`配置忽略字段；
- 使用`@JsonManagedReference`和`@JsonBackReference`配置互相引用关系；
- 使用`@JsonProperty`和`@JsonFormat`配置字段的名称和格式等。

在 Java 提供注解功能之前，同样的配置功能也是可以实现的，一般通过配置文件实现，但是配置项和
要配置的程序元素不在一个地方，难以管理和维护，使用注解就简单多了，代码和配置放在一起，一目了
然，易于理解和维护。

### 依赖注入容器

Java 开发经常利用某种框架管理对象的生命周期及其依赖关系，这个框架一般称为 DI(Dependency Injection)容器。

DI是指依赖注入，流行的框架有Spring、Guice等。在使用这些框架时，一般不通过 new 创建对象，而是由容器管理对象的创建，对于依赖的服务，也不需要自己管理，而是使用注解表达依赖关系。

这么做的好处有：

- 代码更为简单，也更为灵活

比如容器可以根据配置返回一个动态代理，实现 AOP。

### Servlet 3.0

Servlet 是 Java 为 Web 应用提供的技术框架，早期的 Servlet 只能在 web.xml 中进行配置，而 Servlet3.0 则
开始支持注解，可以使用 @WebServlet 配置一个类为 Servlet，比如：

```java
@WebServlet(urlPatterns ="/async", asyncSupported true)
public class AsyncDemoServlet extends Httpservlet {..}
```

### Web 应用框架

在 Web 开发中，典型的架构都是 MVC(Model-View-Controller)， 典型的需求是配置哪个方法处理哪个 URL 的什么 HTTP 方法，然后将 HTTP 请求参数映射为 Java 方法的参数。

各种框架如Spring MVC、Jersey 等都支持使用注解进行配置，比如，使用Jersey的一个配置示例为：

```java
@Path("/hello")
public class HelloResource {
    @GET
    @Path("test")
    @Produces(MediaType.APPLICATION_JSON)
    public Map<string,object> test(@QueryParam("a")string a) {
        Map<string,object>map new HashMap<>();
        map.put("status","ok");
        return map;
    }
}
```

HelloResource 将处理 Jersey 配置的根路径下`/hello`下的所有请求，而 test 方法将处理`/hello/test`的 GET请求，响应格式为 JSON，自动映射 HTTP 请求参数 a 到方法参数 String a。

### 注解的好处

通过上面的栗子，可以看出，注解有某种神奇的力量，通过简单的声明，就可以达到某种效果。

在某些方面，它类似于序列化，序列化机制中通过简单的 Serializable 接口，Java 就能自动处理很多复杂的事情。它也类似于synchronized关键字，通过它可以自动实现同步访问。

这些都是**声明式编程风格**，在这种风格中，程序都由三个组件组成：

- 声明的关键字和语法本身；
- 系统/框架/库，它们负责解释、执行声明式的语句；
- 应用程序，使用声明式风格写程序。

在编程的世界里，访问数据库的 SQL 语言、编写网页样式的 CSS ,以及正则表达式、函数式编程都是这种风格，这种风格降低了编程的难度，提供了更为高级的语言，使得可以在更高的抽象层次上思考和解决问题，而不是陷于底层的细节实现。

## 创建注解

@Override 的定义如下：

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.SOURCE)
public @interface Override {}
```

定义注解与定义接口有点类似，都用了 interface，不过注解的 interface 前多了@。

另外，它还有两个元注解`@Target`和`@Retention`，这两个注解专门用于定义注解本身。

@Target：表示注解的目标，@Override 的目标是方法（ElementType.METHOD）。ElementType是一个枚举，主要可选值有：

- TYPE:表示类、接口（包括注解），或者枚举声明；
- FIELD:字段，包括枚举常量；
- METHOD:方法；
- PARAMETER:方法中的参数；
- CONSTRUCTOR:构造方法；
- LOCAL_VARIABLE:本地变量；
- MODULE:模块(Java9引入的)。

目标可以有多个，用 {} 表示，比如`@Suppress Warnings`的`@Target`就有多个。Java 7 的定义为：

```java
@Target({TYPE, FIELD, METHOD, PARAMETER, CONSTRUCTOR, LOCAL_VARIABLE})
@Retention(RetentionPolicy.SOURCE)
public @interface SuppressWarnings {
    String[] value();
}
```

如果没有声明`@Target`，默认为适用于所有类型。

`@Retention`表示注解信息保留到什么时候，取值只能有一个，类型为`RetentionPolicy,`它是一个枚
举，有三个取值。

- SOURCE：只在源代码中保留，编译器将代码编译为字节码文件后就会丢掉。
- CLASS：保留到字节码文件中，但Java虚拟机将class文件加载到内存时不一定会在内存中保留。
- RUNTIME：一直保留到运行时。

如果没有声明`@Retention`，则默认为 CLASS。

`@Override`和`@SuppressWarnings`都是给编译器用的，所以`@Retention`都是`Retention-Policy.SOURCE`。

可以为注解定义一些参数，定义的方式是在注解内定义一些方法，比如`@Suppress-Warnings`内定义的
方法 value，返回值类型表示参数的类型，这里是 String[]。使用 @Suppress-Warnings 时必须给 value 提供
值，比如：

```java
@SuppressWarnings(value = {"deprecation", "unused"})
// 可简写为
@SuppressWarnings({"deprecation", "unused"})
```

注解内参数的类型不是什么都可以的，合法的类型有基本类型、String、Class、枚举、注解，以及这
些类型的数组。



参数定义时可以使用 default 指定一个默认值，比如，Guice 中 Inject 注解的定义：

```java
@Target({METHOD,CONSTRUCTOR,FIELD }
@Retention(RUNTIMB)
@Documented
public @interface Inject {
    boolean optional() default false;
}
```

它有一个参数 optional， 默认值为 false。如果类型为 String，默认值可以为""，但不能为null。

如果定义了参数且没有提供默认值，在使用注解时必须提供具体的值，不能为 null。

@Inject 多了一个元注解 @Documented，它表示注解信息包含到生成的文档中。

与接口和类不同，注解不能继承。不过注解有一个与继承有关的元注解@Inherited，例子：

```java
public class InheritDemo {
    
    @Inherited
    @Retention(RetentionPolicy.RUNTIME)
    static @interface Test {
    }
    
    @Test
    static class Base {
    }
    
    static class Child extends Base {
    }

    public static void main(String[] args) {
        System.out.println(Child.class.isAnnotationPresent(Test.class));
    }
}
```



## 查看注解信息

创建了注解，就可以在程序中使用，注解指定的目标，提供需要的参数，但这还是不会影响到程序的运行。

要影响程序，要先能查看这些信息。主要考虑`@Retention`为`RetentionPolicy.RUNTIME`的注解，利用反射机制在运行时进行查看和利用这些信息。

反射相关类中与注解有关的方法有，Class、Field、Method、Constructor 中都有如下方法：

```java
// 获取所有的注解
public Annotation[]getAnnotations()
// 获取所有本元素上直接声明的注解，忍骆inherited来的
public Annotation[]getDeclaredAnnotations()
// 获取指定类型的注解，没有返回nul1
public <A extends Annotation>A getAnnotation(Class<A>annotationclass)
// 判断是否有指定类型的注解
public boolean isAnnotationPresent(Class<?extends Annotation>annotationclass)
```

Annotation 是一个接口，它表示注解，具体定义为：

```java
public interface Annotation {
    boolean equals(Object obj);
    int hashCode();
    String toString();
	// 返回真正的注解类型
    Class<? extends Annotation> annotationType();
}
```

实际上，内部实现时，所有的注解类型都是扩展的 Annotation。
对于 Method 和 Contructor，它们都有方法参数，而参数也可以有注解，所以它们都有如下方法：

```java
public Annotation[][] getParameterAnnotations()
```

返回值是一个二维数组，每个参数对应一个一维数组：

```java
public class MethodAnnotations {

    @Target(ElementType.PARAMETER)
    @Retention(RetentionPolicy.RUNTIME)
    static @interface QueryParam {
        String value();
    }

    @Target(ElementType.PARAMETER)
    @Retention(RetentionPolicy.RUNTIME)
    static @interface DefaultValue {
        String value() default "";
    }

    public void hello(@QueryParam("action") String action) {
        // ...
        System.out.println("hello......");
    }

    public static void main(String[] args) throws NoSuchMethodException {
        Class<?> cls = MethodAnnotations.class;
        Method method = cls.getMethod("hello", new Class[]{String.class, String.class});
        Annotation[][] annts = method.getParameterAnnotations();

        for (int i = 0; i < annts.length; i++) {
            System.out.println("annotations for parameter " + (i + 1));
            Annotation[] anntArr = annts[i];
            for (Annotation annt : anntArr) {
                if (annt instanceof QueryParam) {
                    QueryParam qp = (QueryParam) annt;
                    System.out.println(qp.annotationType().getSimpleName() + ":" + qp.value());
                } else if (annt instanceof DefaultValue) {
                    DefaultValue dv = (DefaultValue) annt;
                    System.out.println(dv.annotationType().getSimpleName() + ":" + dv.value());
                }
            }
        }
    }
}
```



## 注解应用的栗子：定制序列化

## 注解应用的栗子：DI 容器

定义两个注解：

- @SimpleInject
- @SimpleSingleton

### @SimpleInject

1. 定义 @SimpleInject

   ```java
   @Retention(RUNTIME)
   @Target(FIELD)
   public @interface SimpleInject {}
   ```

2. 定于两个服务 ServiceA 和 ServiceB，ServiceA 依赖于 ServiceB，使用 @SimpleInject，如下：

   ```java
   public class ServiceA {
   
       @SimpleInject
       ServiceB b;
   
       public void callB() {
           b.action();
       }
   }
   
   public class ServiceB {
       public void action() {
           System.out.println("I'm B");
       }
   }
   ```

3. DI 容器的类为 SimpleContainer，如下：

   ```java
   public class SimpleContainer {
       // SimpleContainer.getInstance 会创建需要的对象，并配置依赖关系
       // 假定每个类型都有一个 public 默认构造方法，使用它创建对象，然后查看每个字段
       // 如果有 SimpleInject 注解，就根据字段类型获取该类型的实例，并设置字段的值
       public static <T> T getInstance(Class<T> cls) {
           try {
               T obj = cls.newInstance();
               Field[] fields = cls.getDeclaredFields();
               for (Field f : fields) {
                   if (f.isAnnotationPresent(SimpleInject.class)) {
                       if (!f.isAccessible()) {
                           f.setAccessible(true);
                       }
                       Class<?> fieldCls = f.getType();
                       f.set(obj, getInstance(fieldCls));
                   }
               }
               return obj;
           } catch (Exception e) {
               throw new RuntimeException(e);
           }
       }
   }
   ```

### @SimpleSingleton

在上面的代码中，每次获取一个类型的对象，都会新创建一个对象，实际开发中，这不是期望的结果，期望的模式可能是单例，即每个类型只创建一个对象，该对象被所有访问的代码共享，满足这种需求只需要增加一个注解`@SimpleSingleton` 用于修饰类，表示类型是单例，定义如下：

```java
@Retention(RUNTIME)
@Target(TYPE)
public @interface SimpleSingleton {}
```



而后这样修饰 ServiceB：

```java
@SimpleSingleton
public class ServiceB {
    public void action() {
        System.out.println("I'm B");
    }
}
```



SimpleContainer 也需要修改，首先增加一个静态变量，缓存创建过的单例对象：

```java
private static Map<Class<?>, Object> instance = new ConcurrentHashMap();
```

getInstance 的修改如下：

```java
    public static <T> T getInstance(Class<T> cls) {
        try {
            boolean singleton = cls.isAnnotationPresent(SimpleSingleton.class);
            if (!singleton) {
                return createInstance(cls);
            }
            Object obj = instance.get(cls);
            if (obj != null) {
                return (T)obj;
            }
            synchronized (cls) {
                obj = instance.get(cls);
                if (obj == null) {
                    obj = createInstance(cls);
                    instance.put(cls, obj);
                }
            }
            return (T) obj;
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
```

首先检查是否是单例，如果不是，直接调用 createInstance 创建对象。否则，检查缓存，如果有，直接返回，如果没有，则调用 createInstance 创建对象，并放入缓存中。

createInstance 方法如下：

```java
    private static <T> T createInstance(Class<T> cls) throws Exception {
        T obj = cls.newInstance();
        Field[] fields = cls.getDeclaredFields();
        for (Field f: fields) {
            if (f.isAnnotationPresent(SimpleInject.class)) {
                if (!f.isAccessible()){
                    f.setAccessible(true);
                }
                Class<?> fieldCls = f.getType();
                f.set(obj, getInstance(fieldCls));
            }
        }
        return obj;
    }
```



## 总结

- 注解提升了 Java语言 的表达能力；
- 有效实现了应用功能和底层功能的分离，框架/库的程序员可以专注于底层实现，借助反射实现通用功能，提供注解给应用程序员使用，应用程序员可以专注于应用功能，通过简单的声明式注解与框架库进行协作。