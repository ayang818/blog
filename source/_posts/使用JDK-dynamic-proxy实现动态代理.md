---
title: 使用JDK dynamic proxy实现动态代理模式
date: 2019-11-12 20:57:34
tags: [设计模式]
---
## 代理模式
代理模式理解起来很简单，比如说我有一个接口叫做Hello
```java
public interface Hello {
    void say();
}
```
他的实现类是HelloImpl
```java
public class HelloImpl implements Hello {
    @Override
    public void say() {
        System.out.println("hello world");
    }
}
```
但是我又想在say()调用前做些什么，这里其实很关键，因为你找到了一个**切点**，这就是AOP——面向方面编程的特征，这个待会再说。想加新的功能，但是我们都学过开闭原则，所以我们打算再创建一个代理类来实现这一功能(组合)
```java
public class HelloProxy implements Hello {
    private Hello hello;

    public HelloProxy() {
        hello = new HelloImpl();
    }

    @Override
    public void say() {
        System.out.println("start");
        hello.say();
        System.out.println("end");
    }
}
```
然后运行试下
```java
public static void main(String[] args) {
    Hello hello = new HelloProxy();
    hello.say();
}
```
结果就是
```
start
hello world
end
```
这样看着既简单，又实现了需要的功能，似乎已经满足我们的需求了。这种方式叫做静态代理。
<!-- more -->
## 用JDK dynamic proxy实现的动态代理
动态代理也是代理模式的一种实现，我们的两个基础类Hello和HelloImpl都不变，然后使用JDK给我们提供的动态代理方式来实现一下，我们创建一个ProxyHanlder 然后实现invocation接口。
```java
public class ProxyHandler implements InvocationHandler {
    // 被代理对象
    private Object target;

    public ProxyHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("start");
        Object result = method.invoke(target, args);
        System.out.println("end");
        return result;
    }
}
```
然后运行
```java
public static void main(String[] args) {
        // 被代理的对象
        Hello hello = new HelloImpl();
        ProxyHandler proxyHandler = new ProxyHandler();
        Hello helloProxy = (Hello) Proxy.newProxyInstance(hello.getClass().getClassLoader(), hello.getClass().getInterfaces(), proxyHandler);
        helloProxy.say();
    }
```
我们需要使用Proxy.newInstance方法获取代理对象，但是可以发现需要传入的参数太长太恶心了，你需要传入被代理对象的类加载器，接口表，和一个invocation实现类(在这里就是ProxyHandler)。所以其实可以再ProxyHandler封装一个bind方法。
```java
@SuppressWarnings("unchecked")
public <T> T bind(T target) {
    this.target = target;
    // 返回一个被代理对象
    return (T) Proxy.newProxyInstance(target.getClass().getClassLoader(), target.getClass().getInterfaces(), this);
}
```
然后运行
```java
public static void main(String[] args) {
    // 被代理的对象
    Hello hello = new HelloImpl();
    ProxyHandler proxyHandler = new ProxyHandler();
    Hello helloProxy = proxyHandler.bind(hello);
    helloProxy.say();
}
```
结果还是一样的。JDK动态代理实现比较复杂，之后的文章再解析源码。

## 对比动态代理和静态代理
对比一下动态代理和静态代理，这么看着可能会觉得动态代理好复杂，还不如用静态代理呢。但是我们思考下要往Hello接口中加一个新的方法时，静态代理会牵涉到什么。
- 首先得改接口
- 其次要改实现类
- 最后要改代理类

要是加的功能多了，就是一次更新风暴，反之对比动态代理。
动态代理只需要改接口和实现类，而我们的代理类是稳定不变的。这已经是很大优势了。Spring5前的中的AOP就是基于JDK的动态代理实现的。