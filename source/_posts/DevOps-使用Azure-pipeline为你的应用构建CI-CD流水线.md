---
title: '[DevOps] 使用Azure pipeline + Github为你的应用构建CI/CD流水线'
date: 2020-04-24 12:18:07
tags:  [DevOps]
---

**2020/11 upd ：文章应该咕咕咕了，但是具体内容可以参照我的视频**

- [DevOps系列篇 - 1. DevOps是什么？DevOps的主要流程？](https://www.bilibili.com/video/BV1YK4y1k7f2)
- [DevOps系列篇 - 2. 使用Docker将一个最简Web应用容器化](https://www.bilibili.com/video/BV1UT4y1g76F)
- [微软官方devops教程](https://azure.microsoft.com/zh-cn/overview/devops-tutorial/)

# 什么是 DevOps ？什么是 CI/CD 流水线？
这个问题由于网上已经有很多很好的文章了，所以我这里就不再详细写了。大家可以自行参考。
[什么是DevOps？什么是 CI/CD 流水线？](https://www.infoq.cn/article/WHt0wFMDRrBU-dtkh1Xp)

我归纳一下，
1. DevOps是敏捷开发的一种主流实现方式，相比于传统的瀑布模型，DevOps为我们快速开发应用，快速发布应用，快速部署应用提供了一种方法论。
2. DevOps中大致有如下几个环节<br>
    1. Version Control （版本控制）
    2. Continuous Integration（持续集成）
    3. Continuous Delivery （持续交付） 
    4. Continuout Deployment（持续部署）

# 如何为你的应用搭建完整的 CI/CD 流水线
在这篇文章中，我们将使用 Git + [Github](https://www.github.com) + Docker + [Azure pipeline](https://azure.microsoft.com/en-us/services/devops/pipelines/) 的组合来完成一个简单的 node 服务端应用的自动化流水线的搭建。

1. 创建一个node应用
    1. 初始化项目 
    ```bash
    npm init
    ```
    2. 安装typescript
    ```bash
    npm install -g typescript
    ```
    3. 初始化为TypeScript应用
    ```bash
    tsc --init
    ```
    4. 安装我们需要的 express 依赖
    ```bash
    npm install express @types/express
    ```
    5. 编写一个最简单的Web应用

<!-- more -->
`TODO`