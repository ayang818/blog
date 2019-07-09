---
title: 'JQuery: 开箱即用的js库'
date: 2019-07-09 20:51:40
tags: js
---

## 1.基本使用
要使用首先要先引入CDN
```js
<script src="https://apps.bdimg.com/libs/jquery/2.1.4/jquery.min.js">
</script>
```
使用超级简单，通常是这样的格式
```js
$("").action(function() {

})
```
即先选择，后监听事件，然后的形式
<!-- more -->
## 2.选择器
选择器有三种
- 元素选择器
```js
$("TagName")
```

- id选择器
```js
$("#id")
```

- 类选择器
```js
$(".class")
```

## 3.常见DOM(Document Object Model)事件
|鼠标事件|键盘事件|表单事件|文档/窗口事件|
|-|-|-|-|
|click(点击)|keypress(按键盘)|submit|load|
|dbclick(双击)|keydown|change|resize|
|mouseenter(鼠标穿过元素)|keyup|focus(获得光标焦点)|scroll|
|mouseleave(鼠标离开元素)||blur|unload|
|hover(光标悬停,两次操作)||||

## 4.JQuery效果
- hide() 元素隐藏

- show() 元素显示

- toggle(fast/slow/1000, callback) 元素的隐藏和显示切换 

- fadeIn() 淡入

- fadeOut() 淡出

- fadeToggle() 淡入淡出切换

- fadeTo(speed, opacity, callback) 渐变到不透明度

- slideDown() 向下滑动

- slideUp() 向上滑动

- slideToggle() 上下滑动切换

- 
## 5.JQuery捕获

- text() 设置或返回所选元素的文本内容

- html() 设置或返回所选元素的内容（包括 HTML 标记）

- val() 设置或返回表单字段的值

- attr() 用于获取属性值。

## 6.JQuery添加元素

- append() 在被选元素的结尾插入内容（仍然该元素的内部）。

- prepend() 在被选元素的开头插入内容。

- after() 被选元素之后插入内容

- before()  在被选元素之前插入内容

```js
var txt1="<b>Useful </b>";                    // 使用 HTML 创建元素
var txt2=$("<i></i>").text("js-- ");     // 使用 jQuery 创建元素
var txt3 = document.createElement("big");  // 使用 DOM 创建元素
txt3.innerHTML="jQuery!";
$("p").after(txt1,txt2,txt3);          // 在图片后添加文本
```

## 7.JQuery删除元素

- remove() 删除被选元素及其子元素

- empty() 删除被选元素的子元素

## 8.JQuery AJAX
- AJAX load(URL, data, callback) 从服务器加载数据，并把返回的数据放入被选元素中

- $.get(URL, callback) 通过 HTTP GET 请求从服务器上请求数据

- $.post(URL, data, callback) 通过 HTTP POST 请求向服务器提交数据



