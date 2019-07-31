---
title: Java原子性操作以及不如来手写一个简单互斥锁
date: 2019-07-16 22:21:04
tags: [Java,并发编程]
---
## 从i++问题的引入
在多线程编程中我们经常会看到这样一份代码<br>

实现的计数器代码
```java
public class Counter {
    volatile int i = 0;

    public void add() {
        i++;
    }

    public int getValue() {
        return this.i;
    }
}
```
<br>
<!-- more -->
六个线程下的实现的+1
```java
public class AyangCounterTest {
    public static void main(String[] args) throws InterruptedException{
        Counter counter = new Counter();
        for (int i = 0; i < 6; i++) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    for (int j = 0; j < 10000; j++) {
                        counter.add();            
                    }
                }
            }).start();
        }
        Thread.sleep(6000);
        System.out.println(counter.getValue());
    }
}
```
大家都知道这段代码的答案其实不是60000，打印出来的结果是不确定的，但是为什么呢？

---
## 原子性操作
我们可以看一下上面的代码的反编译出来的结果，来看看i++在多线程中到底发生了什么，运行
```
javap -p -v Counter.class
```
我们看到反编译出来的是这样的
```
 public void add();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=3, locals=1, args_size=1
         0: aload_0
         1: dup
         2: getfield      #2                  // Field i:I
         5: iconst_1
         6: iadd
         7: putfield      #2                  // Field i:I
        10: return
      LineNumberTable:
        line 8: 0
        line 9: 10
```
其中i++被反编译成了四条机器指令
```
2: getfield
5: iconst_1
6: iadd
7: putfield
```
&emsp;&emsp;我们先思考i变量在JVM内存中是怎么储存的，JVM分为共享的<font color=red>堆内存</font>这里就存放着counter这个计数器对象，也就是说i这个类变量就在这个共享的堆内存中。<br>
&emsp;&emsp;而每个线程又有它们的<font color=red>线程工作内存</font>，这个对于线程之间是私有的。我们接着继续看看这四条机器码<br>
getfield是先把i这个变量从公共堆内存中提到线程工作内存中的<font color=red>操作栈</font>中，iconst_1就是往操作栈中压入一个1，接着iadd就是将栈中的两个值弹出并相加，最后putfield就是将相加后的值从线程工作空间放回公共堆内存，这就完成了一次一个线程中的i++。所以我们试想多个线程同时进行i++的时候，出现了这样的情况，举个栗子
```
i = 0
线程A > 提取到A线程的工作区内存 > .... > 放回堆内存
                            线程B > 提取到B线程的工作区内存 > ..... > 放回堆内存
```
也就是说在一个线程获取i值但又没有来的及更新i值的时候，另一个线程又获取了i值，显然这个线程获取到的i值不是我们理想状况下的i值，这就导致了放回堆内存的时候导致重复而覆盖，这就是一个i++失效的一个过程。

也就是说我们在多线程的情况下并不能这么直接写i++，我们需要<font color=blue>原子性操作</font>来帮助我们达成目的。

---
## 原子性操作
原子性操作的核心特征其实就是，把一次操作视为一个整体，而资源在这次操作中保持一致！<br>
而原子性操作很重要的一个知识点就是CAS
### CAS（Compare and Swap）
顾名思义，CAS的意思就是比较加交换，这个属于硬件同步用语，处理器提供了基本内存操作的原子性保证。
CAS操作需要输入两个值，期望操作的旧数据的值，以及一定操作后的值。在CAS操作期间，他会对输入旧值和旧值的原始引用内存地址的偏移量(OffSet)计算出的内存地址中的值进行对比，若没有发生变化，才会交换成新值。CAS返回一个布尔类型，若是交换失败，则进行自旋，重新进行一次CAS操纵，知道交换成功为止。
对于以上的int操作，java提供了一个类sun.misc.Unsafe，其中就有conpareAndSwapInt()方法实现了CAS，但是这个类无法直接使用，你需要通过反射得到这个类的实例。

---
## 不如自己来实现一把锁
其实许多锁的机制也就是通过CAS作为基础实现的
我自己也写了一把简单的AyangLock锁QAQ，基于一个线程队列，加上原子性引用实现，代码如下
```java
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicReference;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.LockSupport;

/**
 * JameLock
 */
public class AyangLock implements Lock {
    AtomicReference<Thread> owner = new AtomicReference<>();
    private LinkedBlockingQueue<Thread> waiter = new LinkedBlockingQueue<>();

    @Override
    public boolean tryLock() {
        // if (owner.get() == null) {
        // owner = waiter.
        // }
        // return false;
        return owner.compareAndSet(null, Thread.currentThread());
    }

    @Override
    public void lock() {
        if (!tryLock()) {
            waiter.offer(Thread.currentThread());
            // 自旋
            for (;;) {
                Thread head = waiter.peek();
                if (head == Thread.currentThread()) {
                    if (!tryLock()) {
                        // 挂起线程
                        // wait / notify 需要配合synchronized在这里不可行
                        LockSupport.park();
                    } else {
                        // 抢锁成功
                        waiter.poll();
                        return;
                    }
                } else {
                    // 线程挂起
                    LockSupport.park();
                }
            }
        }
    }

    public boolean tryUnlock() {
        if (owner.get() != Thread.currentThread()) {
            throw new IllegalMonitorStateException();
        } else {
            return owner.compareAndSet(Thread.currentThread(), null);
        }
    }

    @Override
    public void unlock() {
        if (tryUnlock()) {
            Thread th = waiter.peek();
            if (th != null) {
                // unpark是唤醒线程
                LockSupport.unpark(th);
            }
        }
    }

    @Override
    public void lockInterruptibly() throws InterruptedException {

    }

    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return false;
    }

    @Override
    public Condition newCondition() {
        return null;
    }
}
```
好了，现在试着将i++使用我的AyangLock加锁
```java
import java.util.concurrent.locks.Lock;

/**
 * Main
 */
public class AyangCounterTest {
    public static void main(String[] args) throws InterruptedException{
        Lock ayangLock = new AyangLock();
        Counter counter = new Counter();
        for (int i = 0; i < 6; i++) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    for (int j = 0; j < 10000; j++) {
                        ayangLock.lock();
                        counter.add();   
                        ayangLock.unlock();         
                    }
                }
            }).start();
        }
        Thread.sleep(6000);
        System.out.println(counter.getValue());
    }
}
```
答案就是60000啦！




