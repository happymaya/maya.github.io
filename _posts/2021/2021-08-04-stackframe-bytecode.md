---
title: 从栈帧看字节码在 JVM 中进行流转
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2021-08-04 23:23:00 +0800
categories: [Java, JVM]
tags: [JVM]
math: true
mermaid: true
---

- 怎么查看字节码文件？
- 字节码文件长什么样子？
- 对象初始化之后，具体的字节码又是怎么执行的？

## 工具

### javap

javap 是 JDK 自带的反解析工具。它的作用是**将 .class 字节码文件解析成可读的文件格式。**

在使用 javap 时我一般会添加 -v 参数，尽量多打印一些信息。同时，我也会使用 -p 参数，打印一些私有的字段和方法。使用起来大概是这样：

```java
javap -p -v HelloWord
```

在 StackOverflow 上有一个非常有意思的问题：在某个类中增加一行注释之后，为什么两次生成的 .class 文件，它们的 MD5 是不一样的？

这是因为在 javac 中可以指定一些额外的内容输出到字节码。经常用的有：

- `javac -g:lines` ，强制生成 LineNumberTable。
- `javac -g:vars ` ，强制生成 LocalVariableTable。
- `javac -g` ，生成所有的 debug 信息。





### jclasslib

jclasslib 是一个图形化的工具，能够更加直观的查看字节码中的内容。它还分门别类的对类中的各个部分进行了整理，非常的人性化。同时，它还提供了 Idea 的插件，可以从 plugins 中搜索到它。

jclasslib 的下载地址：https://github.com/ingokegel/jclasslib



## 类加载和对象创建的时机

首先，写一个最简单的 Java 程序 A.java。它有一个公共方法 test，还有一个静态成员变量和动态成员变量：

```java
package cn.happymaya.four;
class B {
    private int a = 1234;

    private long c = 1111;

    public long test(long num) {
        long ret = this.a + num + c;
        return ret;
    }
}
public class A {
    private B b = new B();

    public static void main(String[] args) {
        A a = new A();
        long num = 4321;
        long ret = a.b.test(num);
        System.out.println(ret);
    }
}

```

类的初始化发生在类加载阶段，除了常用的 new ，还有下面一些方式：

- 使用 Class 的 newInstance 方法；
- 使用 Constructor 类的 newInstance 方法。
- 反序列化；
- 使用 Object 的 clone 方法。

其中，最后两种方式没有调用到构造函数！

当虚拟机遇到一条 new 指令时：

1. 首先检查这个指令的参数能否在常量池中定位一个符号引用；
2. 然后检查这个符号引用的类字节码是否被加载、解析和初始化。如果没有，将执行对应的类加载过程。

拿上面代码来说，执行 A 代码，在调用 `private B b = new B()` 时，就会触发 B 类的加载：

![](D:\happymaya\images.github.io\assert\java\jvm\jvm-04-01.jpg)

如上图，A 和 B 会被加载到元空间的方法区，进入到 main 方法后，就会交给执行引擎（Execution engine）执行。

这个执行过程是在栈上完成的，其中有几个重要的区域：包括虚拟机栈（Virtual Machine Stacks）、程序计数器（Program counter register， 针对于字节码指令的，主要线程切换的时候使用）等。



## 查看字节码

### 命令行查看字节码

1. 使用命令`javac -g:lines -g:vars A.java`编译源代码 A.java。在 idea 里面，直接将参数追加在 VM options 里面，这会强制生成 LineNumberTable 和 LocalVariableTable；

2. 使用 javap 命令查看 A 和 B 的字节码，

   ```java
   javap -p -v A
   javap -p -v B
   ```

   这个命令，会输出：行号、本地变量表信息、反编译汇编代码、当前类用到的常量池等信息

   javap 中的如下字样：
   
   **<1>** 
   
   ```java
   1: invokespecial #1    // Method java/lang/Object."<init>":()V
   ```
   
   可以看到：对象的初始化，首先调用了 Object 类的初始化方法。注意这里是<init>，而不是 <cinit>。
   
   **<2>** 
   
   ```java
    #7 = Fieldref           #8.#9          // cn/happymaya/four/B.a:I  
   ```
   
   它其实直接拼接了 #8 和 #9 的内容：
   
   ```java
   #8 = Class              #10            // cn/happymaya/four/B         
   #9 = NameAndType        #11:#12        // a:I                         
   ...                           
   #11 = Utf8              a                                             
   #12 = Utf8              I 
   ```
   
   **<3>**
   
   `:I` 这样特殊字符，也是有意义的，大体包括：
   
   - B 基本类型 byte
   - C 基本类型 char
   - D 基本类型 double
   - F 基本类型 float
   - I 基本类型 int
   - J 基本类型 long
   - S 基本类型 short
   - Z 基本类型 boolean
   - V 特殊类型 void
   - L 对象类型，以分号结尾，如 `Ljava/lang/Object;`
   - `[Ljava/lang/String;` 数组类型，每一位使用一个前置的"["字符来描述
   
   在注意下 code 区域，有非常多的二进制指令。跟汇编语言有一定的相似性。但这些二进制指令，操作系统不认识，它们只是提供给 JVM 运行的原材料。
   
    

### 可视化查看字节码

使用更加直观的工具 jclasslib，来查看字节码中的具体内容了。以 B.class 文件为例，来查看它的内容：

1. 首先，看到 Constant Pool（常量池），这些内容，存放于 Metaspace 区域，属于非堆。

   ![](D:\happymaya\images.github.io\assert\java\jvm\jvm-04-02.jpg)

   常量池包含：.class 文件常量池、运行时常量池、String 常量池等部分。大多是一些静态内容。

2. 接下来，可以看到两个默认的 `<init>` 和 `<cinit>`  方法。以下截图是 test 方法的 code 区域，比命令行版的更加直观。

   ![](D:\happymaya\images.github.io\assert\java\jvm\jvm-04-03.jpg)

3. 继续往下看，看到了 **LocalVariableTable** 的三个变量。其中，slot 0 指向的是 this 关键字。该属性的作用是**描述帧栈中局部变量与源码中定义的变量之间的关系**。**如果没有这些信息，那么在 IDE 中引用这个方法时，将无法获取到方法名，取而代之的则是 arg0 这样的变量名。**

   ![](D:\happymaya\images.github.io\assert\java\jvm\jvm-04-04.jpg)

   **本地变量表的 slot 是可以复用的。注意一个有意思的地方，index 的最大值为 3，证明了本地变量表同时最多能够存放 4 个变量**（示例代码的最大值。如果创建了上千个变量，最大值达到1k都有可能）。

   

   另外，观察到还有 **LineNumberTable** 等选项。该属性的作用是**描述源码行号与字节码行号（字节码偏移量，偏移量是字节单位，不是指令的个数，包含操作数等额外数量）之间的对应关系**，有了这些信息，在 debug 时，就能够获取到发生异常的源代码行号。
   
   

## test 函数执行过程

### Code 区域

test 函数同时使用了成员变量 a、静态变量 C，以及输入参数 num。此时说的函数执行，内存其实就是在虚拟机栈上分配的。下面这些内容，就是 test 方法的字节码。

```shell
  public long test(long);
    descriptor: (J)J
    flags: ACC_PUBLIC
    Code:
      stack=4, locals=5, args_size=2
         0: aload_0
         1: getfield      #7                  // Field a:I
         4: i2l
         5: lload_1
         6: ladd
         7: aload_0
         8: getfield      #15                 // Field c:J
        11: ladd
        12: lstore_3
        13: lload_3
        14: lreturn
      LineNumberTable:
        line 8: 0
        line 9: 13
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      15     0  this   Lcn/happymaya/four/B;
            0      15     1   num   J
           13       2     3   ret   J
```

比较重要的 3 三个数值：

1.  stack  ，它此时的数值为 4，表明了 test 方法的最大操作数栈深度为 4。JVM 运行时，会根据这个数值，来分配栈帧中操作栈的深度。
2. locals 变量存储了局部变量的存储空间。它的单位是 Slot（槽），可以被重用。其中存放的内容，包括：
   - this
   - 方法参数
   - 异常处理器的参数
   - 方法体中定义的局部变量
3. args_size 。它指的是方法的参数个数，因为每个方法都有一个隐藏参数 this，所以这里的数字是 2。

## 字节码执行过程

main 线程会拥有两个主要的运行时区域：**Java 虚拟机栈**和**程序计数器**。

其中，虚拟机栈中的每一项内容叫作栈帧，栈帧中包含四项内容：**局部变量报表**、**操作数栈**、**动态链接**和完成出口。

字节码指令，就是靠操作这些数据结构运行的！！！

![](D:\happymaya\images.github.io\assert\java\jvm\jvm-04-05.jpg)

### 0:aload_0

把第 1 个引用型局部变量推到操作数栈，这里的意思是把 this 装载到了操作数栈中。

对于 static 方法，aload_0 表示对方法的第一个参数的操作。

![](D:\happymaya\images.github.io\assert\java\jvm\jvm-04-06.jpg)

### 1: getfield      #2                  // Field a:I

将栈顶指定的对象的第 2 个实例域（Field）的值，压入栈顶。#7 就是指的是成员变量 a。

```
#7 = Fieldref           #8.#9          // cn/happymaya/four/B.a:I
#8 = Class              #10            // cn/happymaya/four/B
#9 = NameAndType        #11:#12        // a:I
```

![](D:\happymaya\images.github.io\assert\java\jvm\jvm-04-07.jpg)

### i2l

将栈顶 int 类型的数据转化为 long 类型，这涉及隐式类型转换了，图中的信息没有变动。

### iload_1

将第一个局部变量入栈。也就是我们的参数 num。这里的 l 表示 long，同样用于局部变量装载。你会看到这个位置的局部变量，一开始就已经有值了。

![](D:\happymaya\images.github.io\assert\java\jvm\jvm-04-08.jpg)

### ladd

把栈顶两个 long 型数值出栈后相加，并将结果入栈。

![](D:\happymaya\images.github.io\assert\java\jvm\jvm-04-09.jpg)

### getsatic #3

根据偏移获取静态属性的值，并把这个值 push 到操作数栈上。

![](D:\happymaya\images.github.io\assert\java\jvm\jvm-04-10.jpg)

### ladd

再次执行 ladd。

![](D:\happymaya\images.github.io\assert\java\jvm\jvm-04-11.jpg)



### lstore_3

把栈顶 long 型数值存入第 4 个局部变量。

上面的图，slot 为 4，索引为 3 的就是 ret 变量。

![](D:\happymaya\images.github.io\assert\java\jvm\jvm-04-12.jpg)

### lload_3

正好与上面相反。上面是变量存入，现在要做的，就是把这个变量 ret，压入虚拟机栈中。

![](D:\happymaya\images.github.io\assert\java\jvm\jvm-04-13.jpg)

### Ireturn

从当前方法返回 long。到此为止，函数就完成了相加动作，执行成功了。JVM 提供非常丰富的字节码指令。

详细的字节码指令列表，参考以下网址：https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html

### 注意：

上面的第 8 步，首先把变量存放到了变量报表，然后又拿出这个值，把它入栈。为什么会有这种多此一举的操作？原因就在于我们定义了 ret 变量。JVM 不知道后面还会不会用到这个变量，所以只好傻瓜式的顺序执行。

为了看到这些差异。可以把程序稍微改动一下，直接返回这个值：

```java
public long test(long num) {
    return this.a + num + C;
}
```

再次看下，对应的字节码指令简单了很多！！！

```
0: aload_0
1: getfield		#2	// Field a:I
4: i2l
5: lload_1
6: ladd
7: getstatic	#3 	// Field C:J
10: ladd
11: lreturn
```

尽管如此，以后编写程序时，是不是要尽量少的定义成员变量？

这是没有必要的。栈的操作复杂度是 O(1)，对程序性能几乎没有影响。平常的代码编写，还是以可读性作为首要任务。

## 总结

字节码的执行流程这样理解：

- 字节码在Java虚拟机栈中被执行，每一项内容都可以看作是一个栈帧；
- 栈帧的结构包括局部变量表、操作数栈、链接、返回地址。这时候就很明了了，栈帧的执行流程就是字节码的执行流程了；
- 类中变量会被解析到局部变量表，然后对操作数栈进行入栈出栈的操作，在此期间有可能引用到动态或静态链接，最后把计算结果的引用地址返回。