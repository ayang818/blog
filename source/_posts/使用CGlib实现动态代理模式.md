---
title: 使用CGlib实现动态代理模式
date: 2019-11-14 18:05:25
tags: [设计模式]

---

## 前言
关于代理模式和使用JDK dynamic proxy实现的动态代理模式，已经在[使用JDK dynamic proxy实现动态代理模式](https://ayang818.gitee.io/blog/)写过了，也说了Spring5之前使用的代理就是JDK默认的动态代理。然后我自己最近也在写一个框架——[Agito](https://github.com/ayang818/Agito)。在自己实现AOP特性的时候，感觉使用JDK自带的动态代理并不容易实现我的需求，所以在技术选型的时候，我选择了CGlib这个字节码增强利器来实现动态代理。

<!-- more -->
## 封装一个CGlib proxy
使用的话，CGlib功能比JDK自带的要更强，JDK自带的动态代理只可以对实现了某个接口的对象进行增强。但是CGlib层级相当于方法拦截。还是举一个例子。
对于一个没有实现接口的类，有一个say()方法
```java
public class HelloImpl  {
    public void say() {
        System.out.println("Hello world");
    }
}
```
然后有一个CGlibProxy实现了CGlib中的MethodInterceptor接口，然后重写了intercept方法，这个和JDK自带的很像
```java
public class CGlibProxy implements MethodInterceptor {

    private static CGlibProxy cGlibProxy = new CGlibProxy();

    public static CGlibProxy getInstance() {
        return cGlibProxy;
    }

    @Override
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
        before();
        Object result = methodProxy.invokeSuper(obj, args);
        after();
        return result;
    }

    private void after() {
        System.out.println("finished");
    }

    private void before() {
        System.out.println("start");
    }

    @SuppressWarnings("unchecked")
    public <T> T getProxy(Class<T> cls) {
        return (T) Enhancer.create(cls, this);
    }
}
```

然后用泛型封装一下调用时候的方法getProxy，然后在内部封装一个单例(线程不安全)，使用一个main调用一下

```java
public static void main(String[] args) {
        HelloImpl enhancedHello = CGlibProxy.getInstance().getProxy(HelloImpl.class);
        enhancedHello.say();
}
```

得到的结果就是

```bash
start
Hello world
finished
```

这就是CGlib的最小实践。之后在框架实现AOP特性的时候，也会高强度用到动态代理，到时候可以补充一些具体的例子。