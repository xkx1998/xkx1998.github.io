---
layout:     post
title:      LinkedList源码学习
subtitle:   
date:       2019-3-20
author:     BY xukexiang
header-img: img/charlotte/b2f68e7ebd6314a8358661a765ca9095527eeee1.jpg
catalog: true
tags:
    - Typora
---

## LinkedList的源码学习

ArrayList在我们开发中经常用到，而且基本面试必问，所以它还是挺重要的。ArrayList的源码我感觉还是比较容易看的懂的，感觉跟HashMap那些源码比起来是
比较容易的。以下是自己的一些总结和在别人博客上学到的一些东西，以前自己写代码只要创建List对象就
用ArrayList这个子类对象。也没有认真分析过ArrayList的源码。感觉分析源码的过程有点难，但是感觉
也挺有意思的。毕竟你知道ArrayList是如何实现的，以后面试人家怎么也问不倒你，自己也可以更好的运用
集合。下面就开始分析源码吧。

### LinkedList源码的分析思路

1.LinkedList 的概述

2.LinkedList 的构造方法

3.LinkedList 的增删改查。

4.LinkedList 作为队列（Queue）使用的时候增删改查。

5.LinkedList 的遍历方法

#### LinkedList概述
![关系图](/img/2019-3-20-LinkedList/1553089001.jpg)

#### LinkedList的基本特点
1.LinkedList 集合底层实现的数据结构为双向链表

2.LinkedList 集合中元素允许为 null

3.允许存放重复数据，存储顺序按照元素的添加顺序

4.LinkedList 是非线程安全的，如果想保证线程安全的前提下操作 LinkedList，
可以使用 List list = Collections.synchronizedList(new LinkedList(...));
 来生成一个线程安全的 LinkedList
 
#### LinkedList 双向链表实现及成员变量
概述上说了双向链表的特点，而 LinkedList 又继承自 Deque 这个双链表接口，
在介绍 LinkedList 的具体方法前我们先了解下双向链表的实现。
```java
private static class Node<E> {
   // 当前节点的元素值
   E item;
   // 下一个节点的索引
   Node<E> next;
   // 上一个节点的索引
   Node<E> prev;

   Node(Node<E> prev, E element, Node<E> next) {
       this.item = element;
       this.next = next;
       this.prev = prev;
   }
}
```
正如我们所说，LinkedList 的节点实现完全符合双向链表的数据结构要求，
而构造方法第一个参数为上一个节点的索引，当前节点的元素，下一个节点索引。

LinkedList 主要成员变量有下边三个：

```java
//LinkedList 中的节点个数
transient int size = 0;

//LinkedList 链表的第一个节点
transient Node<E> first;

//LinkedList 链表的最后一个节点
transient Node<E> last;
```
### LinkedList 的构造方法
LinkedList 有两个构造函数：
***

```java
/**
 * 空参数的构造由于生成一个空链表 first = last = null
 */
 public LinkedList() {
 }

/**
 * 传入一个集合类，来构造一个具有一定元素的 LinkedList 集合
 * @param  c  其内部的元素将按顺序作为 LinkedList 节点
 * @throws NullPointerException 如果 参数 collection 为空将抛出空指针异常
 */
public LinkedList(Collection<? extends E> c) {
   this();
   addAll(c);
}

```

对于上述几个成员变量，我们只是在注释中简单的说明，对于他们具体有什么作用，在下边分析构造方法和扩容机制的时候将会更详细的讲解。

ArrayList 一共三种构造方式，我们先从无参的构造方法来开始：

### LinkedList 的增删改查
***
#### LinkedList 添加节点的方法
***
LinkedList 作为链表数据结构的实现，
不同于数组，它可以方便的在头尾插入一个节点，
而 add 方法默认在链表尾部添加节点：
```java
/**
 * Inserts the specified element at the beginning of this list.
 *
 * @param e the element to add
 */
 public void addFirst(E e) {
    linkFirst(e);
 }

/**
 * Appends the specified element to the end of this list.
 *
 * <p>This method is equivalent to {@link #add}.
 *
 * @param e the element to add
 */
 public void addLast(E e) {
    linkLast(e);
 }
    
/**
 * Appends the specified element to the end of this list.
 *
 * <p>This method is equivalent to {@link #addLast}.
 *
 * @param e element to be appended to this list
 * @return {@code true} (as specified by {@link Collection#add})
 */
 public boolean add(E e) {
    linkLast(e);
    return true;
 }

```

上述英文太过简单不翻译了，我们可以看到 add 方法是有返回值的，
这个可以注意下。看来这一系方法都调用用了 linkXXX 方法，

```java
 /**
  * 添加一个元素在链表的头节点位置
  */
private void linkFirst(E e) {
   // 添加元素之前的头节点
   final Node<E> f = first;
   //以添加的元素为节点值构建新的头节点 并将 next 指针指向 之前的头节点
   final Node<E> newNode = new Node<>(null, e, f);
   // first 索引指向将新的节点
   first = newNode;
   // 如果添加之前链表空则新的节点也作为未节点
   if (f == null)
       last = newNode;
   else
       f.prev = newNode;//否则之前头节点的 prev 指针指向新节点
   size++;
   modCount++;//操作数++
}

/**
 * 在链表末尾添加一个节点
 */
 void linkLast(E e) {
   final Node<E> l = last;//保存之前的未节点
   //构建新的未节点，并将新节点 prev 指针指向 之前的未节点
   final Node<E> newNode = new Node<>(l, e, null);
   //last 索引指向末节点
   last = newNode;
   if (l == null)//如果之前链表为空则新节点也作为头节点
       first = newNode;
   else//否则将之前的未节点的 next 指针指向新节点
       l.next = newNode;
   size++;
   modCount++;//操作数++
}
```

#### LinkedList 删除节点的方法


除了上述几种添加元素的方法，以及之前在将构造的时候说明的 addAll 方法，
LinkedList 还提供了 add(int index, E element); 方法，下面我们来看在这个方法：
