---
title: Java对象的发布与逸出&&线程封闭机制
date: 2019-08-30 16:35:15
tags: [Java,并发编程]
---

## Java对象的发布与逸出
发布一个对象的意思是指，是对象能够在当前作用域外的代码中使用。比如Unsafe包中没有暴露一个对外的构造器，它使用的是现在类内部创建一个对象的引用，然后return 回这个引用的方式。
```java
private static final Unsafe theUnsafe = new Unsafe();

@CallerSensitive
public static Unsafe getUnsafe() {
    Class<?> caller = Reflection.getCallerClass();
    if (!VM.isSystemDomainLoader(caller.getClassLoader()))
        throw new SecurityException("Unsafe");
    return theUnsafe;
}
```

<!-- more -->
### 内部的可变状态逸出(不要这么做)
但是其实发布一个对象的时候，在该对象的非私有域的引用的所有对象都会同样被发布，比如下面的代码
```java
class UnsafeState {
    private String[] states = new String[] {"11","22"};

    public String[] getStates() {return states;}
}
```
发布的String[]对象中引用的String也一起被发布了。这样就会出现问题，因为任何调用者都可以修改这个数组的内容。这样就会导致误用的风险。

### 隐式的this引用逸出
下面代码展示了在使用构造器的时候，对象还没有发布，this就已经逸出了的栗子
```java
public class ThisEscape {
    public ThisEscape() {
        source.registerListener(
            new EventListener() {
                public void onEvent(Event e){
                    // 这里引用了ThisEscape类的引用
                    dosomething(e)
                }
            }
        )
    }
}
```

### 避免this引用逸出
使用工厂模式可以解决这个问题
```java
public class SafeListener {
    private final EventListener listener;

    private SafeListener() {
        listener = new EventListener() {
            public void onEvent(Event e){
                // 这里引用了ThisEscape类的引用
                dosomething(e);
            }
        }
    }

    public static SafeListener ListenserFactory() {
        SafeListener safe = new SafeListener();
        source.registerListener(safe.listener);
        return safe;
    }
}
```


## 线程封闭

### Ad-hoc 线程封闭
Ad-hoc线程封闭是指维护线程封闭性的职责完全由程序实现，其实这种封闭非常脆弱的。不管是volatile还是局部变量，其实都无法将某个属性封闭在某一个目标线程上。

### 栈封闭
我把它理解为线程操作栈封闭，当多个线程访问一个方法时，线程会拷贝基本类型的局部变量，到县城自己的操作栈中，由于不会涉及到类中的全局变量，所以线程之间不会互相影响。(我不清楚是不是会引用非基本类型的局部变量，我对此打算找时间进一步探索一下)

### ThreadLocal类
维持线程封闭性的更规范方法时使用ThreadLocal，这个类能使线程中的某个值与保存值的对象相关联，ThreadLock提供了get和set等访问接口，这些访问方法与每个使用该变量的线程都存有一份独立的副本，因此get方法得到的总是在当前线程调用set的最新值。<br>
下面的例子演示了通过把JDBC的连接保存到ThreadLocal对象中，每个创建的线程都会有自己的连接，这就起到了一个简单的连接池的作用
```java
package ThreadClose;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.SQLException;
import java.util.Properties;

public class ThreadLocalTest {

    private static String DB_URL = "<你自己的数据库URL>";
    private static String USERNAME = "<数据库用户名>";
    private static String PASSWORD = "<数据库密码>";
    private static Properties dataMap = new Properties();
    // 使用ThreadLocal维持线程封闭性
    private static ThreadLocal<Connection> connectHolder = new ThreadLocal<>() {
        // 重写initialValue，设置初始值
        @Override
        public Connection initialValue() {
            try {
                dataMap.setProperty("user", USERNAME);
                dataMap.setProperty("password", PASSWORD);
                return DriverManager.getConnection(DB_URL, dataMap);
            } catch (SQLException e) {
                e.printStackTrace();
            }
            return null;
        }
    };

    public static Connection getConnection() {
        // get方法到这个线程的Value
        Connection connection = connectHolder.get();
        return connection;
    }
}
```
为了验证不同线程得到的Connection不同，我又写了如下代码
```java
package ThreadClose;

import java.sql.Connection;
import java.sql.SQLException;

public class ThreadLocalVerify {
    public static void main(String[] args) throws InterruptedException, SQLException {
        Connection connection = ThreadLocalTest.getConnection();
        System.out.println(System.identityHashCode(connection));
        Thread.sleep(2000);
        new Thread(() -> {
            Connection connection1 = ThreadLocalTest.getConnection();
            System.out.println(System.identityHashCode(connection1));
        }).start();
    }
}
```