---
title: java反射总结
date: 2018-03-04 23:10:37
tags: [java]
---
在框架的源码中经常会用到java反射机制，工厂模式和代理模式都离不开java反射，总结和理解下反射。
## 反射的作用
1. 在运行时判断任意一个对象所属的类；
2. 在运行时获取类的对象；
3. 在运行时访问java对象的属性，方法，构造方法等<!--more-->

## 为什么会有反射
### 灵活实现面向对象
反射可以在运行时确定对象类型，访问对象的方法，属性等，这是动态编译的优点，灵活，降低类的耦合。
啥是动态编译呢，我的理解是java是先编译后运行的，在程序运行时代码已经时固定了，如果没有在代码中new一个对象，那么可以通过类名找到class文件，并加载进内存，用这个字节码文件来创建对象，这就是动态编译，区别与静态编译就是不是程序预先设定好的，而是在运行过程中确定对象类型。
比如我们知道与mysql数据库建立连接，如果不用反射，我们可能会这样写来获取数据驱动
```java
if(isMysql) {
  Connection con = new MysqlConnection();  
} else if(isOracle) {
  Connection con = new OracleConnection();
}
```
这样写的话，如果切换数据库从mysql切到oracel或其他数据库，我们都要修改下代码再编译项目重启服务，可想很麻烦。
有了反射我们就可以通过 Connection con = Class.forName("")来加载驱动，然后再用驱动连接。这样切换数据库只需要修改配置文件就可以了。

### 动态操作对象的属性和方法
比如我们可以通过反射来实现java类方法的远程调用
测试code，看下通过反射可以从字节码中读取类的方法，属性内容是什么。
```java
public class A {

    public void sayHello() {
        System.out.println("hello world1");
    }

    public void sayHello(String str) {
        System.out.println("hello world2");
    }

    public void sayHello(List<String> stringList) {
        System.out.println("hello world3");
    }
}
package reflect;

public class B {
    public static void main(String[] args) throws Exception {
        Class clazz = Class.forName("reflect.A");
        Object a = clazz.newInstance();
        //通过反射获取对象的方法
        Method[] methods = clazz.getDeclaredMethods();
        for(int i=0;i<methods.length;i++) {
            System.out.println(methods[i].getName());
            Class[] parameterTypes = methods[i].getParameterTypes();
            for(Class parameterType : parameterTypes) {
                System.out.println(parameterType.getName());
                System.out.println(parameterType.getClass());
            }
        }
    }
}
// 输出内容：
// sayHello
// java.util.List
// class java.lang.Class
// sayHello
// sayHello
// java.lang.String
// class java.lang.Class

```
通过反射我们可以知道一个类有那些声明的方法及方法的参数，重载的方法可以根据参数来区分。
所以我们可以这样来调用A类的方法。
```java
public class B {
    public static void main(String[] args) throws Exception {
        B b = new B();
        b.callMethod("reflect.A", "sayHello", null);
        b.callMethod("reflect.A", "sayHello", "hello");
        List<String> stringList = new ArrayList<>();
        stringList.add("hello");
        b.callMethod("reflect.A", "sayHello", stringList);
    }

    public void callMethod(String clazzName, String methodName, Object args) throws Exception {
        Class clazz = Class.forName(clazzName);
        Object o = clazz.newInstance();
        //通过反射获取对象的方法
        Method[] methods = clazz.getDeclaredMethods();
        for (int i = 0; i < methods.length; i++) {
            if (methods[i].getName().equals(methodName)) {
                Class[] parameterTypes = methods[i].getParameterTypes();
                if (parameterTypes != null && parameterTypes.length > 0) {
                    Class parameterType = parameterTypes[0];
                    if (parameterType.equals(String.class) && args instanceof String) {
                        methods[i].invoke(o, args);
                    } else if (parameterType.equals(List.class) && args instanceof List) {
                        methods[i].invoke(o, args);
                    }
                } else if (args == null) {
                    methods[i].invoke(o);
                }
            }
        }
    }
}
//输出：
// hello world1
// hello world2
// hello world3
```
很明显，我们是知道调用的时候的Ａ对象方法的参数是哪些才这样写，那假如我们不知道调用的参数有哪些，我们也不确定反射的类是谁的时候，又如何通过反射
去执行对象的方法呢？我们可以在客户端调用的时候将类的参数和对应class类型传进去，或者在反射类方法的时候将传进去的参数类型一个个比对。
```java
public class B {
    public static void main(String[] args) throws Exception {
        B b = new B();
        List<Object> argList = new ArrayList<>();
        b.callMethod("reflect.A", "sayHello", null);
        argList.add("hello");
        b.callMethod("reflect.A", "sayHello", argList);
        argList.clear();
        List<String> stringList = new ArrayList<>();
        stringList.add("hello");
        argList.add(stringList);
        b.callMethod("reflect.A", "sayHello", argList);
    }

    public void callMethod(String clazzName, String methodName, List<Object> argsList) throws Exception {
        //这里要求客户端传的List元素的顺序是严格与反射的方法一致
        Class clazz = Class.forName(clazzName);
        Object o = clazz.newInstance();
        //通过反射获取对象的方法
        Method[] methods = clazz.getDeclaredMethods();
        for (int i = 0; i < methods.length; i++) {
            if (methods[i].getName().equals(methodName)) {
                Class[] parameterTypes = methods[i].getParameterTypes();
                if (parameterTypes != null && parameterTypes.length > 0) {
                    if (argsList == null) {
                        continue;
                    }
                    if (parameterTypes.length != argsList.size()) {
                        continue;
                    }
                    boolean isCorrectArgs = true;
                    for (int j = 0; j < argsList.size(); j++) {
                        Object arg = argsList.get(j);
                        if (parameterTypes[j] == List.class) {
                          　// 如果是List类型
                            if (arg instanceof List) {
                            } else {
                                isCorrectArgs = false;
                                break;
                            }
                        } else {
                            if (arg.getClass() != parameterTypes[j]) {
                                isCorrectArgs = false;
                                break;
                            }
                        }
                    }
                    if (isCorrectArgs) {
                        methods[i].invoke(o, argsList.toArray());
                    }
                } else if (argsList == null) {
                    methods[i].invoke(o);
                }
            }
        }
    }
}
//输出：
// hello world1
// hello world2
// hello world3

```

### 总结和思考
反射给程序提供了一种上帝视觉的模式，大大提高了程序开发的灵活性和扩展能力，但是反射是一种解释操作，
跟直接new创建对象还是有差别的，滥用反射会造成性能问题。


