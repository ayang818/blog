---
title: servlet究竟是什么,servlet及子类源码分析
date: 2019-10-21 20:31:48
tags: [JavaWeb,源码]
---
## servlet是什么

好像在使用了SpringBoot后，或者说Spring后，程序员好像越来越难接触到servlet这个东西了，很多人以为隐藏在重重框架后的servlet是个超级复杂的洪水猛兽，但是并不是这样的。**事实上，servlet就是一个Java接口**，interface! 打开idea，ctrl + shift + n，搜索servlet，就可以看到是一个只有5个方法的interface!

```java
public interface Servlet {
    void init(ServletConfig var1) throws ServletException;

    ServletConfig getServletConfig();

    void service(ServletRequest var1, ServletResponse var2) throws ServletException, IOException;

    String getServletInfo();

    void destroy();
}
```

所谓接口，无非是为了规范。那么对于那些实现了servlet接口的类来说呢？他们就可以处理网络请求的吗？当然是不可以的！servlet并不与客户端直接打交道，真正和客户端打交道的东西是tomcat容器，tomcat监听了端口，每过来一个http请求，tomcat根据http请求的url，将请求转发到tomcat中的对应的servlet容器中，然后servlet处理业务逻辑后返回一个response对象，tomcat再将这个response返回给客户端。

<!-- more -->

## 框架中的servlet

其实对于框架的使用者来说，许多人其实完全不知道自己写的类，什么时候被谁调用了，你甚至没有看到这个类被new以下，但是类里面的代码就是执行了。你好像就是配了个.xml文件，然后按了下Run/Debug按钮，项目就跑了起来。

其实没什么复杂的，这些东西，抽象的讲其实就是——注入和回调，你自己的代码里或许没有一个显式入口，但是tomcat中有个main方法，假设是这样的

![img](https://pic3.zhimg.com/80/v2-ce6e39bb02e3c6a2f4eb1e5afaa6e4e6_hd.jpg)

![img](https://pic3.zhimg.com/80/v2-14c18b69b5fb642f8d56698d2df20171_hd.jpg)

![](https://pic4.zhimg.com/80/v2-d473a8662d758859e75c3f9afce9e982_hd.jpg)

我们只需要写个xml，框架机会解析xml创建实例，通过接口注入实例到合适的位置来执行我们的代码，其实并不很难懂。



## 实现一个Servlet

对于servlet接口来说，你可以发现你如果想直接实现Servlet接口，似乎很难，因为service()方法中的两个参数，一个是Request，Response。这两个东西怎么写嘛！前面说到servlet的请求其实是由tomcat传入的，所以其实tomcat会将将这两个请求封装成对象传给我们。我们往下看，servlet包中还有一个抽象类实现了servlet接口，叫做GenericServlet，但是点进源码一看，里面关于servlet接口的方法都是抽象的。所以再往下看，有一个叫做HttpServlet的抽象类继承了GenericServlet，里面也完善了servlet接口的函数。所以我们要实现一个servlet只需要继承HttpServlet方法就可以了。但是我们要重写HttpServlet里的方法也是有讲究的。不妨先看看里面service方法的源码

```java
protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        String method = req.getMethod();
        long lastModified;
        if (method.equals("GET")) {
            lastModified = this.getLastModified(req);
            if (lastModified == -1L) {
                this.doGet(req, resp);
            } else {
                long ifModifiedSince = req.getDateHeader("If-Modified-Since");
                if (ifModifiedSince < lastModified) {
                    this.maybeSetLastModified(resp, lastModified);
                    this.doGet(req, resp);
                } else {
                    resp.setStatus(304);
                }
            }
        } else if (method.equals("HEAD")) {
            lastModified = this.getLastModified(req);
            this.maybeSetLastModified(resp, lastModified);
            this.doHead(req, resp);
        } else if (method.equals("POST")) {
            this.doPost(req, resp);
        } else if (method.equals("PUT")) {
            this.doPut(req, resp);
        } else if (method.equals("DELETE")) {
            this.doDelete(req, resp);
        } else if (method.equals("OPTIONS")) {
            this.doOptions(req, resp);
        } else if (method.equals("TRACE")) {
            this.doTrace(req, resp);
        } else {
            String errMsg = lStrings.getString("http.method_not_implemented");
            Object[] errArgs = new Object[]{method};
            errMsg = MessageFormat.format(errMsg, errArgs);
            resp.sendError(501, errMsg);
        }

    }
```

可以看到他会根据你请求方式的不同，来调用不同的函数，拿get请求时候的方法举例子，doGet()方法的源码是这样的

```java
protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        String protocol = req.getProtocol();
        String msg = lStrings.getString("http.method_get_not_supported");
        if (protocol.endsWith("1.1")) {
            resp.sendError(405, msg);
        } else {
            resp.sendError(400, msg);
        }

    }
```

意思就是要是http/1.1的话，返回405，否则返回400，反正就是不管怎么样都是失败。但是这也正常啊，因为这段东西其实属于业务逻辑范畴，所以对于父类来说，父类无法知道业务逻辑，所以写一个模板方法，具体业务逻辑抽象成方法让子类来实现。这其实就是设计模式中的模板方法模式。

