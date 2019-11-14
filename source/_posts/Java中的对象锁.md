---

title: Java中的对象锁(Monitor)

date: 2019-10-04 20:17:24

tags: [Java,并发编程]

---

## Java中的对象锁

说到对象锁，要与之对比就不得不讲到类锁。在写单例模式的时候其实经常会用在双重校验锁中用到类锁，如下面的代码

```java
public ClassLock {
    priavte volatiled ClassLock classLock;
    
    private ClassLock() {
        // init
    } 
    
    public ClassLock getInstance() {
        if (classLock ==  null) {
            synchronized (ClassLock.class) {
                if (classLock == null) {
                    classLock = new ClassLock();
                }
            }
        }
        return classLock;
    }
}
```

可以看到在上面的代码中我们是对一整个类进行加锁，包括直接在方法签名上使用synchronized关键字，这都是属于类锁的范畴。因为在调用一个类的实例的时候，当运行其中一个加锁的方法时，其他加锁的方法是不可以执行的。也就是说一个类其实只有一个类锁，它锁住的对象是一个类。但是对象锁不一样，一个类中可以有多个对象，也就是说对象锁在一个类的实例中不止一个, 而每个对象都可以成为一个Monitor。对象锁往往用于多个线程之间的协作，与之配套的工具是synchronized，wait，notify，notifyAll。

<!-- more -->

## 各自作用

- synchronized作用(这个其实已经说烂了，这里不多写)
  - 多用于多线程间的同步加锁操作
- wait()的作用和注意点
  - 不同于Thread.currentThread.wait()的作用，这个wait()方法属于Object中的内容，同样notify，notifyAll也都是Object的内容。
  - wait()总是在一个可以跳出的循环中被调用，调用后它会一直挂起获取当前对象锁的线程，并释放对象锁。调用wait的线程也会被添加到对象的等待队列中。wait调用会一直持续到被同一个对象的notifyAll被调用唤醒或notify随机唤醒此线程。
  - wait()必须在synchronized同步块中使用，因为要使用对象的wait的方法，就必须先获得对象锁
- notifyAll()与notify()
  - notify就是随机选择对象等待队列中的某个线程，并将其添加到入口队列，处于入口队列中的线程就会竞争对象锁，获取对象锁的线程就会继续运行。
  - notifyAll()会添加所有等待队列中的线程到入口队列。相比来说notifyAll更加常用。



## 例子

下面的例子来自[LeetCode 1117. H2O 生成](https://leetcode-cn.com/problems/building-h2o/)，题意可以点开来自己看题面，我使用Semaphore和对象锁都实现了一遍，然后模拟了整个过程——多个线程调用一个对象中的两个方法，一个方法打印H，一个方法打印O，两个方法中使用对象锁进行协调通信，达到输出HHO……的效果。代码如下

```java
package Lock;

public class ObjectLock {
    private static Runnable releaseO = () -> {
        System.out.print("O");
    };
    private static Runnable releaseH = () -> {
        System.out.print("H");
    };
    private static H2O h2O = new H2O();
    private static String inputStr = "OOHHHHOHH";

    public static void main(String[] args) {
        for (int i = 0; i < inputStr.length(); i++) {
            if ("O".equals(Character.toString(inputStr.charAt(i)))) {
                new Thread(() -> {
                    try {
                        h2O.oxygen(releaseO);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }).start();
            } else if ("H".equals(Character.toString(inputStr.charAt(i)))) {
                new Thread(() -> {
                    try {
                        h2O.hydrogen(releaseH);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }).start();
            }
        }
    }

}

class H2O {

    private static final Object LOCK = new Object();

    private int HCounter = 0;

    H2O() {

    }

    void hydrogen(Runnable releaseHydrogen) throws InterruptedException {
        synchronized(LOCK)
        {
            while(HCounter >= 2)
            {
                LOCK.wait();
            }
            releaseHydrogen.run();
            this.HCounter++;
            LOCK.notifyAll();
        }

    }

    void oxygen(Runnable releaseOxygen) throws InterruptedException {
        synchronized(LOCK)
        {
            while(HCounter < 2)
            {
                LOCK.wait();
            }
            releaseOxygen.run();
            this.HCounter = 0;
            LOCK.notifyAll();
        }

    }
}
```



