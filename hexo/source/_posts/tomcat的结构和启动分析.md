---
title: tomcat的结构和启动分析
date: 2018-03-18 14:50:04
tags: 
- tomcat
- java
---
tomcat是作为一个轻量的web服务器，是很多应用开发和部署的首选。

### tomcat结构
tomcat作为一个服务器，最顶层的容器是server，用来代表服务器。Server中包含service，service用来提供服务。
一个server可以拥有多个service，service中包含Connector和Container。Connector用来处理连接，提供socket,创建request和response请求
Container用来封装servlet，对request请求的具体处理。<!--more-->
### tomcat的四个主要配置文件
tomcat的配置文件主要有context.xml、web.xml、server.xml、tomcat-users.xml这四个，context是Tomcat的公共环境配置。web.xml是应用程序描述文件，webApp中的web.xml就继承于这个文件。server.xml则是对tomcat server的配置，设置端口和虚拟机等。
从server.xml配置我们可以大体看出tomcat的体系结构。
![tomcat体系结构](/images/img2.png)

```
<Server>                                                //顶层容器
    <Service>                                           //顶层Service，可包含一个Engine，多个Connecter
        <Connector>                                     //连接器类元素，代表通信接口
                <Engine>                                //容器类元素，为特定的Service组件处理客户请求，要包含多个Host
                        <Host>                          //容器类元素，为特定的虚拟主机组件处理客户请求，可包含多个Context
                                <Context>               //容器类元素，为特定的Web应用处理所有的客户请求
                                </Context>
                        </Host>
                </Engine>
        </Connector>
    </Service>
</Server>
```

### catalina
tomcat中的server是通过一个叫catalina类来进行管理的。catalina类中有load,start.stop方法对tomcat的生命周期进行管理。
实际上，catalina启动tomcat是通过调用server的start方法来启动tomcat的。
```java
\\(部分代码)
public void start() {

    if (getServer() == null) {
        //当前是否已经创建了server，如果没有调用load方法初始化一个
        load();
    }

    // Start the new server
    try {
        getServer().start();
    } catch (LifecycleException e) {
        log.fatal(sm.getString("catalina.serverStartFail"), e);
        try {
            getServer().destroy();
        } catch (LifecycleException e1) {
            log.debug("destroy() failed for failed Server ", e1);
        }
        return;
    }
}
```

### tomcat的启动入口
catalina类负责管理tomcat的生命周期，但是tomcat启动入口不是在catalina类，是一个叫Bootstrap的类。正常启动tomcat执行的startup.sh实际是执行
Bootstrap的main方法。
```java
\\部分代码
public static void main(String args[]) {

      if (daemon == null) {
          // Don't set daemon until init() has completed
          Bootstrap bootstrap = new Bootstrap();
          try {
              bootstrap.init();
          } catch (Throwable t) {
              handleThrowable(t);
              t.printStackTrace();
              return;
          }
          daemon = bootstrap;
      } else {
          Thread.currentThread().setContextClassLoader(daemon.catalinaLoader);
      }
}
```
init方法的伪代码
```java
public void init() {
     //加载类加载器
     ClassLoader catalinaLoader = initClassLoaders();
     //
     Thread.currentThread().setContextClassLoader(catalinaLoader);

     Class<?> startupClass =
             catalinaLoader.loadClass
                     ("org.apache.catalina.startup.Catalina");
     // 反射获的Catalina对象
     Object startupInstance = startupClass.newInstance();
     // 启动Server
     Method method =
             startupInstance.getClass().getMethod("start", null);
     method.invoke(startupInstance, null);
 }
```
其实是创建Bootstrap对象并执行了init()方法。init方法则初始化ClassLoader并用ClassLoarder创建Catalina实例。这的Catalina实例是通过反射
来创建，这样启动入口类和具体的管理类就分开了，Bootstrap就像一个适配器，可以灵活创建多种启动的方式，不得不说反射在java中无处不在。

