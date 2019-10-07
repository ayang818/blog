---

title: Leetcode的几道有意思的多线程题

date: 2019-10-7 20:17:24

tags: [并发编程,Java]

---

## 前记

最近一直在看Java的多线程编程,但苦于这东西有点过于偏向理论,结果今天上了Leetcode上看了下,发现居然有多线程相关的题目，(虽然只有五题),顺便放个链接吧，[题目链接](https://leetcode-cn.com/problemset/concurrency/)。题目感觉就是让你使用一些同步工具类或者锁或其他机制来完成要求，挺有意思的.

<!--more-->

## [1114. 按序打印](https://leetcode-cn.com/problems/print-in-order/)

这个我的做法就是直接使用两个CountDownLatch, 根据运行顺序一层层解开不同的CountDownLatch, 代码如下

```java
class Foo {
       
    private CountDownLatch latchSecond = new CountDownLatch(2);
    private CountDownLatch latchThird = new CountDownLatch(3);
    public Foo() {
        
    }

    public void first(Runnable printFirst) throws InterruptedException {
        // printFirst.run() outputs "first". Do not change or remove this line.
        printFirst.run();
        latchSecond.countDown();
        latchThird.countDown();
    }

    public void second(Runnable printSecond) throws InterruptedException {
        latchSecond.countDown();
        latchSecond.await();
        // printSecond.run() outputs "second". Do not change or remove this line.
        printSecond.run();
        latchThird.countDown();
    }

    public void third(Runnable printThird) throws InterruptedException {
        latchThird.countDown();
        latchThird.await();
        // printThird.run() outputs "third". Do not change or remove this line.
        printThird.run();
    }
}
```



## [1115. 交替打印FooBar](https://leetcode-cn.com/problems/print-foobar-alternately/)

这题题意是让我们在多线程下轮流打印"foo"和"bar"两个字符串, 所以分析一下两个线程其实都只有一种运作方式,就是各自轮流打印"foo"或"bar", 就好像红绿灯一样(没有黄灯), 这个时候我的想法一开始是继续用直接用一个volatile变量做是否标记, 这样还可以做到无锁. 但是不知道为啥Leetcode一直显示超时, emmm 所以后来我换了信号量Semaphore来解决, 代码如下

```java
class FooBar {
    private int n;
    private final Semaphore semFoo = new Semaphore(1);
    private final Semaphore semBar = new Semaphore(0);

    private volatile boolean flag = true;
    
    public FooBar(int n) {
        this.n = n;
    }

    public void foo(Runnable printFoo) throws InterruptedException {
        
        for (int i = 0; i < n; i++) {
            semFoo.acquire();
            // printFoo.run() outputs "foo". Do not change or remove this line.
            printFoo.run();
            semBar.release();
        }
    }

    public void bar(Runnable printBar) throws InterruptedException {
        
        for (int i = 0; i < n; i++) {
            semBar.acquire();
            // printBar.run() outputs "bar". Do not change or remove this line.
        	printBar.run();
            semFoo.release();
        }
    }
}
```



## [1116. 打印零与奇偶数](https://leetcode-cn.com/problems/print-zero-even-odd/)

这道题和上道题超级类似, 只是中间多了个0, 这个就更像我们平时红绿灯的场景了, 0就像黄灯, 交替出现在红灯绿灯之间. 思路一样, 也是通过Semaphore解决, 代码如下

```java
class ZeroEvenOdd {
    private int n;
    private Semaphore semZero = new Semaphore(1);
    private Semaphore semEven = new Semaphore(0);
    private Semaphore semOdd = new Semaphore(0);
    private int evenTimes;
    private int oddTimes;

    public ZeroEvenOdd(int n) {
        this.n = n;
        evenTimes = n/2;
        oddTimes = n%2 == 0 ? n/2 : n/2+1;
    }

    // printNumber.accept(x) outputs "x", where x is an integer.
    public void zero(IntConsumer printNumber) throws InterruptedException {
        for (int i = 1; i <= n; i++) {
            semZero.acquire();
            printNumber.accept(0);
            if (i%2 == 1) {
                semOdd.release();
            } else {
                semEven.release();
            }
        }
    }

    public void even(IntConsumer printNumber) throws InterruptedException {
        for(int i = 0, j = 2; i < evenTimes; i++,j+=2) {
            semEven.acquire();
            printNumber.accept(j);
            semZero.release();
        }
    }

    public void odd(IntConsumer printNumber) throws InterruptedException {
        for (int i = 0,j = 1; i < oddTimes; i++,j+=2) {
            semOdd.acquire();
            printNumber.accept(j);
            semZero.release();
        }
    }
}
```

## [1195. 交替打印字符串](https://leetcode-cn.com/problems/fizz-buzz-multithreaded/)

题目的意思就是让我们接受一个数字，然后从1遍历到这个数字，看被三整除还是被五整除，还是被三被五都整除。然后按需输出，有多个线程对其进行读取，所以要保证输出顺序，思路也是使用Semaphore对其进行阻塞操作，代码如下(又臭又长)，
```java
class FizzBuzz {
    private int n;
    private volatile int start = 1;
    private Semaphore semFizzBuzz = new Semaphore(1);
    private Semaphore semFizz = new Semaphore(0);
    private Semaphore semBuzz = new Semaphore(0);
    private Semaphore semNone = new Semaphore(0);


    public FizzBuzz(int n) {
        this.n = n;
    }

    // printFizz.run() outputs "fizz".
    public void fizz(Runnable printFizz) throws InterruptedException {
        while(true) {
            semFizz.acquire();
            if (start > n) {
                semBuzz.release();
                semNone.release();
                semFizzBuzz.release();
                break;
            }
            if (start%3 == 0) {
                printFizz.run();
                start++;
                semFizzBuzz.release();
            } else {
                semBuzz.release();
            }
        }
    }

    // printBuzz.run() outputs "buzz".
    public void buzz(Runnable printBuzz) throws InterruptedException {
        while(true) {
            semBuzz.acquire();
            if (start > n) {
                semFizz.release();
                semNone.release();
                semFizzBuzz.release();
                break;
            }
            if (start%5 == 0) {
                printBuzz.run();
                start++;
                semFizzBuzz.release();
            } else {
                semNone.release();
            }
        }
    }

    // printFizzBuzz.run() outputs "fizzbuzz".
    public void fizzbuzz(Runnable printFizzBuzz) throws InterruptedException {
        while(true) {
            semFizzBuzz.acquire();
            if (start > n) {
                semFizz.release();
                semBuzz.release();
                semNone.release();
                break;
            }
            if (start%3 == 0 && start%5 == 0) {
                printFizzBuzz.run();
                if (start == n) {
                    start++;
                    semFizz.release();
                    semBuzz.release();
                    semNone.release();
                    break;
                }
                start++;
                semFizzBuzz.release();
            } else {
                semFizz.release();
            }
        }
    }

    // printNumber.accept(x) outputs "x", where x is an integer.
    public void number(IntConsumer printNumber) throws InterruptedException {
        while(true) {
            semNone.acquire();
            if (start > n) {
                semFizz.release();
                semBuzz.release();
                semFizzBuzz.release();
                break;
            }
            printNumber.accept(start);
            start++;
            semFizzBuzz.release();
        }
    }
}
```
还有刚刚想出来的一种思路是无锁结构，由于每个数字在四个线程中其实只会执行一次，所以只要使用volatile保证可见性即可，代码如下
```java
class FizzBuzz {
    private int n;
    private volatile int index = 1;

    public FizzBuzz(int n) {
        this.n = n;
    }

    // printFizz.run() outputs "fizz".
    public void fizz(Runnable printFizz) throws InterruptedException {
        for (; index <= n ; ) {
            if (index%3 == 0 && index%5!=0 && index <= n) {
                printFizz.run();
                index ++;
            }
        }
    }

    // printBuzz.run() outputs "buzz".
    public void buzz(Runnable printBuzz) throws InterruptedException {
        for (;index <= n;) {   
            if (index%5 == 0 && index%3 != 0 && index <= n) {
                printBuzz.run();
                index ++;
            }
        }
    }

    // printFizzBuzz.run() outputs "fizzbuzz".
    public void fizzbuzz(Runnable printFizzBuzz) throws InterruptedException {
        for (; index <= n;) {
            if (index%5 == 0 && index%3 == 0 && index <= n) {
                printFizzBuzz.run();
                index ++;
            }
        }
    }

    // printNumber.accept(x) outputs "x", where x is an integer.
    public void number(IntConsumer printNumber) throws InterruptedException {
        for (;index <= n; ) {
            if (index%5 != 0 && index%3 != 0 && index <= n) {
                printNumber.accept(index);
                    index ++;
            } 
        }
    }
}
```

## [1117. H2O 生成](https://leetcode-cn.com/problems/building-h2o/)
这道题目的难度居然是困难，但是感觉做起来挺简单的，用Semaphore或者ObjectMonitor都可以做，放两种代码
- ObjectMonitor
```java
class H2O {
    private Object lock = new Object();
    private volatile int hNum = 0;

    public H2O() {
        
    }

    public void hydrogen(Runnable releaseHydrogen) throws InterruptedException {
		synchronized(lock) {
            while (hNum >= 2) {
                lock.wait();
            }
            releaseHydrogen.run();
            hNum ++;
            lock.notifyAll();
        }
        // releaseHydrogen.run() outputs "H". Do not change or remove this line.
    }

    public void oxygen(Runnable releaseOxygen) throws InterruptedException {
        synchronized (lock) {
            while (hNum < 2) {
                lock.wait();
            }
		    releaseOxygen.run();
            hNum = 0;
            lock.notifyAll();
        }
        // releaseOxygen.run() outputs "H". Do not change or remove this line.
    }
}
```

- Semaphore
```java
class H2O {
    private Semaphore o = new Semaphore(0);
    private Semaphore h = new Semaphore(2);

    public H2O() {
        
    }

    public void hydrogen(Runnable releaseHydrogen) throws InterruptedException {
		h.acquire(1);
        // releaseHydrogen.run() outputs "H". Do not change or remove this line.
        releaseHydrogen.run();
        o.release(1);
    }

    public void oxygen(Runnable releaseOxygen) throws InterruptedException {
        o.acquire(2);
        // releaseOxygen.run() outputs "H". Do not change or remove this line.
		releaseOxygen.run();
        h.release(2);
    }
}
```