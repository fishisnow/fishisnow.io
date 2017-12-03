---
title: spring源码阅读-ioc设计与实现（二）
date: 2017-12-03 20:00:43
tags:
---
### 从异常栈说起
我们已经知道spring ioc容器初始的顶层核心接口是BeanFactory，而其最终的默认实现类是DefaultListableBeanFactory．
我们可以从spring项目中初始化容器的时候，获取bean报的异常来理解spring获取单例bean的流程．
![spring初始化容器获取bean失败报错](/images/img1.1.png)
<!--more-->

### ioc获取单例的bean
从异常栈截图中看出，单例模式的实现在support包下DefaultSingletonBeanRegistry类中
DefaultSingletonBeanRegistry继承SingletonBeanRegistry接口
![SingletonBeanRegistry接口](/images/img1.2.png)

#### 注册单例的方法

```
public void registerSingleton(String beanName, Object singletonObject) throws IllegalStateException {
    synchronized (this.singletonObjects) {
        Object oldObject = this.singletonObjects.get(beanName);
        if (oldObject != null) {
            throw new IllegalStateException("Could not register object [" + singletonObject +
                    "] under bean name '" + beanName + "': there is already object [" + oldObject + "] bound");
        }
        addSingleton(beanName, singletonObject);
    }
}
	
```
```
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    Object singletonObject = this.singletonObjects.get(beanName);
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        synchronized (this.singletonObjects) {
            singletonObject = this.earlySingletonObjects.get(beanName);
            if (singletonObject == null && allowEarlyReference) {
                ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                if (singletonFactory != null) {
                    singletonObject = singletonFactory.getObject();
                    this.earlySingletonObjects.put(beanName, singletonObject);
                    this.singletonFactories.remove(beanName);
                }
            }
        }
    }
    return singletonObject;
}
```
DefaultSingletonBeanRegistry中维护了一个线程安全的缓存和非线程安全的缓存来存放这些单实例的bean,这个map就是singletonObjects和earlySingletonObjects．
注册中心注册bean实例的时候会从单例对象map中查找，如果没有则添加到对应的单例对象map和缓存中，确保容器中只有一个单实例．

#### 添加实例的方法
```
protected void addSingleton(String beanName, Object singletonObject) {
    synchronized (this.singletonObjects) {
        this.singletonObjects.put(beanName, singletonObject);
        this.singletonFactories.remove(beanName);
        this.earlySingletonObjects.remove(beanName);
        this.registeredSingletons.add(beanName);
    }
}
```
其中的sigletonFactory是单例模式的工厂实现，如果获取一个单实例，先从singletonObjects中查找，如果没有再从缓存的earlySingletonObject中查找
，如果仍然找不到对应的单实例，则通过该单实例的制造工厂创建一个．
从获取实例的过程中可以看出，earlySingletonObject是专门存放通过单例工厂创建的单例的．这个缓存不是线程安全的．

### spring的单例工厂类的实现
从异常栈中可以看出AbstractBeanFactory的AbstractAutowireCapableBeanFactory模板类中通过了单例工厂来创建单例
doGetBean方法的代码片段
```
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

createBean的伪代码
```
RootBeanDefinition mbdToUse = mbd;
获取类的class
Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
//声明一个bean描述类
mbdToUse = new RootBeanDefinition(mbd);
mbdToUse.setBeanClass(resolvedClass);
//获取对应对象Class类型
Class<?> targetType = determineTargetType(beanName, mbd);
bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);

//....构造bean

protected Object applyBeanPostProcessorsBeforeInstantiation(Class<?> beanClass, String beanName) {
    for (BeanPostProcessor bp : getBeanPostProcessors()) {
        if (bp instanceof InstantiationAwareBeanPostProcessor) {
            InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
            Object result = ibp.postProcessBeforeInstantiation(beanClass, beanName);
            if (result != null) {
                return result;
            }
        }
    }
    return null;
}

```
RootBeanDefinition mbd就是继承BeanDefinition的实现类，这个接口描述bean的结构，对应XML中的bean定义，
可以看出spring是通过单例工厂创建单例时是通过InstantiationAwareBeanPostProcessor或者说BeanProcessor的回调
来返回对应实例的．这个接口其实是bean实例化的后置处理类，其返回的对象会代理原来实例的bean，这个不做探讨．

### 总结
从源码中可以看出，spring容器中实现的单例和我们通常情况下理解的设计模式中的单例是不一样，设计模式中的单例模式是将构造方法私有，在
整个应用中只有一个实例．而spring容器的单例是在整个容器中只有一个实例，由于ioc的控制反转，bean实例的生成和销毁都交给了spring容器，
本质上只是控制反射生成类实例的时候不在容器中重复创建，不通过私有构造方法便可以达到单例的效果也就不难理解了．
