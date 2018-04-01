---
title: spring-mvc之dispatcherServlet
date: 2018-04-02 00:06:59
tags: 
- spring
- java
---

spring-mvc是一个基于servlet-api的一个web框架，它是围绕前端控制器来设计的。spring-mvc核心是一个servlet,即dispatcherServlet，它对请求进行分发，并将请求交给请求映射、视图解析、异常处理等所需的组件来处理。

### dispatcherServlet的结构
![dispatcherServlet的结构](/images/img3.png)
因为其本身也是servlet，所以也是要实现servlet的规范的。GennericServlet和HttpServlet是java中的servlet实现，剩余的
HttpServletBean、FrameworkServlet和DispatcherServlet则是spring-mvc的了。其中继承有Aware和Capatable接口，在spring中Aware表示可以感知到，get到的意思。如果我们想要使用spring中的某一样东西，就可以通过继承实现XXXAware接口,spring就会通过setXXX()方法把这个资源给我们。在这里，DispatcherServlet继承了ApplicationContextAware和EnvironmentAware接口，可以使用spring中ApplicationContext和Environment。这两个分别是整个应用容器和对应的环境配置信息。<!--more-->



### HttpServletBean
我们知道servlet创建时候会调用其init()方法，我们通过看HttpServletBean的init()方法看下它是如何初始化的。
```java
@Override
public final void init() throws ServletException {

  // Set bean properties from init parameters.
  PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);
  if (!pvs.isEmpty()) {
    try {
      BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
      ResourceLoader resourceLoader = new ServletContextResourceLoader(getServletContext());
      bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, getEnvironment()));
      initBeanWrapper(bw);
      bw.setPropertyValues(pvs, true);
    }
    catch (BeansException ex) {
      throw ex;
    }
  }

  // Let subclasses do whatever initialization they like.
  initServletBean();
}
```
在HttpServletBean中两个属性
```java
private ConfigurableEnvironment environment;
private final Set<String> requiredProperties = new HashSet<>(4);
```
分别表示servlet的环境和必须参数。
HttpServletBean的init()方法把servlet中配置的参数封装到pvs中，然后通过 initServletBean();这行代码做初始化。
戳进去看这里的实现是空的，其实是一个模板方法，子类就是通过这个方法来实现初始化。

###
这里BeanWrapper是一个操作javaBean属性的方法。在这里就是代表其类本身。
PropertyValues是spring bean中的一个属性封装类，可以设置将Bean属性及value封装进去。
```java
public class BeanWrapperTest {
    public static void main(String[] args) {
        User user = new User();
        BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(user);
        PropertyValue propertyValue = new PropertyValue("name", "张三");
        bw.setPropertyValue(propertyValue);
        //输出张三
        System.out.println(user.getName());
    }
}

class User {
    private String name;
    private int age;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }
}
```

### FrameworkServlet
我知道FrameworkServlet的初始化就是initServletBean。它只干了两件事就是初始化webApplicationContext和它自己。
```java
protected final void initServletBean() throws ServletException {

		long startTime = System.currentTimeMillis();

		try {
			this.webApplicationContext = initWebApplicationContext();
			initFrameworkServlet();
		}
		catch (ServletException | RuntimeException ex) {
			this.logger.error("Context initialization failed", ex);
			throw ex;
		}
	}
```
```java
protected WebApplicationContext initWebApplicationContext() {
  WebApplicationContext rootContext =
      WebApplicationContextUtils.getWebApplicationContext(getServletContext());
  WebApplicationContext wac = null;

  if (this.webApplicationContext != null) {
    // 如果已经通过构造方法设置了它则使用这个context
    wac = this.webApplicationContext;
    if (wac instanceof ConfigurableWebApplicationContext) {
      ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
      if (!cwac.isActive()) {
        // The context has not yet been refreshed -> provide services such as
        // setting the parent context, setting the application context id, etc
        if (cwac.getParent() == null) {
          // The context instance was injected without an explicit parent -> set
          // the root application context (if any; may be null) as the parent
          cwac.setParent(rootContext);
        }
        configureAndRefreshWebApplicationContext(cwac);
      }
    }
  }
  if (wac == null) {
    // No context instance was injected at construction time -> see if one
    // has been registered in the servlet context. If one exists, it is assumed
    // that the parent context (if any) has already been set and that the
    // user has performed any initialization such as setting the context id
    wac = findWebApplicationContext();
  }
  if (wac == null) {
    // No context instance is defined for this servlet -> create a local one
    wac = createWebApplicationContext(rootContext);
  }

  if (!this.refreshEventReceived) {
    // Either the context is not a ConfigurableApplicationContext with refresh
    // support or the context injected at construction time had already been
    // refreshed -> trigger initial onRefresh manually here.
    onRefresh(wac);
  }

  if (this.publishContext) {
    // Publish the context as a servlet context attribute.
    String attrName = getServletContextAttributeName();
    getServletContext().setAttribute(attrName, wac);
    if (this.logger.isDebugEnabled()) {
      this.logger.debug("Published WebApplicationContext of servlet '" + getServletName() +
          "' as ServletContext attribute with name [" + attrName + "]");
    }
  }

  return wac;
}
```
这里的注释中说明了，webApplicationContext分别尝试从构造方法，是否已经在ServletContext中了寻找，如果存在则使用它。如果没有就创建一个，并且最后会根据publishContext来决定是否将webApplicationContext注入到ServletContext的属性中。ServletContext是代表的是整个web应用。其中重点在onRefresh方法，FrameworkServlet会根据监听到的刷新事件来执行。
这是个模板方法，子类继承它来执行属于自己的刷新应用的工作

### DispatcherServlet
DispatcherServlet作为一个前端控制器，只是对前台传过来的请求进行分发，具体怎么处理请求，怎么响应视图是交给别的组件完成。
所以onRefresh中可以看到其会初始化哪些组件。其实就是webApplicationContext.getBean()在容器中查找，如果我们在配置文件中定义有这些组件就会找到，没有则会默认创建。
```java
protected void onRefresh(ApplicationContext context) {
  initStrategies(context);
}
protected void initStrategies(ApplicationContext context) {
  initMultipartResolver(context);
  initLocaleResolver(context);
  initThemeResolver(context);
  initHandlerMappings(context);
  initHandlerAdapters(context);
  initHandlerExceptionResolvers(context);
  initRequestToViewNameTranslator(context);
  initViewResolvers(context);
  initFlashMapManager(context);
}
```

### 总结
分析和阅读了spring-mvc的创建过程，其初始化实际是通过HttpServletBean初始Servlet的环境配置，FrameworkServlet初始化
web容器,DispatcherServlet初始化相应mvc的组件，为之后整个容器工作提供支持。其中包含了许多设计的思想，结构虽然看起来不复杂，但内部的设计还是比较复杂的。

