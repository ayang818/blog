---
title: javascript的浏览器对象
date: 2019-07-07 23:08:23
tags:  js
---
## 学习JavaScript的初衷
其实我是一个后端选手，但是缺了前端，后端性能再怎么好，总是无法具象化体现。而工作室里现在差不多感觉只有我一个在做Web开发，所以也不得不成为一位前后端全干代码狗啦😁。

1. window

    window对象不但充当全部作用域，而且表示浏览器窗口。
    window对象有innerHeight对象和innerHeight，可以获取浏览器的内部宽度和高度
<!-- more -->
2. navigator

    navigator对象对象表示浏览器的信息，最常用的属性包括
    - navigator.appName 浏览器名称
    - navigator.appVersion 浏览器版本
    - navigator.language 浏览器语言
    - navigator.platform 浏览器系统类型
    - navigator.userAgent 浏览器设置的User-Agent字串

    下面代码是使用短路运算符得到网页宽度的方法
    ```js
    var width = window.innerWidth || document.body.clientWidth;
    ```
3. screen

    screen对象表示屏幕的信息，常用的属性有
    - screen.width 屏幕宽度
    - screen.height 屏幕高度，px为单位
    - screen.colorDepth 返回颜色位数

4. location

    - location对象表示当前页面的URl信息
    - location.href 获取当前url
    - location.protocol
    - location.host
    - location.port
    - location.pathname
    - location.search
    - location.hash
    - location.reload()  刷新页面
    - location.assign("url") 重定向到一个网页

5. docement

    document对象表示当前页面。由于HTML在浏览器中以DOM形式表示为树形结构，document对象就是整个DOM树的根节点。
    
    ```js
    // 修改标题
    document.title = '努力学习JavaScript!';
    ```
    常用方法
    - document.getElementById()
    - document.getElementByTagName()
    - document.cookie 获取当前页面的cookie

    为了解决这个问题cookie信息泄露问题，服务器在设置Cookie时可以使用httpOnly，设定了httpOnly的Cookie将不能被JavaScript读取。这个行为由浏览器实现，主流浏览器均支持httpOnly选项，IE从IE6 SP1开始支持。
    为了确保安全，服务器端在设置Cookie时，应该始终坚持使用httpOnly。



    