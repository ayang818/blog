---
title: Vector和ArrayList的线程安全性&&浅析ArrayList源码
date: 2019-08-31 19:43:08
tags: [并发编程,Java]
---
## 一个例子出发
下面的一个例子展示了ArrayList的非线程安全，以及ArrayList的错误检查机制
<!-- more -->
```java
public class VectorSimpleTest {
    public static void main(String[] args) throws NoSuchMethodException, InvocationTargetException, IllegalAccessException {

//        Vector<Integer> list = new Vector<>();

        ArrayList<Integer> list = new ArrayList<>();
        for (int i = 0; i < 100; i++) {
            list.add(i);
        }
        new Thread(() -> {
            while (!list.isEmpty()) {
                Integer index = list.size() - 1;
                Integer integer = list.get(index);
                System.out.println("Read"+integer);
                System.out.println(list);
            }
        }).start();
        new Thread(() -> {
            while (!list.isEmpty()) {
                Integer index = list.size() - 1;
                System.out.println("Remove"+list.get(index));
                list.remove(index);
            }
        }).start();
        
        Method checkForComodification = ArrayList.class.getDeclaredMethod("checkForComodification");
        checkForComodification.setAccessible(true);
        checkForComodification.invoke(list);
        
    }
}
```

这段代码会报一个错误
```
ConcurrentModificationException
```
，就是说在修改这个实例的时候出现了意外，错误的原因很简单，就是多个线程在一个arrayList对象上工作没有同步。这个意外是由
```
ArrayList$Itr.checkForComodification
```
抛出的，这个方法的源码如下

```java
final void checkForComodification() {
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
}
```

可以关注modCount这个变量，这个变量继承于AbstractList中

```java
protected transient int modCount = 0;
```

就名字来看，我们就可以知道，这个方法其实就是用来检查ArrayList更改次数的，就好像一个简单的版本管理工具，他那里有一个实例给你备份了正确的版本历史(Github远程仓库)，但是有一次你提交了一个错误的版本(本地版本push到远端)，arrayList发现了你提交的版本号和他那里记录的版本历史不一致，就会给你抛异常。我们通过查看这个方法的调用就可以知道

```java
public E next()
public void remove()
public void forEachRemaining()
public E previous()
public void set()
public void add()
```
以上的方法都会调用一次检查。解决的办法其实也很简单，只要把ArrayList换成Vector就可以了，Vector在这种场景下是可以维持好他本身的原子性的，因为vector内置的会让modCount++的方法都是加了synchronized关键字的。synchronized会对vector的Class加一把互斥锁来保证数据的同步。但是使用了Vector就一定可以保证数据同步吗？

## 同步容器Vector的非线程安全因素
```java
public class SafeContainer {
//    private static ArrayList<Integer> vector = new ArrayList<>();
    private static Vector<Integer> vector = new Vector<>();
    private static SafeContainer safeContainer = new SafeContainer();

    public static SafeContainer safeContainerFactory() {
        vector.add(1);
        vector.add(2);
        return safeContainer;
    }

    public  Integer changeLast() {
        int lastIndex = vector.size() - 1;
        System.out.println(vector.get(lastIndex));
        Integer remove = vector.remove(lastIndex);
        return remove;
    }

    public  boolean isEmpty() {
        return vector.isEmpty();
    }

    public  Integer deleteLast() {
        int size = vector.size() - 1;
        System.out.println(vector.get(size));
        Integer integer = vector.get(size);
        return integer;
    }

    public static void main(String[] args) {
        SafeContainer safeContainer = SafeContainer.safeContainerFactory();
        new Thread(() -> {
            while (!safeContainer.isEmpty()) {
                System.out.println(safeContainer.deleteLast());
            }
        }).start();
        new Thread(() -> {
            while (!safeContainer.isEmpty()) {
                System.out.println(safeContainer.changeLast());
            }
        }).start();
    }
}
```
以上的代码使用了工厂模式对Vector进行了封装，并封装了两个方法，看起来这个类使用了Vector也是线程安全的，但是运行起来却有概率报
```
ArrayIndexOutOfBoundsException
```
，其实是因为safeContainer在使用getLast获取最后一位值的时候，又调用了deleteLast，于是vector的size就又减了1，所以导致下标逸出。这其实就是因为synchronized的作用域仅限于vector对象，无法对SafeContainer类进行同步。这个机制就需要我们自行加锁，修改代码

```java
public class SafeContainer {
//    private static ArrayList<Integer> vector = new ArrayList<>();
    private static Vector<Integer> vector = new Vector<>();
    private static SafeContainer safeContainer = new SafeContainer();

    public static SafeContainer safeContainerFactory() {
        vector.add(1);
        vector.add(2);
        return safeContainer;
    }

    public synchronized Integer deleteLast() {
        int lastIndex = vector.size() - 1;
        System.out.println(vector.get(lastIndex));
        Integer remove = vector.remove(lastIndex);
        return remove;
    }

    public synchronized boolean isEmpty() {
        return vector.isEmpty();
    }

    public synchronized Integer getLast() {
        int size = vector.size() - 1;
        System.out.println(vector.get(size));
        Integer integer = vector.get(size);
        return integer;
    }

    public synchronized static void main(String[] args) {
        SafeContainer safeContainer = SafeContainer.safeContainerFactory();
        new Thread(() -> {
            while (!safeContainer.isEmpty()) {
                System.out.println(safeContainer.getLast());
            }
        }).start();
        new Thread(() -> {
            while (!safeContainer.isEmpty()) {
                System.out.println(safeContainer.deleteLast());
            }
        }).start();
    }
}

```
这样无论是ArrayList还是Vector都可以在这段代码里正常同步运行了。