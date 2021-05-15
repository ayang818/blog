---
title: python 装饰器从走路到跑路（误
date: 2021-04-19 13:28:30
tags: [python]
---

## 计时器的例子

在平时工作中，我们可能需要为一些业务方法添加一些通用的功能，比如 日志打点、调用耗时、重试机制等。举个例子，我们要统计某些方法的调用耗时并上报，那么对于一个业务方法，

```python
import time

def biz_method(uid):
    print('processing')
    time.sleep(1) # 模拟业务代码运行
```

我们可能想到这么改造

```python
import time
def biz_method():
    start = time.time()
    print('processing')
    time.sleep(1) # 模拟业务代码运行
    print ('time spent %.4fs', % (time.time() - start))
```

但是这样的话，一大段业务无关的代码就混入了业务流程中，这显然很不合理，并且我们需要记录统计时间的方法也不止这一个，这个时候，我们就需要 **装饰器** 这个东东

<!-- more -->
python 的装饰器的用法类似于 java 的注解，使用装饰器改造上面的代码后，变成了

```python
import time

def time_spent_count(func):
    def wrapper():
        start = time.time()
        func()
        print ('time spent %.4fs' % (time.time() - start))
    return wrapper

@time_spent_count
def biz_method():
    print('processing')
    time.sleep(1) # 模拟业务代码运行

biz_method()

# 输出
# processing
# time spent 1.0033s
```
这样我们就可以对业务代码无侵入地增加一个计时功能。

## 带参数的计时器
这个时候，我们的产品有给我们提了一个需求：部分接口需要把调用耗时上传到统计平台，而不走 ELK。
抽象成代码，不就是多加一个是否上传的参数嘛~
```python
import time

def time_spent_count(report=False):
    def wrapper(func):
        def inner_wrap():
            start = time.time()
            func()
            cost = time.time() - start
            print ('time spent %.4fs' % cost)
            if report:
                report()

        def report():
            print("report to remote~")
        return inner_wrap
    return wrapper

@time_spent_count(True)
def biz_method():
    print('processing')
    time.sleep(1) # 模拟业务代码运行

biz_method()
```
这里我们带参数的计时器就写好了，那么这里有人可能就要问了，<del>马老师，你这个不好用....</del>，为啥带参数的计时器比原来多了一个内部函数呀；原先的计时器是 time_spent_count -> wrapper。怎么到了你这里就变成了 time_spent_count -> wrapper -> inner_wrap 了。

其实装饰器的就是干了下面这么一件事情：接收一个函数，返回给你一个新函数
我们把装饰器的皮扒了吧 <del>害怕</del>
```py
# 第一个例子中不带参的计时器等价于下面
import time

def time_spent_count(func):
    def wrapper():
        start = time.time()
        func()
        print ('time spent %.4fs' % (time.time() - start))
    # 把 wrapper 函数返回
    return wrapper

def biz_method():
    print('processing')
    time.sleep(1) # 模拟业务代码运行

biz_method = time_spent_count(biz_method) # 这里的这个biz_method 实际上就变成了 wrapper 函数了
biz_method()
```

```py
# 第而个例子中带参的计时器等价于下面
import time

def time_spent_count(report=False):
    def wrapper(func):
        def inner_wrap():
            start = time.time()
            func()
            cost = time.time() - start
            print ('time spent %.4fs' % cost)
            if report:
                report()

        def report():
            print("report to remote~")
        return inner_wrap
    return wrapper

def biz_method():
    print('processing')
    time.sleep(1) # 模拟业务代码运行

biz_method = time_spent_count(report=True)(biz_method)
biz_method()
```


## 带参数的业务方法
为了让有些看到这里还没懂得同学有作业抄，这里定义了一个平时可能最常用的形式：装饰器带参，业务方法也带参的例子
```py
import time
import functools

def time_spent_count(report=False):
    def wrapper(func):
        @functools.wraps(func)
        def inner_wrap(*args, **kwargs):
            start = time.time()
            func(*args, **kwargs)
            cost = time.time() - start
            print ('time spent %.4fs' % cost)
            if report:
                report()

        def report():
            print("report to remote~")
        return inner_wrap
    return wrapper

@time_spent_count(True)
def biz_method(data):
    print('processing %s'% data)
    time.sleep(1) # 模拟业务代码运行

biz_method("hahah")
```

有的同学可能又要问了，不对呀，你这个突然冒出来的 @functools.wraps 又是个啥呀，这个装饰器其实对于业务的执行结果来说没有影响。

使用这个装饰器的原因是，装饰器其实返回了包含业务方法功能的另外一个函数，也就是说原有函数被狸猫换太子了耶~，原有函数的一些原信息就被丢失了，使用 functools.wraps 可以 **保留原函数的元信息。**

你可以分别去掉 time_spent_count 这个装饰器的 functools.wraps 测试一下

``` py
# 不使用 functools.wraps
print(biz_method.__name__)
>> inner_wrapper

# 使用 functools.wraps
print(biz_method.__name__)
>> biz_method
```


> 看到这里，你总该可以说你知道装饰器是个啥了吧，好耶~