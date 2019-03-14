---
layout:     post
title:      ArrayList源码学习
subtitle:   
date:       2019-1-21
author:     BY xukexiang
header-img: img/charlotte/b2f68e7ebd6314a8358661a765ca9095527eeee1.jpg
catalog: true
tags:
    - Typora
---

## ArrayList的源码学习

ArrayList在我们开发中经常用到，而且基本面试必问，所以它还是挺重要的。ArrayList的源码我感觉还是比较容易看的懂的，感觉跟HashMap那些源码比起来是
比较容易的。以下是自己的一些总结和在别人博客上学到的一些东西，以前自己写代码只要创建List对象就
用ArrayList这个子类对象。也没有认真分析过ArrayList的源码。感觉分析源码的过程有点难，但是感觉
也挺有意思的。毕竟你知道ArrayList是如何实现的，以后面试人家怎么也问不倒你，自己也可以更好的运用
集合。下面就开始分析源码吧。

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


- ArrayList 实现 RandomAccess 接口标识着其支持随机快速访问，
查看源码可以知道RandomAccess 其实只是一个标识，
标识某个类拥有随机快速访问的能力，
针对 ArrayList 而言通过 get(index)去访问元素可以达到 O(1) 的时间复杂度。
有些集合类不拥有这种随机快速访问的能力，比如 LinkedList 就没有实现这个接口。


- ArrayList 实现 Cloneable 接口标识着他可以被克隆/复制，
其内部实现了 clone 方法供使用者调用来对 ArrayList 进行克隆，
但其实现只通过 Arrays.copyOf 完成了对 ArrayList 进行「浅复制」，
也就是你改变 ArrayList clone后的集合中的元素，源集合中的元素也会改变。


- 对于 java.io.Serializable 标识着集合可被被序列化。
我们发现了一些有趣的事情，除了List<E> 以外，
ArrayList 实现的接口都是标识接口，标识着这个类具有怎样的特点，看起来更像是一个属性。

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

这是我们经常使用的一个构造方法，其内部实现只是将 elementData 指向了我们刚才讲得 
DEFAULTCAPACITY_EMPTY_ELEMENTDATA 这个空数组，
这个空数组的容量是 0， 但是源码注释却说这是构造一个初始容量为10的空列表。这是为什么？
其实在集合调用 add 方法添加元素的时候将会调用 ensureCapacityInternal 方法，
在这个方法内部判断了：

```java
if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
       minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
}
```

可见，如果采用无参数构造方法的时候第一次添加元素肯定走进 if 判断中
 minCapacity 将被赋值为 10，所以「构造一个初始容量为10的空列表。」也就是这个意思。


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

调用 add 方法的时候总会调用 ensureCapacityInternal 来判断是否需要进行数组扩容，
ensureCapacityInternal 参数为当前集合长度 size + 1，这很好理解，
是否需要扩充长度，需要看当前底层数组是否够放 size + 1 个元素的。

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

上边的源码主要做了扩容前的判断操作，
注意参数为当前集合元素个数+1，
第一次添加元素的时候 size + 1 = 1 
,而 elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA, 
长度为 0 ，minCapacity被赋值为10，10 - 0 > 0,  所以需要进行 grow 操作也就是扩容。

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

由此看来 ArrayList 的扩容机制的知识点如下
1、先调用ensureCapacityInternal(int Capacity)
和ensureExplicitCapacity(int Capacity)进行扩容检查，
若参数size + 1 小于数组的长度elementData.length
则要调用grow方法进行扩容

2、每次扩容的大小为原来大小的 1.5倍,若扩容或的newCapacity仍然小于oldCapacity
newCapacity就直接等于oldCapacity,然后newCapacity再与MAX_ARRAY_SIZE相比较
若newCapaticy比MAX_ARRAY_SIZE大的话，就直接将newCapacity直接设置为Integer.MAX_VALUE 
（当然这里没有包含 1.5倍后大于 MAX_ARRAY_SIZE 的情况）

3、调用Arrays.copyOf(elementData,newCapaticy)
扩容的过程其实是一个将原来元素拷贝到一个扩容后数组大小的长度新数组中。
所以 ArrayList 的扩容其实是相对来说比较消耗性能的。


#### 在指定角标位置添加元素的方法
```java
/**
* 将指定的元素插入该列表中的指定位置。将当前位置的元素(如果有)和任何后续元素移到右边(将一个元素添加到它们的索引中)。
* 
* @param 要插入的索引位置
* @param 要添加的元素
* @throws 如果 index 大于集合长度 小于 0 则抛出角标越界 IndexOutOfBoundsException 异常
*/
public void add(int index, E element) {
   // 检查角标是否越界
   rangeCheckForAdd(index);
    // 扩容检查
   ensureCapacityInternal(size + 1);      
   //调用 native 方法新型数组拷贝
   System.arraycopy(elementData, index, elementData, 
                    index + 1,size - index);
    // 添加新元素
   elementData[index] = element;
   size++;
}
```

我们知道一个数组是不能在角标位置直接插入元素的，
ArrayList 通过数组拷贝的方法将指定角标位置以及其后续元素整体向后移动一个位置，
空出 index 角标的位置，来赋值新的元素。
将一个数组 src 起始 srcPos 角标之后 length 长度间的元素，
赋值到 dest  数组中 destPos 到 destPos + length -1长度角标位置上。
只是在 add 方法中 src 和 destPos 为同一个数组而已。

native方法如下:
```java
public static native void arraycopy(Object src,  int  srcPos,
                                        Object dest, int destPos,
                                        int length);

```

#### 批量添加元素

```java
public boolean addAll(Collection<? extends E> c) {
        // 调用 c.toArray 将集合转化数组
        Object[] a = c.toArray();
        // 要添加的元素的个数
        int numNew = a.length;
        //扩容检查以及扩容
        ensureCapacityInternal(size + numNew);  // Increments modCount
        //将参数集合中的元素添加到原来数组 [size，size + numNew -1] 的角标位置上。
        System.arraycopy(a, 0, elementData, size, numNew);
        size += numNew;
        //与单一添加的 add 方法不同的是批量添加有返回值，如果 numNew == 0 表示没有要添加的元素则需要返回 false 
        return numNew != 0;
}
```

#### 在数组指定角标位置添加
```java
public boolean addAll(int index, Collection<? extends E> c) {
        //同样检查要插入的位置是否会导致角标越界
        rangeCheckForAdd(index);
        
        Object[] a = c.toArray();
        int numNew = a.length;
        ensureCapacityInternal(size + numNew); 
        //这里做了判断，如果要numMoved > 0 代表插入的位置在集合中间位置，和在 numMoved == 0最后位置 则表示要在数组末尾添加 如果 < 0  rangeCheckForAdd 就跑出了角标越界
        int numMoved = size - index;
        if (numMoved > 0)
            System.arraycopy(elementData, index, elementData, index + numNew,
                             numMoved);

        System.arraycopy(a, 0, elementData, index, numNew);
        size += numNew;
        return numNew != 0;
    }
    
private void rangeCheckForAdd(int index) {
   if (index > size || index < 0)
       throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
```

#### ArrayList的遍历
ArrayList 的遍历方式 jdk 1.8 之前有三种 ：for 循环遍历， foreach 遍历，
迭代器遍历,jdk 1.8 之后又引入了forEach 操作，我们先来看看迭代器的源码实现：


##### 迭代器
迭代器 Iterator 模式是用于遍历各种集合类的标准访问方法。
它可以把访问逻辑从不同类型的集合类中抽象出来，从而避免向客户端暴露集合的内部结构。
 ArrayList 作为集合类也不例外，迭代器本身只提供三个接口方法：
```java
   public interface Iterator {
        boolean hasNext();//是否还有下一个元素
        Object next();// 返回当前元素 可以理解为他相当于 fori 中 i 索引
        void remove();// 移除一个当前的元素 也就是 next 元素。
    }
```

ArrayList 中调用 iterator() 将会返回一个内部类对象 Itr 其实现了 Iterator 接口。

```java
public Iterator<E> iterator() {
        return new Itr();
}
```
下面让我们看下其实现的源码：
正如我们的 for 循环遍历一样，数组角标总是从 0 开始的，
所以 cursor 初始值为 0 ， hasNext 表示是否遍历到数组末尾，
即 i < size 。对于 modCount 变量之所以一直没有介绍是因为他集合并发访问有关系，
用于标记当前集合被修改（增删）的次数，如果并发访问了集合那么将会导致这个 modCount 的变化，
在遍历过程中不正确的操作集合将会抛出 ConcurrentModificationException ，
这是 Java 「fast-fail 的机制」，对于如果正确的在遍历过程中操作集合稍后会有说明。


```java
private class Itr implements Iterator<E> {
   int cursor; // 对照 hasNext 方法 cursor 应理解为下个调用 next 返回的元素 初始为 0
   int lastRet = -1; // 上一个返回的角标
   int expectedModCount = modCount;//初始化的时候将其赋值为当前集合中的操作数，
   // 是否还有下一个元素 cursor == size 表示当前集合已经遍历完了 所以只有当 cursor 不等于 size 的时候 才会有下一个元素
   public boolean hasNext() {
       return cursor != size;
   }
}
```

next 方法是我们获取集合中元素的方法，next 返回当前遍历位置的元素，
如果在调用 next 之前集合被修改，并且迭代器中的期望操作数并没有改变，
将会引发ConcurrentModificationException。
next 方法多次调用 checkForComodification 来检验这个条件是否成立。


```java
   @SuppressWarnings("unchecked")
   public E next() {
        // 验证期望的操作数与当前集合中的操作数是否相同 如果不同将会抛出异常
       checkForComodification();
       // 如果迭代器的索引已经大于集合中元素的个数则抛出异常，这里不抛出角标越界
       int i = cursor;
       if (i >= size)
           throw new NoSuchElementException();
           
       Object[] elementData = ArrayList.this.elementData;
       // 由于多线程的问题这里再次判断是否越界，如果有异步线程修改了List（增删）这里就可能产生异常
       if (i >= elementData.length)
           throw new ConcurrentModificationException();
       // cursor 移动
       cursor = i + 1;
       //最终返回 集合中对应位置的元素，并将 lastRet 赋值为已经访问的元素的下标
       return (E) elementData[lastRet = i];
   }
```

只有 Iterator 的 remove 方法会在调用集合的 remove 之后让 期望 操作数改变使expectedModCount与 modCount 再相等，所以是安全的。


```java
    // 实质调用了集合的 remove 方法移除元素
   public void remove() {
        // 比如操作者没有调用 next 方法就调用了 remove 操作，lastRet 等于 -1的时候抛异常
       if (lastRet < 0)
           throw new IllegalStateException();
           
        //检查操作数
       checkForComodification();
    
       try {
            //移除上次调用 next 访问的元素
           ArrayList.this.remove(lastRet);
           // 集合中少了一个元素，所以 cursor 向前移动一个位置（调用 next 时候 cursor = lastRet + 1）
           cursor = lastRet;
           //删除元素后赋值-1，确保先前 remove 时候的判断
           lastRet = -1;
           //修改操作数期望值， modCount 在调用集合的 remove 的时候被修改过了。
           expectedModCount = modCount;
       } catch (IndexOutOfBoundsException ex) {
            // 集合的 remove 会有可能抛出 rangeCheck 异常，catch 掉统一抛出 ConcurrentModificationException 
           throw new ConcurrentModificationException();
       }
   }
```

#### 错误操作导致 ConcurrentModificationException 异常

我们分析迭代器的时候，知道 ConcurrentModificationException是指因为迭代器调用 checkForComodification 
方法比较 modCount 和 expectedModCount 方法大小的时候抛出异常。
我们在分析 ArrayList 的时候在每次对集合进行修改， 即有 add 和 remove 操作的时候每次都会对 modCount ++。
modCount 这个变量主要用来记录 ArrayList 被修改的次数，那么为什么要记录这个次数呢？
是为了防止多线程对同一集合进行修改产生错误，记录了这个变量，
在对 ArrayList 进行迭代的过程中我们能很快的发现这个变量是否被修改过，
如果被修改了 ConcurrentModificationException 将会产生。
下面我们来看下例子，这个例子并不是在多线程下的，而是因为我们在同一线程中对 list 进行了错误操作导致的：


```java
Iterator<SubClass> iterator = lists.iterator();

while (iterator.hasNext()) {
  SubClass next = iterator.next();
  int index = next.test;
  if (index == 3) {
      list2.remove(index);//操作1： 注意是 list2.remove 操作
      //iterator.remove()；/操作2 注意是 iterator.remove 操作
  }
}
//操作1： Exception in thread "main" java.util.ConcurrentModificationException
//操作2：  [SubClass{test=1}, SubClass{test=2}]
System.out.println(list2);
```


我们对操作1，2分别运行程序，可以看到，操作1很快就抛出了 java.util.ConcurrentModificationException 异常，
操作2 则顺利运行出正常结果，如果对 modCount 注意了的话，我们很容易理解，
list.remove(index) 操作会修改List 的 modCount，
而 iterator.next() 内部每次会检验 expectedModCount != modCount，
所以当我们使用 list.remove 下一次再调用 iterator.next() 就会报错了，
而iterator.remove为什么是安全的呢？
因为其操作内部会在调用 list.remove 后重新将新的 modCount 赋值给  expectedModCount。
所以我们直接调用 list.remove 操作是错误的。


### 总结：
1.ArrayList 底层是一个动态扩容的数组结构,每次扩容需要增加1.5倍的容量

2.ArrayList 扩容底层是通过 Arrays.CopyOf 和  System.arraycopy 来实现的。每次都会产生新的数组，和数组中内容的拷贝，所以会耗费性能，所以在多增删的操作的情况可优先考虑 LinkList 而不是 ArrayList。

3.ArrayList 的 toArray 方法重载方法的使用。

4.允许存放（不止一个） null 元素，

5.允许存放重复数据，存储顺序按照元素的添加顺序

6.ArrayList 并不是一个线程安全的集合。如果集合的增删操作需要保证线程的安全性，可以考虑使用 CopyOnWriteArrayList 或者使collections.synchronizedList(List l)函数返回一个线程安全的ArrayList类.

7.不正确访问集合元素的时候 ConcurrentModificationException和 java.lang.IndexOutOfBoundsException 异常产生的时机和原理。


