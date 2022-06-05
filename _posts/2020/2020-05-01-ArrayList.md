---
title:  ArrayList
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2020-05-02 23:33:00 +0800
categories: [Java, Collection]
tags:  [List, ArrayList]
math: true
mermaid: true
---

## 基本用法

ArrayList 的主要方法有：
```java
  
public boolean add(E e) // 添加元素到末尾
public void add(int index, E element) // 在指定位置添加元素，index 为 0 表示添加到最前面，index 为 ArrayList 的长度表示添加到最后面

public boolean isEmpty() // 判断是否为空
public int size()   // 获取长度
public E get(int index) // 访问指定位置的元素

public int indexof(Object o) // 查找元素，如果找到，返回索引位置，否则返回 -1
public int lastIndexOf(Object o) // 从后往前找

public boolean contains(Object o) // 是否包含指定元素，依据的是 equals() 方法的返回值

public E remove(int index) // 删除指定位置的元素，返回值为被删对象
public boolean remove(Object o) // 删除指定对象，只删除第一个相同的对象，返回值表示是否删除了元素，如果 o 为 null，则删除值为 null 的元素
public void clear() // 删除所有元素

public E set(int index, E element) // 修改指定位置的元素内容

```

ArrayList 是一个泛型容器，新建 ArrayList 需要实例化泛型参数，比如：
```java
void basicUsage() {
    ArrayList<Integer> intList = new ArrayList<Integer>();
    intList.add(8848);
    intList.add(6379);
    for (int i = 0; i < intList.size(); i++) {
        System.out.println(intList.get(i));
    }
    ArrayList<String> strList = new ArrayList<String>();
    strList.add("清锋");
    strList.add("码呀");
    strList.add(3,"快乐码呀");
    for (int i = 0; i < strList.size(); i++) {
        System.out.println(strList.get(i));
    }
}
```

## 基本原理

ArrayList 内部有一个数组 elementData，一般会有一些预留的空间；有一个整数 size 记录实际的元素个数（基于 Java 8），如下所示：
```java
/**
 * The array buffer into which the elements of the ArrayList are stored.
 * The capacity of the ArrayList is the length of this array buffer. Any
 * empty ArrayList with elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA
 * will be expanded to DEFAULT_CAPACITY when the first element is added.
 */
transient Object[] elementData; // non-private to simplify nested class access(非私有以简化嵌套类访问)

/**
 * The size of the ArrayList (the number of elements it contains).
 *
 * @serial
 */
private int size;

```
ArrayList 中各种 public 方法内部操作的基本都是这个数组和这个整数，elementData 会随着实际元素个数的增多而重新分配，而 size 则始终记录实际的元素个数。

比如 add 和 remove 方法的实现。 

add() 方法的主要代码为：
```java
/**
 * Appends the specified element to the end of this list.
 *
 * @param e element to be appended to this list
 * @return <tt>true</tt> (as specified by {@link Collection#add})
 */
public boolean add(E e) {
    ensureCapacityInternal(size + 1);
    elementData[size++] = e;
    return true;
}
```
可以看到，add() 代码的执行步骤是：
1. 首先调用 ensureCapacityInternal() 方法来确保数组容量是够的，ensureCapacityInternal() 的代码如下：
    ```java
    private void ensureCapacityInternal(int minCapacity) {
        ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
    }
    
    private static int calculateCapacity(Object[] elementData, int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            return Math.max(DEFAULT_CAPACITY, minCapacity);
        }
        return minCapacity;
    }
   
    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // overflow-conscious code(溢出意识代码)
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
    ```
   - 该方法，先判断数组是不是空的，如果是空的，则首次只少要分配的大小为 DEFAULT_CAPACITY（此字段的值为 10）；
   - 接下来调用 ensureExplicitCapacity() 方法，其中的 `modCount` 字段表示内部的修改次数，自然`modCount++`操作表示增加修改次数
   - 如果需要的长度大于当前数组的长度，则调用 grow 方法，主要代码为：
   ```java
       /**
     * Increases the capacity to ensure that it can hold at least the
     * number of elements specified by the minimum capacity argument.
     *
     * @param minCapacity the desired minimum capacity
     */
    private void grow(int minCapacity) {
        // overflow-conscious code(溢出意识代码)
        int oldCapacity = elementData.length;
        // 右移一位相当于除以 2，所以，newCapacity 相当于 oldCapacity 的 1.5 倍
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        // 如果扩展 1.5 倍还是小于 minCapacity ，就扩展为 minCapacity
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
   ```

remove() 方法的代码，如下所示：
```java
    /**
     * Removes the element at the specified position in this list.
     * Shifts any subsequent elements to the left (subtracts one from their
     * indices).
     *
     * @param index the index of the element to be removed
     * @return the element that was removed from the list
     * @throws IndexOutOfBoundsException {@inheritDoc}
     */
    public E remove(int index) {
        rangeCheck(index);

        modCount++;
        E oldValue = elementData(index);

        int numMoved = size - index - 1; // 计算要移动的元素个数
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work（将 size 减 1，同时释放引用以便原对象被垃圾回收）

        return oldValue;
    }
```
由此可见，remove() 方法也增加了 `modCount`，而后计算要移动的元素个数，从 `index` 往后的元素都要往前移动一位，实际调用 `System.arraycopy()` 方法
移动元素；`elementData[--size] = null`，这行代码将 size 减 1，同时将最后一个位置设为 null，设为 null 后不再引用原来对象，如果原来对象也不再被其他对象引用，就可以被垃圾回收。

## 迭代
**迭代（遍历）**是 ArrayList 的常见的操作，迭代(遍历)可以循环打印出 ArrayList 中的每一个元素，有两种方式，代码如下所示：
```java
    void iterate() {
        ArrayList<Integer> intList = new ArrayList<Integer>();
        intList.add(123);
        intList.add(456);
        intList.add(789);
        intList.add(321);
        intList.add(654);
        intList.add(987);
        
        // 方式一 
        for (Integer a: intList) {
            System.out.println(a);
        }
        
        // 方式二
        for (int i = 0; i < intList.size(); i++) {
            System.out.println(intList.get(i));
        }
    }
```
通过上面的代码示例，很明显 foreach 看上去更为简洁，而且它适用于各种容器、更为通用。
编译器会将 foreach 语法转换为类似如下代码：
```java
Iterator var2 = intList.iterator();

while(var2.hasNext()) {
    Integer a = (Integer)var2.next();
    System.out.println(a);
}
```
### 1. 迭代器接口
### 2. ListIterator
除了 iterator(), ArrayList 还提供了两个返回 Iterator 接口的方法
### 3. 迭代的陷阱
关于迭代器，有一种常见的误用：在迭代中间调用容器的删除方法。比如，要删除一个整数 ArrayList 中所有小于 100 的数，在直觉是，代码可以这样写：
```java
   public void remove(ArrayList<Integer> list) {
        for (Integer a: list) {
            if (a <= 100) {
                list.remove(a);
            }
        }
    }
```
在运行时会抛出异常：`java.util.ConcurrentModificationException` ，并发修改异常！

发生这样的情景是因为：迭代器内部会维护一些索引位置相关的数据，要求在迭代过程中，容器不能发生结构性变化（指的是添加、插入和删除元素，只是修改元素内容不算结构性变化），否则这些索引位置就失效了。

想要避免这样的异常，使用迭代器的 remove 方法就可以了，如下所示：
```java
    public static void remove(ArrayList<Integer> list) {
//        for (Integer a: list) {
//            if (a <= 100) {
//                list.remove(a);
//            }
//        }
        Iterator<Integer> it = list.iterator();
        while (it.hasNext()) {
            if (it.next() <= 100) {
                it.remove();
            }
        }
    }
```
### 4. 迭代器的实现原理
### 5. 迭代器的好处

- foreach 语法更为简洁，更为通用，适用于各种容器类
- 迭代器表示的是一种关注点分离的思想，将数据的实际组织方式与数据的迭代遍历相分离，是一种常见的设计模式。需要访问容器元素的代码只需要一个 Iterator 接口的引用，不需要关注数据的实际组织方式，可以使用一致和统一的方式进行访问
- 提供 Iterator 接口的代码了解数据的组织方式，提高高效的实现
- 在 ArrayList 中，size/get(index) 语法与迭代器性能是差不多，但在其他的容器中，则不一定，比如 LinkedList，迭代器性能就要很多
- 从封装的思路来说，迭代器封装了各种数据组织方式的迭代操作，提供了简单和一致的接口

## 实现的接口
## 其他方法
## 特点分析