---
title: spring-mvc之HandlerAdapter适配器模式
date: 2018-05-27 23:35:00
tags:
- spring
- java
---
## 什么是适配器设计模式
四人帮的<设计模式>是这样说的,将一个类的接口转换成客户希望的另外一个接口。 Adapter模式使得原本由于接口不兼容
而不能一起工作的那些类可以一起工作,也叫包装器Wrapper.<!--more-->

## spring mvc中的适配器模式
在dispatcherServlet处理请求的过程中,大概逻辑是这样的.
```java
  Handler handler = getHandler(request);
  handlerAdapter adapter = getHandlerAdapter(handler.getHandler());
  ModelAndView mv = adapter.handle(request, response, handler.getHandler());
```
spring的SimpleControllerHandlerAdapter中代码比较好理解这个适配器模式
```java
@Override
public boolean supports(Object handler) {
  return (handler instanceof Controller);
}

@Override
public ModelAndView handleRender(RenderRequest request, RenderResponse response, Object handler)
    throws Exception {

  return ((Controller) handler).handleRenderRequest(request, response);
}
```
handler是spring的处理器,但是处理请求真正干活的却是handlerAdapter,handlerAdapter遍历当前的所有的adapter通过
supports方法找到可以处理当前处理器的适配器,然后调用handler的handleRenderRequest方法.
可以理解为什么要使用handlerAdapter, 因为spring对处理器没有任何限制,因为处理器有可能是一个类(比如一个Controller)或者一个方法(handlerMethod),
处理器极大的灵活使得它无法取兼容dispatcherServlet的一个统一调用,试想如果dispatcherServlet需要写if/else
来判断这个处理器是一个Controller还是一个servlet还是一个特殊的方法,那就显得很啰嗦,也不好扩展.如果我将来的处理器不再是Controller而是别的呢.
handlerAdapter则将处理器做了包装,对dispatcherServlet提供了一个统一使用处理器而不用管处理器实现细节的方式.

### 开闭原则
开闭原则明确的告诉我们：软件实现应该对扩展开放，对修改关闭，也就是我们可以改变模块的功能。对模块行为进行扩展时，不必改动模块的源代码或者二进制代码.(百科)

# handlerAdapter的主要方法
|bean属性|描述|
|-|-|
|boolean supports(Object handler)|判断能否使用某个handler|
|ModelAndView handleRender(RenderRequest request, RenderResponse response, Object handler)|具体使用handler干活,返回统一视图对象|
