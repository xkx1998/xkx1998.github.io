---
layout:     post
title:      Spring Ioc(Bean的装配)
subtitle:   
date:       2019-3-26
author:     BY xukexiang
header-img: img/charlotte/b2f68e7ebd6314a8358661a765ca9095527eeee1.jpg
catalog: true
tags:
    - Typora
---

### 1.IoC是什么

Ioc(Inversion of Control) 控制反转，这是一种设计思想，我的理解就是把
它分成两个部分控制和反转，控制的意思就是，我们用传统的java程序设计去开发的时候，
要自己去用new去创建对象，而Ioc则用容器去控制对象的创建，控制外部资源的获取。
还有反转的意思就是我们在传统的开发中，我们要在对象中主动控制去直接获取依赖对象，
在Ioc中，我们不需要自己去获取，而是由Ioc的容器去创建并注入依赖，对象只是被动的接受依赖。
这个获取依赖对象的责任反转给了Ioc容器。

### 2.装配Bean的三种方式

#### 2.1 自动化装配Bean

Spring从两个角度来实现自动化装配：

1) 组件扫描（component scanning）：Spring会扫描@Component注解的类，并会在应用上下文中为这个类创建一个bean。

2) 自动装配（autowiring）：Spring自动满足bean之间的依赖。

使用到的注解：

- @Component：表明这个类作为组件类，并告知Spring要为这个类创建bean。默认bean的id为第一个字母为小写类名，可以用value指定bean的id。
- @Configuration：代表这个类是配置类。
- @ComponentScan：启动组件扫描，默认会扫描所在包以及包下所有子包中带有@Component注解的类，并会在Spring容器中为其创建一个bean。可以用basePackages属性指定包。
- @RunWith(SpringJUnit4ClassRunner.class)：以便在测试开始时，自动创建Spring应用上下文。
- @ContextConfiguration：告诉在哪加载配置。
- @Autowired：将一个类的依赖bean装配进来。


```java
//播放器接口
public interface CdPlayer {
    void play();
}
```

```java
//唱片接口
public interface CDDisk {
    void sing();
}
```

```java
//播放器实现类
@Component
public class MediaPlayer implements CdPlayer{
    private CDDisk cd;

    @Autowired
    public MediaPlayer(CDDisk cd) {
        this.cd = cd;
    }

    @Override
    public void play() {
        cd.sing();
    }
}
```

```java
//唱片实现类
@Component
public class JJLIN implements CDDisk {
    private String title = "裂缝中的阳光";
    private String singer = "林俊杰";

    public void sing() {
        System.out.println(title);
        System.out.print(singer);
    }
}
```

```java
//配置类
@Configuration
@ComponentScan(basePackages = "cn.xkx.ssm.test")
public class CDplayConfig {
}
```
```java
//测试类
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = CDplayConfig.class)
public class CdPlayTest {
    @Autowired
    private CdPlayer cdPlayer;

    @Test
    public void test() {
        cdPlayer.play();
    }
}
```

![运行结果](/img/1553672536(1).jpg)


### 2.2 JavaConfig用java代码进行Bean的装配

优点：

可以实现基本数据类型的值、字符串字面量、类字面量等注入。
　　使用到的注解：

- @Bean：默认情况下配置后bean的id和注解的方法名一样，可以通过name属性自定义id。
- @ImportResourse：将指定的XML配置加载进来
- @Import：将指定的Java配置加载进来。

注：不会用到@Component与@Autowired注解。


```java
//播放器接口
public interface CdPlayer {
    void play();
}
```

```java
//唱片接口
public interface CDDisk {
    void sing();
}
```

```java
//播放器实现类
public class MediaPlayer implements CdPlayer{
    private CDDisk cd;

    public MediaPlayer(CDDisk cd) {
        this.cd = cd;
    }

    @Override
    public void play() {
        cd.sing();
    }
}
```

```java
//唱片实现类
public class JJLIN implements CDDisk {
    private String title;
    private String singer;
    private List<String> tracks;

    public JJLIN(String title, String singer, List<String> tracks) {
        this.title = title;
        this.singer = singer;
        this.tracks = tracks;
    }

    public void sing() {
        System.out.println(title);
        System.out.print(singer);
        for (String s : tracks) {
            System.out.println(s);
        }
    }
}
```

```java
//唱片配置类
@Configuration
public class CDDiskConfig {

    @Bean
    public CDDisk cdDiskConfig() {
        return new JJLIN("学不会","林俊杰",new ArrayList<String>(){{
            add("那些你很冒险的梦");
            add("不存在的情人");
        }});
    }
}
```

```java

//播放器配置类
@Configuration
public class CDplayConfig {

    @Bean
    public CdPlayer cdPlayerConfig(CDDisk cd) {
        return new MediaPlayer(cd);
    }
}
```

```java
//总配置类
@Configuration
@Import({CDplayConfig.class,CDDiskConfig.class})
public class AllConfig {
}
```

```java
//测试类
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = AllConfig.class)
public class CdPlayTest {
    @Autowired
    private CdPlayer cdPlayer;


    @Test
    public void test() {
        cdPlayer.play();
    }
}

```

运行结果：

![](/img/1553674236.jpg)

### 2.3 通过XML装配bean(不怎么用)

4.通过XML装配bean
优点：什么都能做。

缺点：配置繁琐。

使用到的标签：
- <bean>:将类装配为bean，也可以导入java配置。属性id是为bean指定id，class是导入的类。
- <constructor-arg>:构造器中声明DI，属性value是注入值，ref是注入对象引用。
- spring的c-命名空间：起着和<constructor-arg>相似的作用。
- <property>：设置属性，name是方法中参数名字，ref是注入的对象。
- Spring的p-命名空间：起着和<property>相似的作用。
- <import>：导入其他的XML配置。属性resource是导入XML配置的名称。





