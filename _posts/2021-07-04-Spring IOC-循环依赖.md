---
layout:     post
title:      Spring IOC如何循环依赖处理
subtitle:   
date:       2019-3-26
author:     BY xukexiang
header-img: img/charlotte/b2f68e7ebd6314a8358661a765ca9095527eeee1.jpg
catalog: true
tags:
    - Typora
---

### 什么是循环依赖
循环依赖实际上就是循环引入，就是有两个或以上的bean相互引用对方，例如有两个对象A和B，A引用B，B引用A，
形成了一个闭环，这就是IOC的循环依赖问题。示例图如下：
![示例图](/img/循环依赖/202105092052019271.png)

Spring的循环依赖有两种场景:
1) 构造器方法注入
2) setter方法注入

对于情况1的构造方法注入，Spring是无法解决的，原因是: 因为我们都知道对象的创建分为两步，
实例化和初始化，构造方法是在对象实例化的时候才执行的，这时候其他的对象还没有实例化完成，
因此不可能在实例化的时候就获取到其他bean对象。Spring只能解决setter注入的循环依赖问题，
而且还要限制bean的scope为singleton, 不能为prototype,因为scope为prototype的bean无法
放到三个缓存map中，无法提前暴露正在创建的对象。


### 解决循环依赖
创建Bean的大致涉及到的方法有如下:
- getBean -> doGetBean -> createBean -> doCreateBean -> populateBean -> initializeBean

我们先从加载 bean 最初始的方法 doGetBean() 开始。 
在 doGetBean() 中调用getSingleton(), 首先会根据 beanName 从一级缓存singletonObjects中获取，如果不为空则直接返回。
```java
Object sharedInstance = getSingleton(beanName);
```
调用 getSingleton() 方法从单例缓存中获取，如下：
```java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
            Object singletonObject = this.singletonObjects.get(beanName);
            if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {//判断一级缓存为空并且传入beanName的对象正在创建中
                synchronized (this.singletonObjects) {
                    singletonObject = this.earlySingletonObjects.get(beanName);
                    if (singletonObject == null && allowEarlyReference) {
                        ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                        if (singletonFactory != null) {
                            singletonObject = singletonFactory.getObject();
                            //将提前暴露的对象放入二级缓存中，将三级缓存中的对移除，只能保留一个
                            this.earlySingletonObjects.put(beanName, singletonObject);
                            this.singletonFactories.remove(beanName);
                        }
                    }
                }
            }
            return singletonObject;
}
```
这个方法主要是从三个缓存中获取，分别是：singletonObjects、earlySingletonObjects、singletonFactories，三者定义如下：
```java
/** Cache of singleton objects: bean name --> bean instance */
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);
    
/** Cache of singleton factories: bean name --> ObjectFactory */
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
    
/** Cache of early singleton objects: bean name --> bean instance */
private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);
    
```
三级缓存的意义如下:
- singletonObjects(一级缓存): 存放的是完整对象
- earlySingletonObjects(二级缓存): 存放的是半成品对象(实例化完成但未初始化), 也就是执行在三级缓存中存放的lambda表达式得到的结果
- singletonFactories(三级缓存): 存放的是一个lambda表达式，这个表达式是实现了一个函数式接口的抽象方法，返回一个原始对象或者代理对象

他们就是 Spring 解决 singleton bean 的关键因素所在，我称他们为三级缓存，第一级为 singletonObjects，第二级为 earlySingletonObjects，
第三级为 singletonFactories。这里我们可以通过 getSingleton() 看到他们是如何配合的，这分析该方法之前，提下其中的 isSingletonCurrentlyInCreation() 和 allowEarlyReference。

- isSingletonCurrentlyInCreation()：判断当前 singleton bean 是否处于创建中。
bean 处于创建中也就是说 bean 在初始化但是没有完成初始化，有一个这样的过程其实和 Spring 解决 bean 循环依赖的理念相辅相成，
因为 Spring 解决 singleton bean 的核心就在于提前曝光 bean。

- allowEarlyReference：从字面意思上面理解就是允许提前拿到引用。
其实真正的意思是是否允许从 singletonFactories 缓存中通过 getObject() 拿到对象，
为什么会有这样一个字段呢？原因就在于 singletonFactories 才是 Spring 解决 singleton bean 的诀窍所在，
singletonFactories的beanName所对应value，存放了一个实现lambda表达式，这个lambda表达式执行后会返回一个提前暴露的对象。

getSingleton() 整个过程如下：首先从一级缓存 singletonObjects 获取，如果没有且当前指定的 beanName 正在创建，就再从二级缓存中 earlySingletonObjects 获取，
如果还是没有获取到且运行 singletonFactories 通过 getObject() 获取，则从三级缓存 singletonFactories 获取，如果获取到则，
通过其 getObject() 获取对象，并将其加入到二级缓存 earlySingletonObjects 中 从三级缓存 singletonFactories 删除，如下：

```java
singletonObject = singletonFactory.getObject();
this.earlySingletonObjects.put(beanName, singletonObject);
this.singletonFactories.remove(beanName);
```
这样就从三级缓存升级到二级缓存了。beanName只能在三级缓存或者二级缓存存在。
上面是从缓存中获取，但是缓存中的数据从哪里添加进来的呢？

一直往下跟会发现在 doCreateBean() ( AbstractAutowireCapableBeanFactory ) 中，有这么一段代码：
```java
boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences && isSingletonCurrentlyInCreation(beanName));
if (earlySingletonExposure) {
    if (logger.isTraceEnabled()) {
        logger.trace("Eagerly caching bean '" + beanName +
                "' to allow for resolving potential circular references");
    }
    addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
}
```

如果 earlySingletonExposure == true 的话，则调用 addSingletonFactory() 将他们添加到缓存中，
但是一个 bean 要具备如下条件才会添加至缓存中：

- scope为singleton
- 可以提前暴露bean
- 当前bean正在创建中

addSingletonFactory() 代码如下：
```java
protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
    Assert.notNull(singletonFactory, "Singleton factory must not be null");
    synchronized (this.singletonObjects) {
        if (!this.singletonObjects.containsKey(beanName)) {
            this.singletonFactories.put(beanName, singletonFactory);
            this.earlySingletonObjects.remove(beanName);
            this.registeredSingletons.add(beanName);
        }
    }
}
```

从这段代码我们可以看出 singletonFactories 这个三级缓存才是解决 Spring Bean 循环依赖的诀窍所在。
同时这段代码发生在 createBeanInstance() 方法之后，也就是说这个 bean 其实已经被创建出来了，
但是它还不是很完美（没有进行属性填充和初始化），但是对于其他依赖它的对象而言已经足够了（可以根据对象引用定位到堆中对象），
能够被认出来了，所以 Spring 在这个时候选择将该对象提前曝光出来让大家认识认识。 
介绍到这里我们发现三级缓存 singletonFactories 和 二级缓存 earlySingletonObjects 中的值都有出处了，
那一级缓存在哪里设置的呢？在doGetBean() 中可以发现这个 addSingleton() 方法，放入以及缓存是在对象完整创建完后放入，源码如下：

```java
protected void addSingleton(String beanName, Object singletonObject) {
    synchronized (this.singletonObjects) {
        this.singletonObjects.put(beanName, singletonObject);
        this.singletonFactories.remove(beanName);
        this.earlySingletonObjects.remove(beanName);
        this.registeredSingletons.add(beanName);
    }           
}
```

添加至一级缓存，同时从二级、三级缓存中删除。这个方法在我们创建 bean 的链路中有哪个地方引用呢？
，在 doGetBean() 处理不同 scope 时，如果是 singleton，则调用 getSingleton()，如下：
```java
// Create bean instance.
if (mbd.isSingleton()) {
    sharedInstance = getSingleton(beanName, () -> {
        try {
            return createBean(beanName, mbd, args);
        }
        catch (BeansException ex) {
            // Explicitly remove instance from singleton cache: It might have been put there
            // eagerly by the creation process, to allow for circular reference resolution.
            // Also remove any beans that received a temporary reference to the bean.
            destroySingleton(beanName);
            throw ex;
        }
    });
    bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
}
```

```java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
    Assert.notNull(beanName, "Bean name must not be null");
    synchronized (this.singletonObjects) {
        Object singletonObject = this.singletonObjects.get(beanName);
        if (singletonObject == null) {
            //....
            try {
                //调用lambda表达式中的createBean()
                singletonObject = singletonFactory.getObject();
                newSingleton = true;
            }
            //.....
            if (newSingleton) {
                //放入一级缓存
                addSingleton(beanName, singletonObject);
            }
        }
        return singletonObject;
    }
}
```

最后我们来总结一个，Spring解决singleton bean循环依赖的过程，假设有A、B、C三个对象，A依赖B，B依赖C，C依赖A。
在创建A对象的时候, 在执行到doCreateBean()方法的时候，先提前暴露对象，将为初始化完成的A对象放到三级缓存中。
接着填充A对象的属性B。接着创建B对象，B对象执行跟A对象的相同流程，接着创建C对象，在填充C对象属性A的时候，调用
getSingleton()，可以调用ObjectFactory.getObject()获取到三级缓存中A对象的半成品，此时C对象可以创建完毕，
接着调用addSingleton()把完整的C对象放到一级缓存中，C对象初始化完毕，接着B也可以拿到A，A拿到B，A、B、C都初始化完毕。


几个问题:
1) 只有一级缓存可以吗？
答: 不可以，因为一级缓存和二级缓存分别存放的是完整对象和半成品对象，只用一级缓存区分不了。
2) 只有二级缓存可以吗?
答: 拿到ObjectFactory对象后，调用ObjectFactory.getObject()方法最终会调用getEarlyBeanReference()方法，
getEarlyBeanReference这个方法主要逻辑大概描述下如果bean被AOP切面代理则返回的是beanProxy对象，
如果未被代理则返回的是原bean实例，这时我们会发现能够拿到bean实例(属性未填充)，
然后从三级缓存移除，放到二级缓存earlySingletonObjects中，而此时B注入的是一个半成品的实例A对象，
不过随着B初始化完成后，A会继续进行后续的初始化操作，最终B会注入的是一个完整的A实例，
因为在内存中它们是同一个对象。下面是重点，我们发现这个二级缓存好像显得有点多余，好像可以去掉，
只需要一级和三级缓存也可以做到解决循环依赖的问题？？？
只要两个缓存确实可以做到解决循环依赖的问题，但是有一个前提这个bean没被AOP进行切面代理，
如果这个bean被AOP进行了切面代理，那么只使用两个缓存是无法解决问题，下面来看一下bean被AOP进行了切面代理的场景


我们会发现再执行一遍singleFactory.getObject()方法又是一个新的代理对象，这就会有问题了，
因为AService是单例的，每次执行singleFactory.getObject()方法又会产生新的代理对象，
假设这里只有一级和三级缓存的话，我每次从三级缓存中拿到singleFactory对象，执行getObject()方法又会产生新的代理对象，
这是不行的，因为AService是单例的，所有这里我们要借助二级缓存来解决这个问题，将执行了singleFactory.getObject()产生的对象放到二级缓存中去，
后面去二级缓存中拿，没必要再执行一遍singletonFactory.getObject()方法再产生一个新的代理对象，
保证始终只有一个代理对象。还有一个注意的点: 既然singleFactory.getObject()返回的是代理对象，
那么注入的也应该是代理对象，我们可以看到注入的确实是经过CGLIB代理的AService对象。
所以如果没有AOP的话确实可以两级缓存就可以解决循环依赖的问题，如果加上AOP，两级缓存是无法解决的，
不可能每次执行singleFactory.getObject()方法都给我产生一个新的代理对象，所以还要借助另外一个缓存来保存产生的代理对象

