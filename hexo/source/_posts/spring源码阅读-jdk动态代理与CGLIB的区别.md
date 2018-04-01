---
title: spring源码阅读-jdk动态代理与CGLIB的区别
date: 2018-02-04 23:09:11
tags: 
- spring
- java
---
## spring中实现动态代理的方式有两种,一种是使用jdk的原生接口,一种是使用CGLIB的技术.<!--more-->

### jdk的原生接口
1. 需要继承InvocationHandler接口, 代理类的方法调用都会转发至该接口的invoke方法,本质是基于java的反射机制
2. 被代理的对象需要继承接口

#### 一个简单的测试
```java
public class JdkDynamicTest implements InvocationHandler {

    private Object targetObject;

    public JdkDynamicTest(Object targetObject) {
        this.targetObject = targetObject;
    }

    public void before() {
        System.out.println("before.........");
    }

    public void after() {
        System.out.println("after..........");
    }

    public Object getProxy() {
        return Proxy.newProxyInstance(targetObject.getClass().getClassLoader(), targetObject.getClass().getInterfaces()
                , this);
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        before();
        Object result = method.invoke(targetObject, args);
        after();
        return result;
    }

    public static void main(String[] args) {
        HelloWorld helloWorld = new HelloWorldImpl();
        JdkDynamicTest jdkDynamicTest = new JdkDynamicTest(helloWorld);
        HelloWorld helloWorldProxy = (HelloWorld) jdkDynamicTest.getProxy();
        helloWorldProxy.print();
    }
}
```

### CGLIB技术
1. 需要单独引用CGLIB的依赖,是一个基于ASM的字节码生成库,因为原生的jdk代理方式不支持非接口继承的类,所以spring会切换至cglib方式.
2. cglib通过继承实现的动态代理,代理类是被代理类的子类,所以不需要继承接口
```java
public class CglibTest implements MethodInterceptor {

    private Object targetObject;

    public CglibTest(Object targetObject) {
        this.targetObject = targetObject;
    }

    public Object getProxy() {
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(targetObject.getClass()); //指定代理的目标对象,类似jdk的proxy
        enhancer.setCallback(this); //类似InvocationHandler的invoke方法,方法调用的中转
        return enhancer.create();
    }

    public void before() {
        System.out.println("before.........");
    }

    public void after() {
        System.out.println("after..........");
    }

    @Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        before();
        Object result = methodProxy.invokeSuper(o, objects);
        after();
        return result;
    }

    public static void main(String[] args) {
        HelloWorld helloWorld = new HelloWorldImpl();
        CglibTest cglibTest = new CglibTest(helloWorld);
        HelloWorld helloWorldProxy = (HelloWorld) cglibTest.getProxy();
        helloWorldProxy.print();
    }
}
```

### 问题与思考
因为cglib是继承父类来实现的代理,在java中final方法是不能重载的,所以这种情况cglib应该无法处理的.Object类中有的方法是final方法,我们可以通过打印代理类这些方法的输出看看.同理,final声明的类也是一样.
```java
@Override
   public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
       before();
       Object result = methodProxy.invokeSuper(o, objects);
       System.out.println(method.getName());
       after();
       return result;
   }
   public static void main(String[] args) {
    HelloWorld helloWorld = new HelloWorldImpl();
    CglibTest cglibTest = new CglibTest(helloWorld);
    HelloWorld helloWorldProxy = (HelloWorld) cglibTest.getProxy();
    helloWorldProxy.hashCode();
    helloWorldProxy.getClass();
}
\\输出
before.........
hashCode
after..........
```
