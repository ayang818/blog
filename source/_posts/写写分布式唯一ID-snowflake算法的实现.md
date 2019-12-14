---
title: '浅析分布式唯一ID生成算法——snowflake的实现'
date: 2019-12-14 20:29:25
tags: [分布式理论]
---
> 开头先放一个snowflake的实现——[仓库地址](https://github.com/ayang818/snowflake-id-generator/tree/master)
## 分布式唯一Id是什么
分布式唯一Id其实是一个分布式系统里面的一个不可或缺的元素，他一般具有如下几个特性

- 同一业务的全局唯一性(globally unique)
- 包含时间信息(timestamp)
- 递增有序性(incremented and ordered)

## 分布式唯一Id有什么用
这样一个分布式唯一Id，在一个分布式系统中的作用都有些什么呢。

举个例子，比如说业务量大了要做分库分表了，拿MyCat这个中间件举例，你对接了MyCat这个中间件后，你是接触不到MyCat后面被取余策略或其他策略分离的表的。MyCat呈现给你的仍然是一张表，但是却是分离的表的合并。 那这个时候你后面的数据库的主键要是还是和单机的策略一样，仍然是普通的auto_increment 1的话，你的分割后的每张表都会有重复的Id。
比如说有一组订单表order_1, order_2

<!-- more -->

对于order_1，有

|id|message|
|---|---|
|1|x|
|2|xx|

对于order_2，有

|id|message|
|---|---|
|1|xxx|
|2|xxxx|

但是MyCat呈现给你的视图却是order

|id|message|
|---|---|
|1|xxx|
|2|xxxx|
|1|x|
|2|xx|

由于你的应用程序对接的是MyCat，这样你的根据Id查询的策略就会出现很大的问题
```sql
select from order where id = 1;
```
这样一条筛选居然可以有两个结果，这就意味着一个订单号对应着两个交易，这在业务上显然是不符合实际的。这个时候我们就需要分布式唯一Id来解决这个问题。

## 如何实现一个分布式唯一Id

- 最简单的实现
```java
UUID.randomUUID().toString();
```
有兴趣看看关于UUID.randomUUID()中一些有意思的点的还可以看看w我的这篇文章[使用0xff来保证二进制补码的一致性](https://ayang818.gitee.io/blog/2019/12/14/%E4%BD%BF%E7%94%A80xff%E6%9D%A5%E4%BF%9D%E8%AF%81%E4%BA%8C%E8%BF%9B%E5%88%B6%E8%A1%A5%E7%A0%81%E7%9A%84%E4%B8%80%E8%87%B4%E6%80%A7/)

没写完。。。。。