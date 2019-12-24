---
title: maven模块化最小实践
date: 2019-12-18 21:25:05
tags: [maven,微服务]
---

# 为什么要进行maven模块化
在说模块化之前，不妨说说非模块化的项目是怎么样的，非模块化的项目往往是一个单体应用，他的目录结构如下

![](https://upload-serve.oss-cn-beijing.aliyuncs.com/image/single.png)

为什么说是单体的呢，事实上
1. 上面看到的所有的包比如controller，mapper，utils，
, 这些一个个应用里的功能都是业务的一部分，到时候打jar包的时候其实就是把这些所有的功能打到同一个jar包中。然后运行就可以了。
2. 整个项目只有一个pom文件，功能与项目强耦合，代码复用能力差。

那么对于一个多模块项目来说呢，他的目录结构如下

<!-- more -->

下面是一个maven模块化项目

![maven项目](https://upload-serve.oss-cn-beijing.aliyuncs.com/image/module.png)

下面是一个maven模块化项目的一个子目录

![maven项目单个模块](https://upload-serve.oss-cn-beijing.aliyuncs.com/image/module_detail.png)

下面是一个. net core的微服务项目

![非maven项目](https://upload-serve.oss-cn-beijing.aliyuncs.com/image/module_cs.png)

不管是第一个maven项目和第二个非maven(. net core)项目，他们都不再是以一个功能文件夹来分割，而是通过一个个模块来组织代码，每个模块有其自己继承自根目录的pom文件。

这样看来maven的模块化项目的优势就呼之欲出了
- 划分模块，鼓励代码多项目重用，解耦合，便于部署独立微服务
- 防止某一个pom文件过于庞大，所有的模块pom继承自根目录下的pom，版本控制粒度不会受到影响

最后其实写这篇博客之前，我自己又给自己挖了一个坑————[Kugga Kugga的服务端，提供消息服务，授权认证，文件储存转发等功能](https://github.com/ayang818/Kugga)。我打算写一个小型的聊天软件(没错，摆脱CRUD的苦海)，顺带学习NIO和Netty。但是事实上这应该算是我写过的业务最复杂的项目了，然后我参照一些微服务框架搭建的项目，比如说[Kahla](https://github.com/AiursoftWeb/Kahla)，[Pig](https://gitee.com/log4j/pig)。发现他们都是基于模块化的应用。再想到自己很少服用自己之前的代码，总是重复造轮子，于斯我打算尝试使用模块化来搭建这次的项目。

## 搭建过程
搭建过程参考了[这篇文章](https://segmentfault.com/a/1190000011367492)，内容我不细写了。说一下基本的搭建逻辑

- 根目录使用spring initializr创建一个spring boot项目，然后把出了pom.xml外的文件都删了。在pom.xml中添加子模块的名称

```xml
<modules>
    <module>kugga-easy-server</module>
    <module>kugga-starter</module>
</modules>
```

- 在根目录下创建子模块(maven工程)，将其parent指向根目录pom的坐标，，然后可以省略模块目录的groupId和version，这两个属性会自动继承根目录pom的值

```xml
<parent>
    <artifactId>kugga</artifactId>
    <groupId>com.ayang818</groupId>
    <version>1.0.0</version>
</parent>
```

- 还有几个要注意的点就是要是想要解决模块下dependence的版本控制，可以在根目录下写一个

```xml
<dependencyManagement>
    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>fastjson</artifactId>
        <version>1.2.57</version>
    </dependency>
</dependencyManagement>
```

然后模块下的pom就可以不写版本号了，他会递归直到找到这个依赖的版本，这种方法在一开始创建的spring boot中也会看到。
```xml
<dependencies>
    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>fastjson</artifactId>
    </dependency>
</dependencies>
```