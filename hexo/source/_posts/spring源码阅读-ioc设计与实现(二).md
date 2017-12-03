---
title: spring源码阅读-ioc设计与实现（二）
date: 2017-12-03 20:00:43
tags:
---
### 从异常栈说起
我们已经知道spring ioc容器初始的顶层核心接口是BeanFactory，而其最终的默认实现类是DefaultListableBeanFactory．
我们可以从spring项目中初始化容器的时候，获取bean报的异常来理解spring获取单例bean的流程．
![spring初始化容器获取bean失败报错](/images/img1.1.img)
<!--more-->

### ioc获取单例的bean
从异常栈截图中看出，单例模式的实现在support包下DefaultSingletonBeanRegistry类中