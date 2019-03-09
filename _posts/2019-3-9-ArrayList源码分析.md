---
layout:     post
title:      ArrayList源码学习
subtitle:   
date:       2019-1-21
author:     BY xukexiang
header-img: img/charlotte/005.jpg
catalog: true
tags:
    - Typora
---

## ArrayList的源码学习

ArrayList在我们开发中经常用到，而且基本面试必问，所以它还是挺重要的。ArrayList的源码我感觉还是比较容易看的懂的，感觉跟HashMap那些源码比起来是
比较容易的。

### ArrayList源码的分析思路
***
1.ArrayList 概述

2.ArrayList 的构造函数，也就是我们创建一个 ArrayList 的方法。

3.ArrayList 的添加元素的方法， 以及 ArrayList 的扩容机制

4.ArrayList 的删除元素的常用方法

5.ArrayList 的 改查常用方法

6.ArrayList 的 toArray 方法

7.ArrayList 的遍历方法，以及常见的错误操作即产生错误操作的原因

### ArrayList概述
***
#### ArrayList的基本特点
1.ArrayList 底层是一个动态扩容的数组结构

2.允许存放（不止一个） null 元素

3.允许存放重复数据，存储顺序按照元素的添加顺序

4.ArrayList 并不是一个线程安全的集合。如果集合的增删操作需要保证线程的安全性，可以考虑使用 CopyOnWriteArrayList 或者使用 collections.synchronizedList(List l)函数返回一个线程安全的ArrayList类.

#### ArrayList的继承关系
```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```

从 ArrayList 的继承关系来看，ArrayList 继承自 AbstractList，实现了List<E>, RandomAccess, Cloneable, java.io.Serializable 接口。

- 其中 AbstractList和 List<E> 是规定了 ArrayList 作为一个集合框架必须具备的一些属性和方法，ArrayList 本身覆写了基类和接口的大部分方法，这就包含我们要分析的增删改查操作。


- ArrayList 实现 RandomAccess 接口标识着其支持随机快速访问，查看源码可以知道RandomAccess 其实只是一个标识，标识某个类拥有随机快速访问的能力，针对 ArrayList 而言通过 get(index)去访问元素可以达到 O(1) 的时间复杂度。有些集合类不拥有这种随机快速访问的能力，比如 LinkedList 就没有实现这个接口。


- ArrayList 实现 Cloneable 接口标识着他可以被克隆/复制，其内部实现了 clone 方法供使用者调用来对 ArrayList 进行克隆，但其实现只通过 Arrays.copyOf 完成了对 ArrayList 进行「浅复制」，也就是你改变 ArrayList clone后的集合中的元素，源集合中的元素也会改变，对于深浅复制我以后会单独整理一篇文章来讲述这里不再过多的说。


- 对于 java.io.Serializable 标识着集合可被被序列化。

我们发现了一些有趣的事情，除了List<E> 以外，ArrayList 实现的接口都是标识接口，标识着这个类具有怎样的特点，看起来更像是一个属性。

### ArrayList 的构造方法
***
在说构造方法之前我们要先看下与构造参数有关的几个全局变量：

```java
/**
 * ArrayList 默认的数组容量
 */
 private static final int DEFAULT_CAPACITY = 10;

/**
 * 这是一个共享的空的数组实例，当使用 ArrayList(0) 或者 ArrayList(Collection<? extends E> c) 
 * 并且 c.size() = 0 的时候讲 elementData 数组讲指向这个实例对象。
 */
 private static final Object[] EMPTY_ELEMENTDATA = {};

/**
 * 另一个共享空数组实例，再第一次 add 元素的时候将使用它来判断数组大小是否设置为 DEFAULT_CAPACITY
 */
 private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

/**
 * 真正装载集合元素的底层数组 
 * 至于 transient 关键字这里简单说一句，被它修饰的成员变量无法被 Serializable 序列化 
 * 有兴趣的可以去网上查相关资料
 */
transient Object[] elementData; // non-private to simplify nested class access
```

对于上述几个成员变量，我们只是在注释中简单的说明，对于他们具体有什么作用，在下边分析构造方法和扩容机制的时候将会更详细的讲解。

ArrayList 一共三种构造方式，我们先从无参的构造方法来开始：

#### 无参构造方法
```java
/**
 * 构造一个初始容量为10的空列表。
 */
public ArrayList() {
   this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
```

这是我们经常使用的一个构造方法，其内部实现只是将 elementData 指向了我们刚才讲得 DEFAULTCAPACITY_EMPTY_ELEMENTDATA 这个空数组，这个空数组的容量是 0， 但是源码注释却说这是构造一个初始容量为10的空列表。这是为什么？其实在集合调用 add 方法添加元素的时候将会调用 ensureCapacityInternal 方法，在这个方法内部判断了：

```java
if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
       minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
}
```

可见，如果采用无参数构造方法的时候第一次添加元素肯定走进 if 判断中 minCapacity 将被赋值为 10，所以「构造一个初始容量为10的空列表。」也就是这个意思。


#### 指定初始容量的构造方法
```java
/**
 * 构造一个具有指定初始容量的空列表。
 * @param  初始容量 
 * @throws 如果参数小于 0 将会抛出 IllegalArgumentException  参数不合法异常
 */
public ArrayList(int initialCapacity) {
   if (initialCapacity > 0) {
       this.elementData = new Object[initialCapacity];
   } else if (initialCapacity == 0) {
       this.elementData = EMPTY_ELEMENTDATA;
   } else {
       throw new IllegalArgumentException("Illegal Capacity: "+
                                          initialCapacity);
   }
}
```

如果我们预先知道一个集合元素的容纳的个数的时候推荐使用这个构造方法，比如我们有个一 FragmentPagerAdapter 一共需要装 15 个 Fragment ，那么我们就可以在构造集合的时候生成一个初始容量为 15 的一个集合。有人会认为 ArrayList 自身具有动态扩容的机制，无需这么麻烦，下面我们讲解扩容机制的时候我们就会发现，每次扩容是需要有一定的内存开销的，而这个开销在预先知道容量的时候是可以避免的。
源代码中指定初始容量的构造方法实现，判断了如果 我们指定容量大于 0 ，将会直接 new 一个数组，赋值给 elementData 引用作为集合真正的存储数组，而指定容量等于 0 的时候讲使用成员变量 EMPTY_ELEMENTDATA  作为暂时的存储数组，这是 EMPTY_ELEMENTDATA 这个空数组的一个用处（不必太过于纠结 EMPTY_ELEMENTDATA 的作用，其实它的在源码中出现的频率并不高）。

#### 使用另个一个集合 Collection 的构造方法
```java

/**
 * 构造一个包含指定集合元素的列表，元素的顺序由集合的迭代器返回。
 *
 * @param 源集合，其元素将被放置到这个集合中。 
 * @如果参数为 null，将会抛出 NullPointerException 空指针异常
 */
 public ArrayList(Collection<? extends E> c) {
    elementData = c.toArray();
    if ((size = elementData.length) != 0) {
        // c.toArray 可能(错误地)不返回 Object[]类型的数组 参见 jdk 的 bug 列表(6260652)
        if (elementData.getClass() != Object[].class)
            elementData = Arrays.copyOf(elementData, size, Object[].class);
    } else {
        // 如果集合大小为空将赋值为 EMPTY_ELEMENTDATA 等同于 new ArrayList(0);
        this.elementData = EMPTY_ELEMENTDATA;
    }
}

```

### ArrayList的添加元素 & 扩容机制
***

#### 在集合末尾添加一个元素的方法
```java
//成员变量 size 标识集合当前元素个数初始为 0
int size；
/**
 * 将指定元素添加到集合（底层数组）末尾
 * @param 将要添加的元素
 * @return 返回 true 表示添加成功
 */
 public boolean add(E e) {
     
    //检查当前底层数组容量，如果容量不够则进行扩容
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    //将数组添加一个元素，size 加 1
    elementData[size++] = e;
    return true;
 }
```

调用 add 方法的时候总会调用 ensureCapacityInternal 来判断是否需要进行数组扩容，ensureCapacityInternal 参数为当前集合长度 size + 1，这很好理解，是否需要扩充长度，需要看当前底层数组是否够放 size + 1 个元素的。

#### 扩容机制
```java
//扩容检查
private void ensureCapacityInternal(int minCapacity) {
    //如果是无参构造方法构造的的集合，第一次添加元素的时候会满足这个条件 minCapacity 将会被赋值为 10
   if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
       minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
   }
    // 将 size + 1 或 10 传入 ensureExplicitCapacity 进行扩容判断
   ensureExplicitCapacity(minCapacity);
}

private void ensureExplicitCapacity(int minCapacity) {
  //操作数加 1 用于保证并发访问 
   modCount++;
   // 如果 当前数组的长度比添加元素后的长度要小则进行扩容 
   if (minCapacity - elementData.length > 0)
       grow(minCapacity);
}
```

上边的源码主要做了扩容前的判断操作，注意参数为当前集合元素个数+1，第一次添加元素的时候 size + 1 = 1 ,而 elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA, 长度为 0 ，1 - 0 > 0,  所以需要进行 grow 操作也就是扩容。

```java
/**
 * 集合的最大长度 Integer.MAX_VALUE - 8 是为了减少出错的几率 Integer 最大值已经很大了
 */
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

/**
 * 增加容量，以确保它至少能容纳最小容量参数指定的元素个数。
 * @param 满足条件的最小容量
 */
private void grow(int minCapacity) {
  //获取当前 elementData 的大小，也就是 List 中当前的容量
   int oldCapacity = elementData.length;
   //oldCapacity >> 1 等价于 oldCapacity / 2  所以新容量为当前容量的 1.5 倍
   int newCapacity = oldCapacity + (oldCapacity >> 1);
   //如果扩大1.5倍后仍旧比 minCapacity 小那么直接等于 minCapacity
   if (newCapacity - minCapacity < 0)
       newCapacity = minCapacity;
    //如果新数组大小比  MAX_ARRAY_SIZE 就需要进一步比较 minCapacity 和 MAX_ARRAY_SIZE 的大小
   if (newCapacity - MAX_ARRAY_SIZE > 0)
       newCapacity = hugeCapacity(minCapacity);
   // minCapacity通常接近 size 大小
   //使用 Arrays.copyOf 构建一个长度为 newCapacity 新数组 并将 elementData 指向新数组
   elementData = Arrays.copyOf(elementData, newCapacity);
}

/**
 * 比较 minCapacity 与 Integer.MAX_VALUE - 8 的大小如果大则放弃-8的设定，设置为 Integer.MAX_VALUE 
 */
private static int hugeCapacity(int minCapacity) {
   if (minCapacity < 0) // overflow
       throw new OutOfMemoryError();
   return (minCapacity > MAX_ARRAY_SIZE) ?
       Integer.MAX_VALUE :
       MAX_ARRAY_SIZE;
}
```

由此看来 ArrayList 的扩容机制的知识点一共又两个

1、每次扩容的大小为原来大小的 1.5倍 （当然这里没有包含 1.5倍后大于 MAX_ARRAY_SIZE 的情况）

2、扩容的过程其实是一个将原来元素拷贝到一个扩容后数组大小的长度新数组中。所以 ArrayList 的扩容其实是相对来说比较消耗性能的。

