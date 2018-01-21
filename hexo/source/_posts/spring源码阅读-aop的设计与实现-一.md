---
title: spring源码阅读-aop的设计与实现(一)
date: 2018-01-21 20:58:34
tags: spring,java
---
## spring的面向切面的思想
　　在spring中许多功能的设计使用了AOP的方式，如事务的管理，通过声明＠transactional元数据来切换不同的数据源，
理解AOP的设计与实现更有利于我们日常对spring相关功能的使用和问题解决．
## jdk提供的代理实现
　　AOP其实是通过代理类来执行对被代理类执行切面的方法，通过执行方法中加入切面拦截的逻辑和处理，来实现面向切面编程．JDK
提供了生成对象的动态代理功能．<!--more-->

|类名|说明|
|:---|:--|
|Proxy|Object newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h) 方法返回一个代理对象|
|InvocationaHandler|Object invoke(Object proxy, Method method,Object[] args) 方法实现代理对象中方法的调用和处理|

## AOP思想涉及的元素
  说白了就是拦截什么对象，在哪里做的拦截，拦截后做什么．在Aspectj(一个面向切面的框架)对其进行了声明和定义．

|类名|说明|
|---:|--:|
|PointcutAdvisor|切点通知器，通知哪个对象的哪个方法进行拦截|
|Pointcut|切点对象，在AOP中有连接点的概念，这个连接点可以理解为程序的某个执行过程，比如调用方法，执行方法等，Pointcut就是对这些连接点做定义|
|Advisor|通知器对象，对应spring中的前置通知，后置通知，环切通知，也就是被拦截方法调用前，调用后等，通过通知器通知代理类执行拦截方法|
这里面的设计思想具体看Aspectj的源码，这里我们主要讲spring怎么运用的．

## spring中AOP是怎么嵌入的
  我们知道spring的bean是基于IOC自动注入的，也就是说要对某个bean实现切面编程，就得在BeanFactory中getBean()方法修改其返回的bean．
前面提到的BeanFactory.getBean->doCreateBean->initializeBean()中提到BeanPostProcessor类．这个类就是用来在初始化bean的时候对这个bean
进行前置和后置的一些操作．
```
Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
  Object wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
  invokeInitMethods(beanName, wrappedBean, mbd);
  wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
  \\.....
  return wrappedBean;
}

public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
    throws BeansException {

  Object result = existingBean;
  for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {
    Object current = beanProcessor.postProcessAfterInitialization(result, beanName);
    if (current == null) {
      return result;
    }
    result = current;
  }
  return result;
}
```
从代码我们可以看出，我们只要找出AOP声明的BeanPostProcessor抽象类就可以找到IOC植入AOP的方法．它就是AbstractAdvisingBeanPostProcessor类
我们看下它继承的BeanPostProcessor的postProcessAfterInitialization的实现．（这里为了清晰，只是简单写了其逻辑）
```
    if(bean instanceof AopInfrastructureBean || bean instanceof Advised || this.Advisor == null) {
      //如果是AOP的拦截器类或者通知器类，则返回．
      return;
    }

		if (isEligible(bean, beanName)) {
      //找到生成这个bean代理的工厂类，接着看看这个类有没有合理的接口方法，如果有则使用JDK生成动态代理对象，
      //没有则直接通过目标对象的class字节码来生成代理对象（也就是CGLIB的那一套）
			ProxyFactory proxyFactory = prepareProxyFactory(bean, beanName);
			if (!proxyFactory.isProxyTargetClass()) {
				evaluateProxyInterfaces(bean.getClass(), proxyFactory);
			}
      //向拦截器链中增加本拦截器
			proxyFactory.addAdvisor(this.advisor);
			customizeProxyFactory(proxyFactory);
      //通过JDK动态代理或CGLIB创建代理对象
			return proxyFactory.getProxy(getProxyClassLoader());
		}

```
创建代理对象后，spring的使用者执行方法时其实就是拿代理对象的方法去执行了．这样AOP就植入了spring中容器中了．

## 总结
　　本文主要是梳理了下AOP在spring中是如何使用和植入的，方便我们进一步理解AOP的设计与实现．下一文将重点学习关于Spring如何利用AspectJ的定义来设计其面向切面的功能的．

