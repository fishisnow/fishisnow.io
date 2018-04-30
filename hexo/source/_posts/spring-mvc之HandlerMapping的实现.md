---
title: spring-mvc之HandlerMapping的实现
date: 2018-05-01 02:38:49
tags:
- spring
- java
---
### 什么是handlerMapping
spring mvc中handlerMapping用来通过request找到相应处理器的handler和Interceptor.它的接口十分简单，
只有getHandler(HttpServletRequest request)方法，返回HandlerExecutionChain，HandlerExecutionChain主要有
两个属性，一个页面控制器（handler）和拦截器HandlerInterceptor组成的list。<!--more-->

### handlerMapping类结构图  
![HandlerMapping的子类的继承结构](/images/img4.png)
AbstractHandlerMapping是HandlerMapping的抽象实现，所有的handlerMapping都继承于AbstractHandlerMapping,这里使用了模板模式设计了HandlerMapping的实现.  
HandlerMapping实现分为两个分支，一个是AbstractUrlHandlerMapping，另一个是AbstractHandlerMethodMapping。AbstractUrlHandlerMapping主要是通过url来映射控制器，而AbstractHandlerMethodMapping是将Method来作为控制器的，我们通常在方法中配置@RequestMapping来控制请求处理就是通过这种形式实现的。
AbstractHandlerMapping继承了Ordered和WebApplicationObjectSupport，handlerMapping是有顺序的，映射控制器的时候会根据顺序的优先级来处理。后者则是提供了获取容器ServletContext的能力。
![获取handler调用逻辑时序图](/images/img5.png)

### AbstractHandlerMethodMapping
AbstractHandlerMethodMapping中有个一个内部类MappingRegistry，其中部分属性是
```java
private final Map<T, MappingRegistration<T>> registry = new HashMap<T, MappingRegistration<T>>();

private final Map<T, HandlerMethod> mappingLookup = new LinkedHashMap<T, HandlerMethod>();

private final MultiValueMap<String, T> urlLookup = new LinkedMultiValueMap<String, T>();

private final Map<String, List<HandlerMethod>> nameLookup =
    new ConcurrentHashMap<String, List<HandlerMethod>>();
```
这里的T在RequestMappingInfoHandlerMapping是RequestMappingInfo类，是用来存放匹配handler的条件的类。
在AbstractHandlerMethodMapping初始化时，会从容器中拿所有是handler类型的bean,然后去获取这个bean的方法，
将方法用来匹配映射handler的条件封装成一个RequestMappingInfo,并注册到map中。这个map在lookupHandlerMethod会用到。
```java
//伪代码
initHandlerMethods() {
  ApplicationContext context = getApplicationContext();
  String[] beanNames = context.getBeanNamesForType(Object.class);; //从容器中获取所有的bean
  for(String beanName: beanNames) {
    //遍历
    Class<?> beanType = context.getType(beanName);
    //isHandler由子类RequestMappingInfoHandlerMapping实现，判断是否有＠Controller和＠RequestMapping注解
    if(beanType != null && isHandler(beanType)) {
        //获取这个bean所有符合handler要求的方法，已经它对应的匹配条件组成的k-v形式的map
        Map<Method, RequestMappingInfo> methods = selectMethods(beanType);

        for() {
          //分别将这些bean和对应mehods注册到MappingRegistry中
          //这个handler就是beanName
          registerHandlerMethod(handler, invocableMethod, mapping)
        }
    }
  }
}
lookupHandlerMethod(String lookupPath, HttpServletRequest request) {
  List<Match> matches = new ArrayList<Match>();
  //根据传进来的请求，从MappingRegistry中查找是否有跟当前url匹配的HandlerMethod
  List<T> directPathMatches = this.mappingRegistry.getMappingsByUrl(lookupPath);
  //如果有　加入到matches中
  if(directPathMatches != null) {
    addMatchingMappings(directPathMatches, matches, request);
  }

  //...如果没有与当前url匹配　再根据匹配条件遍历

  // matches不为空时　排序取匹配度最高的
  if(matches.isNotEmpty()) {
    Collections.sort(matches, comparator);
    Match bestMatch = matches.get(0);
    return bestMatch.handlerMethod;
  } else {
    return null;
  }
}
```

### 总结
HandlerMapping的功能就是根据request找到对应的handler和Interceptor, 组成HandlerExecutionChain类返回。
寻找handler的方法通过模板方法getHandlerInternal交给了子类去实现，其他的整体结构都在AbstractHandlerMapping中完成。我们常用的
在Controller中声明@RequestMapping注解来实现请求的映射是由AbstractHandlerMethodMapping子类来实现的。
