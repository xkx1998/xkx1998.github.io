---
layout:     post
title:      Java8 Stream
subtitle:   
date:       2020-6-7
author:     BY xukexiang
header-img: img/charlotte/b2f68e7ebd6314a8358661a765ca9095527eeee1.jpg
catalog: true
tags:
    - Typora
---

## Java8 List的Stream流操作

### Stream流
Stream 中文称为 “流”，通过将集合转换为这么一种叫做 “流” 的元素序列，
通过声明性方式，能够对集合中的每个元素进行一系列并行或串行的流水线操作。

函数式编程带来的好处尤为明显。这种代码更多地表达了业务逻辑的意图，而不是它的实现机制。易读的代码也易于维护、更可靠、更不容易出错。

*面对一对多结构，查询主实体时需要附带主实体的子实体列表怎么写？查出主列表，循环差子列表*

List的Stream流操作可以简化我们的代码，减少程序运行的压力，应对上面的问题，
以前的话是先查出对应的list数据，然后根据取到集合中id去查找对应的子实体中数据，
接着在放入对应的集合中去，key值表示主实体的id，value值表示对应主实体id查到的结合数据，
这样就会三次foreach循环组装数据，会很麻烦，当数据量大的时候，会增加程序运行的负荷，
造成运行缓慢。所以，流式操作代替我们的这一堆操作，提高了代码的简易性，可维护性，可靠性，更不容易出错。

### 常用到的方法
1. stream() / parallelStream()  最常用到的方法，将集合转换为流
2. filter(T -> boolean)  保留 boolean 为 true 的元素
3. distinct()，去除重复元素，
4. sorted() / sorted((T, T) -> int)
如果流中的元素的类实现了 Comparable 接口，即有自己的排序规则，那么可以直接调用 sorted() 方法对元素进行排序，如 Stream<Integer>
反之, 需要调用 sorted((T, T) -> int) 实现 Comparator 接口
5. limit(long n)  返回前n个元素。
6. skip(long n)  去除前 n 个元素
7. map(T -> R) 将流中的每一个元素 T 映射为 R（类似类型转换）
```java
List<String> newlist = list.stream().map(Person::getName).collect(toList());
```
8. flatMap(T -> Stream<R>)
将流中的每一个元素 T 映射为一个流，再把每一个流连接成为一个流
```java
List<String> list = new ArrayList<>();
list.add("aaa bbb ccc");
list.add("ddd eee fff");
list.add("ggg hhh iii");
list = list.stream().map(s -> s.split(" ")).flatMap(Arrays::stream).collect(toList());
```
上面例子中，我们的目的是把 List 中每个字符串元素以" "分割开，变成一个新的 List<String>。
首先 map 方法分割每个字符串元素，但此时流的类型为 Stream<String[ ]>，因为 split 方法返回的是 String[ ] 类型；所以我们需要使用 flatMap 方法，先使用Arrays::stream将每个 String[ ] 元素变成一个 Stream<String> 流，然后 flatMap 会将每一个流连接成为一个流，最终返回我们需要的 Stream<String>

9. anyMatch(T -> boolean)
流中是否有一个元素匹配给定的 T -> boolean 条件
是否存在一个 person 对象的 age 等于 20：
```java
boolean b = list.stream().anyMatch(person -> person.getAge() == 20);
```

10. allMatch(T -> boolean)
流中是否所有元素都匹配给定的 T -> boolean 条件

11. noneMatch(T -> boolean)
流中是否没有元素匹配给定的 T -> boolean 条件

12. findAny() 和 findFirst()
findAny()：找到其中一个元素 （使用 stream() 时找到的是第一个元素；使用 parallelStream() 并行时找到的是其中一个元素）
findFirst()：找到第一个元素

13. reduce((T, T) -> T) 和 reduce(T, (T, T) -> T)
用于组合流中的元素，如求和，求积，求最大值等
```java
计算年龄总和：
int sum = list.stream().map(Person::getAge).reduce(0, (a, b) -> a + b);
与之相同:
int sum = list.stream().map(Person::getAge).reduce(0, Integer::sum);
```
14. count()
返回流中元素个数，结果为 long 类型

15. collect()
收集方法，我们很常用的是 collect(toList())，当然还有 collect(toSet()) 等，参数是一个收集器接口，这个后面会另外讲

16. forEach()
还有这个比较不起眼的方法，返回一个等效的无序流，当然如果流本身就是无序的话，那可能就会直接返回其本身

打印各个元素：
list.stream().forEach(System.out::println);




