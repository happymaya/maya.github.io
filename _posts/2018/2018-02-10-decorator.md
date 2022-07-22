---
title: 装饰器模式：在基础组件上扩展新功能
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2018-02-10 22:18:32 +0800
categories: [Design Pattern, structural]
tags:  [设计模式, Design Pattern, 装饰器模式, Decorator, 对象结构型模式]
math: true
mermaid: true
---

**装饰器模式**看上去和**适配器模式、桥接模式**很相似，都是使用**组合方式**来扩展原有类的，但其实本质上却相差甚远呢。

简单来说，**适配器模式侧重于转换，而装饰模式侧重于动态扩展；桥接模式侧重于横向宽度的扩展，而装饰模式侧重于纵向深度的扩展。**

## 原理

装饰模式的原始定义是：允许动态地向一个现有的对象添加新的功能，同时又不改变其结构，相当于对现有的对象进行了一个包装。

这个定义非常清晰易懂，因为不能直接修改原有对象的功能，只能在外层进行功能的添加，所以**装饰器模式**又叫**包装器模式。**

装饰器模式的 UML 图：

![](https://images.happymaya.cn/assert/design-patterns/decorator-uml.png#crop=0&crop=0&crop=1&crop=1&id=G0GZ2&originHeight=281&originWidth=510&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

装饰模式的四个关键角色：

- **组件：**作为装饰器类包装的目标类；
- **具体组件：**实现组件的基础子类。
- **装饰器：**一个抽象类，其中包含对组件的引用，并且还重写了组件接口方法。
- **具体装饰器：**继承扩展了装饰器，并重写组件接口方法，同时可以添加附加功能。

代码实现如下：

```java
/* 组件 */
public interface IComponent {
    void excute();
}

/* 具体组件 */
public class BaseComponent implements IComponent{
    @Override
    public void excute() {
        // TODO: 2022/6/6 do something
    }
}

/* 装饰器 */
public class BaseDecorator implements IComponent{

    private IComponent wrapper;

    public BaseDecorator(IComponent wrapper) {
        this.wrapper = wrapper;
    }

    @Override
    public void excute() {
        wrapper.excute();
    }
}

/* 具体装饰器 A */
public class DecoratorA extends BaseDecorator{

    public DecoratorA(IComponent wrapper) {
        super(wrapper);
    }

    @Override
    public void excute() {
        super.excute();
    }
}

/* 具体装饰器 B */
public class DecoratorB extends BaseDecorator{

    public DecoratorB(IComponent wrapper) {
        super(wrapper);
    }

    @Override
    public void excute() {
        super.excute();
    }
    
}
```

代码实现比较简单：

- 组件 Component 定义了组件具备的基本功能；
- 具体组件 BaseComponent 是对组件（接口）的一种基础功能的实现；
- 装饰器 BaseDecorator 中包含 Component 的抽象实例对象，作为装饰器装饰的目标对象；
- 具体装饰器 DecoratorA 和 DecoratorB 继承装饰器 BaseDecorator 来进行具体附加功能的沿用与扩展。

所以说，**装饰器模式本质上就是给已有不可修改的类附加新的功能，同时还能很方便地撤销。**

## 使用的场景

装饰模式常用的使用场景有以下几种：

1. **快速动态扩展和撤销一个类的功能场景。** 比如，有的场景下对 API 接口的安全性要求较高，那么就可以使用装饰模式对传输的字符串数据进行压缩或加密。如果安全性要求不高，则可以不使用。
2. **可以通过顺序组合包装的方式来附加扩张功能的场景。** 比如，加解密的装饰器外层可以包装压缩解压缩的装饰器，而压缩解压缩装饰器外层又可以包装特殊字符的筛选过滤的装饰器等。
3. **不支持继承扩展类的场景。** 比如，使用 final 关键字的类，或者系统中存在大量通过继承产生的子类。

在现实中有一个很形象的关于装饰器使用场景的例子，那就是**单反相机镜头前的滤镜。**

不加滤镜其实不会影响拍照，而滤镜实际上就是一个装饰器，滤镜上又可以加滤镜，这样就做到了不改变镜头而又给镜头增加了附加功能。

假设要创建一个文件读写器程序，能够读字符串，又能将读入的字符串写入文件，如下：

1.  首先，创建一个**抽象的文件读取接口 DataLoader**，代码如下： 
```java
/* 抽象的文件读取接口 */
public interface IDataLoader {
    String read();
    void write(String data);
}
```
 

2.  之后，创建一个**具体组件 BaseFileDataLoader**，重写组件 DataLoader 的读写方法： 
```java
/* 具体组件 */
public class BaseFileDataLoader implements IDataLoader {

    private String filePath;

    public BaseFileDataLoader(String filePath) {
        this.filePath = filePath;
    }

    @Override
    public String read() {
        char[] buffer = null;
        File file = new File(filePath);
        try (FileReader reader = new FileReader(file)) {
            buffer = new char[(int) file.length()];
            reader.read(buffer);
        } catch (IOException e) {
            e.printStackTrace();
        }
        return new String(buffer);
    }

    @Override
    public void write(String data) {
        File file = new File(filePath);
        try (OutputStream fos = new FileOutputStream(file)) {
            fos.write(data.getBytes(), 0, data.length());
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    
}
```
 

3.  接下来，再创建一个**装饰器** DataLoaderDecorator，这里要包含一个引用 DataLoader 的对象实例 wrapper，同样是重写 DataLoader 方法，不过这里使用 wrapper 来读写： 
```java
public class DataLoaderDecorator implements IDataLoader{

    private IDataLoader wrapper;

    public DataLoaderDecorator(IDataLoader wrapper) {
        this.wrapper = wrapper;
    }

    @Override
    public String read() {
        return wrapper.read();
    }

    @Override
    public void write(String data) {
        wrapper.write(data);
    }
    
}
```
 

4.  紧接着，创建在读写时**有加解密功能的具体装饰器** EncryptionDataDecorator，它继承了装饰器 DataLoaderDecorator 重写读写方法。
不过，需要注意的是，这里新建了 encode 和 dcode 方法来包装它的父类 DataLoaderDecorator 的读写方法，实现在读文件时进行解密、写文件时进行加密的功能。 
```java
/* 有加解密功能的具体装饰器 */
public class EncryptionDataDecorator extends DataLoaderDecorator{

    public EncryptionDataDecorator(IDataLoader wrapper) {
        super(wrapper);
    }

    @Override
    public String read() {
        return decode(super.read());
    }

    @Override
    public void write(String data) {
        super.write(encode(data));
    }

    private String encode(String data) {
        byte[] result = data.getBytes();
        for (int i = 0; i < result.length; i++) {
            result[i] += (byte) 1;
        }
        return Base64.getEncoder().encodeToString(result);
    }

    private String decode(String data) {
        byte[] result = Base64.getDecoder().decode(data);
        for (int i = 0; i < result.length; i++) {
            result[i] -= (byte) 1;
        }
        return new String(result);
    }
    
}
```
 

5.  再然后，创建一个**压缩和解压的具体装饰器类** CompressionDataDecorator，新建 compress 和 decompress 方法用来包装父类 DataLoaderDecorator 的读写方法，也就是在读取时解压、写入时压缩。 
```java
public class CompressionDataDecorator extends  DataLoaderDecorator{
    public CompressionDataDecorator(IDataLoader wrapper) {
        super(wrapper);
    }

    @Override
    public String read() {
        return decompress(super.read());
    }

    @Override
    public void write(String data) {
        super.write(compress(data));
    }

    private String compress(String stringData) {
        byte[] data = stringData.getBytes();
        try {
            ByteArrayOutputStream bout = new ByteArrayOutputStream(512);
            DeflaterOutputStream dos = new DeflaterOutputStream(bout, new Deflater());
            dos.write(data);
            dos.close();
            bout.close();
            return Base64.getEncoder().encodeToString(bout.toByteArray());
        } catch (IOException e) {
            e.printStackTrace();
            return null;
        }
    }

    private String decompress(String stringData) {
        byte[] data = Base64.getDecoder().decode(stringData);
        try {
            InputStream in = new ByteArrayInputStream(data);
            InflaterInputStream iin = new InflaterInputStream(in);
            ByteArrayOutputStream bout = new ByteArrayOutputStream(512);
            int b;
            while ((b = iin.read()) != -1) {
                bout.write(b);
            }
            in.close();
            iin.close();
            bout.close();
            return new String(bout.toByteArray());
        } catch (IOException e) {
            e.printStackTrace();
            return null;
        }
    }

}
```
 

6.  最后，运行一个单元测试：创建一个具体装饰器，写入的时候先加密再压缩，然后通过普通组件类和具体装饰器类读取进行对比 
```java
public class Demo {
    public static void main(String[] args) {
        String testInfo = "Name, testInfo\nMia, 10000\nMax, 9100";
        DataLoaderDecorator encoded = new CompressionDataDecorator(
                new EncryptionDataDecorator(
                        new BaseFileDataLoader("demo.txt")));
        encoded.write(testInfo);
        IDataLoader plain = new BaseFileDataLoader("demo.txt");
        System.out.println("- 输入 ----------------");
        System.out.println(testInfo);
        System.out.println("- 加密 + 压缩 写入文件--------------");
        System.out.println(plain.read());
        System.out.println("- 解密 + 解压 --------------");
        System.out.println(encoded.read());
    }
}
```
 

总之，装饰模式**适用于一个通用功能需要做扩展而又不想继承原有类的场景，**同时**还适合一些通过顺序排列组合就能完成扩展的场景。**

## 使用装饰模式的理由

主要有以下两个：

**第一个，为了快速动态扩展类功能，降低开发的时间成本。** 比如，一个类 A，有子类 A01、A02，然后 A01 又有子类 A001，以此类推，A0001、A00001……这样的设计会带来一个严重的问题，那就是：当需要扩展 A01 时，所有 A01 的子类和父类都会受到影响。但是，如果这时我们使用装饰器 B01、B02、C01、C02，那么扩展 A01 就会变为 A01B01C01、A01B02C02 这样的组合。这样就能快速地扩展类功能，同时还可以按需来任意组合，极大地节省了开发时间。

**第二个，希望通过继承的方式扩展老旧功能。** 比如，前面说的，当类标识有 final 关键字时，要想复用这个类就只能通过重新复制代码的方式，不过通常这样的类又处于需要对外提供功能的状态，不能轻易修改，而梳理上下文逻辑又费时费力，那么采用装饰模式就是一个很好的选择。因为装饰器是在外层进行扩展，即使功能不合适，也能及时地撤销而不影响原有的功能。所以说，在一些维护系统的升级或重构场景中，使用装饰模式来重构代码，在短期内都能达到快速解耦的效果。

## 优缺点

使用装饰模式主要有以下四个大的优点：

1. **快速扩展对象的功能。** 对于一些独立且无法修改的类来说，当需要在短期内扩展功能时，采用装饰模式能快速有效地扩展功能，同时也不会影响原有的功能；
2. **可以动态增删对象实例的功能。** 比如，在上面文件读写器的例子中，我们可以在创建对象的时候再决定是一起使用压缩装饰器和加密装饰器，还是分开使用，或者只是用基本的读写功能；
3. **可以在统一行为上组合几种行为。** 装饰模式是对某一个接口行为进行的组合扩展，通过包装的方式不断扩展代码的行为，从而实现了更多行为的组合；
4. **满足单一职责原则。** 每一个具体装饰器类只实现一个组件的具体行为，即便附加了新的功能也是围绕着组件的职责而做扩展，保证了职责的单一性。

装饰器模式的缺点：

1. **在调用链中删除某个装饰器时需要修改代码。** 装饰模式的最大弊端在于，当在某个组件上附加了太多装饰器后，想要删除其中的某个装饰器时，就需要修改前后的装饰器的引用位置，这样容易导致上下文中代码都需要修改的情况，大大增加了出错的可能性；
2. **容易导致产生很多装饰对象，增加代码理解难度。** 由于使用了组合方式，并且在调用时使用了链式结构，这样间接增加了很多装饰器对象，而一旦不了解装饰模式的特性，就很容易误解为多个对象的参数调用，增加了代码的理解难度；
3. **增加问题定位和后期代码维护成本。** 虽然装饰模式使用的组合方式比继承更加灵活，但同时也会增加代码的复杂性，在维护代码时会增加问题定位难度，同时调试时也需要逐级排查，比较烦琐，增加了后期代码维护成本。

## 总结

装饰模式就像是送人礼物时的“包装盒”，可以选择各种各样的包装盒，还可以在包装盒里嵌套包装盒。

装饰模式在结构上体现为**链式结构，**通过在外层不断地添加具体装饰器类来对原有的组件类进行扩展，这样在保证原有功能的情况下，还能额外附加新的功能。这也是学习和理解装饰模式的核心所在。

虽然装饰模式的原理和使用都很简单，但是有时链式结构本身会让代码调用链条变得很长，变成了一种对原有组件接口的定制化开发。因此，一般情况下**不建议装饰器超过 10 个，**如果超过还是要考虑重构组件功能。除此之外，**对于没有上下逻辑的装饰器，也要尽量避免使用装饰模式。**
