---
title: spring源码阅读-ioc设计与实现(一)
date: 2017-11-20 23:47:58
tags: [spring],[java]
---

### spring核心包
  spring的核心包包含beans,context,core,SpEl，其中beans包是spring对bean的设计和管理的实现，spring的核心思想ioc就
在这个包中。

### spring ioc
spring ipc的基本意思是控制反转，java通过new来实例化的一个对象, 通常情况下这个对象的生命周期十分简单，但是在spring容器中，bean的生命周期更加灵活，更加便于管理。spring核心包中也是围绕着ioc的设计和实现，以及如何管理bean的生命周期来实现的。
<!--more-->

### spring beans包结构
1. annotation spring的bean的注解支持包
2. factory spring的ioc容器的实现包
3. support beans包的支持包，比如BeanDefinition的构建类，读取类等。

### BaseBeanFactory
BeanFactory是一个工厂类的接口，提供了bean的生命周期的标准接口。
其中factory下的xml是一个基于xml配置文件加载bean的实现。
通常我们在开发spring web项目时，都是通过xml来配置和管理的spring的bean的，通过根据xml注入bean的方式来理解spring在
bean的注入，销毁，生命周期管理时都使用哪些接口和设计模式。

#### BeanFactory中如何存储和管理bean的
BeanDefinitionRegistry bean的注册器，BeanFactory接口的实现类继承实现
registerBeanDefinition(String beanName, BeanDefinition beanDefinition)方法来实现bean的注册。
继承并实现这个方法的bean工厂类是DefaultListableBeanFactory。
这个工厂类中Map<String, BeanDefinition> beanDefinitionMap来存放Bean，key是对应的bean的名字。每一次
在spring中初始化的bean都会放在这个map中。
spring BeanFactory初始化的伪代码
```
    //new 一个工厂类
    DefaultListableBeanFactory factory = new DefaultListableBeanFactory();

    //读取xml配置文件的资源
    new XmlBeanDefinitionReader(factory).loadBeanDefinitions(
            new ClassPathResource("test.xml", getClass()));

    //将bean解析成spring内部的BeanDefinition结构，传入的class可以通过反射来创建对象
    BeanDefinition bd = new BeanDefinition(TestBean.class);
    //向容器注册bean
    factory.registerBeanDefinition("bd", bd);

    // 获取bean
    TestBean bean = (TestBean) factory.getBean("bd");

```
