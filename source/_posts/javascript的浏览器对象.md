---
title: javascript中的浏览器操作
date: 2019-07-07 23:08:23
tags:  js
---
## 学习JavaScript的初衷
其实我是一个后端选手，但是缺了前端，后端性能再怎么好，总是无法具象化体现。而工作室里现在差不多感觉只有我一个在做Web开发，所以也不得不成为一位前后端全干代码狗啦😁。

# 浏览器对象
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
    document.title = '快回来嗷!';
    ```
    常用方法
    - document.getElementById()

    - document.getElementByTagName()
    - document.cookie 获取当前页面的cookie

    为了解决这个问题cookie信息泄露问题，服务器在设置Cookie时可以使用httpOnly，设定了httpOnly的Cookie将不能被JavaScript读取。这个行为由浏览器实现，主流浏览器均支持httpOnly选项，IE从IE6 SP1开始支持。
    为了确保安全，服务器端在设置Cookie时，应该始终坚持使用httpOnly。

# 操作DOM

1. 更新DOM
    - 直接修改innerHTML的属性

    - 修改innerText或textContent属性，不会修改DOM树结构

2. 插入DOM
    - 如果这个DOM节点是空的，例如，```<div></div>```，那么，直接使用innerHTML = '```<span>child</span>```'就可以修改DOM节点的内容，相当于“插入”了新的DOM节点。

    - 如果这个DOM节点不是空的，那就不能这么做，因为innerHTML会直接替换掉原来的所有子节点。

    - 使用appendChild，把一个子节点添加到父节点的最后一个子节点。
    ```html
    <p id="js">JavaScript</p>
    <div id="list">
    <p id="java">Java</p>
    <p id="python">Python</p>
    <p id="scheme">Scheme</p>
    </div>
    ```
    使用如下js代码
    ```js
    var js = document.getElementById("js");
    var list = document.getElementById("list");
    list.appendChild(js);
    ```

    - 创建一个节点并append<br>
    ```js
    var list = document.getElementById("list");
    var node = document.createElement("p");
    node.innerText = "Scala";
    node.id = "scala";
    list.appendChild(node);
    ```
    下面代码实现了创建一个style节点，并把它添加到```<head>```标签的下面，这样就添加了新的CSS样式
    ```js
    var d = document.createElement('style');
    d.setAttribute('type', 'text/css');
    d.innerHTML = 'p { color: red }';
    document.getElementsByTagName('head')[0].appendChild(d);
    ```
    - 插入到指定位置
    ```js
    var py = document.getElementById("python");
    var list = document.getElementById("list");
    var scala = document.createElement("p");
    scala.id = "scala";
    scala.innerHtml = "Scala";
    list.insertBefore(scala, py);
    ```

    - Demo<br>
    对于一个已有的HTML结构
    ```html
    <!-- HTML结构 -->
    <ol id="test-list">
        <li class="lang">Scheme</li>
        <li class="lang">JavaScript</li>
        <li class="lang">Python</li>
        <li class="lang">Ruby</li>
        <li class="lang">Haskell</li>
    </ol>
    ```
    需要将其中的li按照顺序排好
    代码如下
    ```js
    var ol = document.getElementById("test-list");
    var list = ol.getElementsByTagName("li");
    var node_list = [];
    for(let item of list){
        node_list.push(item.innerText);
    }
    node_list.sort();
    index = 0;
    for(let item of list){
        item.innerText = node_list[index];  
        index++;
    }
    ```

3. 删除DOM
    
    - 删除一个DOM节点就比插入要容易得多。
    要删除一个节点，首先要获得该节点本身以及它的父节点，然后，调用父节点的removeChild把自己删掉。
    ```js
    // 拿到待删除节点:
    var self = document.getElementById('to-be-removed');
    // 拿到父节点:
    var parent = self.parentElement;
    // 删除:
    var removed = parent.removeChild(self);
    removed === self; // true
    ```

# 操作表单

- 文本框，对应的```<input type="text">```，用于输入文本；

- 口令框，对应的```<input type="password">```，用于输入口令；

- 单选框，对应的```<input type="radio">```，用于选择一项；

- 复选框，对应的```<input type="checkbox">```，用于选择多项；

- 下拉框，对应的```<select>```，用于选择一项；

- 隐藏文本，对应的```<input type="hidden">```，用户不可见，但表单提交时会把隐藏文本发送到服务器。

1. 获取值

    ```js
    // <input type="text" id="email">
    var input = document.getElementById('email');
    input.value; // '用户输入的值'
    ```
    这种方式可以应用于text、password、hidden以及select。<br>
    对于radio， checkbox来说，value返回的永远是实现设定好的值，所以需要使用
    ```js
    // <label><input type="radio" name="weekday" id="monday" value="1"> Monday</label>
    // <label><input type="radio" name="weekday" id="tuesday" value="2"> Tuesday</label>
    var mon = document.getElementById("monday");
    var tues = document.getElementById("tuesday");
    mon.checked // true or flase
    ```

2. 设置值<br>
    这个和获取值类似
    ```js
     // <input type="text" id="email">
    var input = document.getElementById('email');
    input.value = "ayang818@qq.com"; // '用户输入的值'
    ```

# 提交表单

1. 通过```<form>```元素的submit()方法提交一个表单，例如，响应一个```<button>```的click事件，在JavaScript代码中提交表单
    ```js
    <form id="test-form">
        <input type="text" name="test">
        <button type="button" onclick="doSubmitForm()">Submit</button>
    </form>

    <script>
    function doSubmitForm() {
        var form = document.getElementById('test-form');
        // 可以在此修改form的input...
        // 提交form:
        form.submit();
    }
    </script>
    ```
    这种方式的缺点是扰乱了浏览器对form的正常提交。浏览器默认点击```<button type="submit">```时提交表单，或者用户在最后一个输入框按回车键。
2. 因此，第二种方式是响应```<form>```本身的onsubmit事件，在提交form时作修改：

    ```js
    <form id="test-form" onsubmit="return checkForm()">
        <input type="text" name="test">
        <button type="submit">Submit</button>
    </form>

    <script>
    function checkForm() {
        var form = document.getElementById('test-form');
        // 可以在此修改form的input...
        // 继续下一步:
        return true;
    }
    </script>
    ```
    注意要return true来告诉浏览器继续提交，如果return false，浏览器将不会继续提交form，这种情况通常对应用户输入有误，提示用户错误信息后终止提交form。

3. 使用```<input type="hidden">```间接传输数据
    ```js
    <form id="login-form" method="post" onsubmit="return checkForm()">
        <input type="text" id="username" name="username">
        <input type="password" id="input-password">
        <input type="hidden" id="md5-password" name="password">
        <button type="submit">Submit</button>
    </form>

    <script>
    function checkForm() {
        var input_pwd = document.getElementById('input-password');
        var md5_pwd = document.getElementById('md5-password');
        // 把用户输入的明文变为MD5:
        md5_pwd.value = toMD5(input_pwd.value);
        // 继续下一步:
        return true;
    }
    </script>
    ```
    注意到id为md5-password的```<input>```标记了name="password"，而用户输入的id为input-password的```<input>```没有name属性。没有name属性的```<input>```的数据不会被提交。

# 操作文件
1. 提交文件
    ```<input type="file">```
    注意：当一个表单包含```<input type="file">```时，表单的enctype必须指定为multipart/form-data，method必须指定为post，浏览器才能正确编码并以multipart/form-data格式发送表单的数据。

    出于安全考虑，浏览器只允许用户点击```<input type="file">```来选择本地文件，用JavaScript对```<input type="file">```的value赋值是没有任何效果的。当用户选择了上传某个文件后，JavaScript也无法获得该文件的真实路径。

    检查文件后缀名
    ```js
    var file = document.getElementById("file-update");
    var fileName = file.value;
    if (!fileName || fileName.endWith(".jpg") || fileName.endWith("png")){
        alert("can only update picture");
        return false;
    }
    ```

2. FileAPI
    <br>
    HTML5的File API提供了File和FileReader两个主要对象，可以获得文件信息并读取文件。
    ```js
    var
        fileInput = document.getElementById('test-image-file'),
        info = document.getElementById('test-file-info'),
        preview = document.getElementById('test-image-preview');
    // 监听change事件:
    fileInput.addEventListener('change', function () {
        // 清除背景图片:
        preview.style.backgroundImage = '';
        // 检查文件是否选择:
        if (!fileInput.value) {
            info.innerHTML = '没有选择文件';
            return;
        }
        // 获取File引用:
        var file = fileInput.files[0];
        // 获取File信息:
        info.innerHTML = '文件: ' + file.name + '<br>' +
                        '大小: ' + file.size + '<br>' +
                        '修改: ' + file.lastModifiedDate;
        if (file.type !== 'image/jpeg' && file.type !== 'image/png' && file.type !== 'image/gif') {
            alert('不是有效的图片文件!');
            return;
        }
        // 读取文件:
        var reader = new FileReader();
        // 回调函数
        reader.onload = function(e) {
            var
                data = e.target.result; // 'data:image/jpeg;base64,/9j/4AAQSk...(base64编码)...'            
            preview.style.backgroundImage = 'url(' + data + ')';
        };
        // 以DataURL的形式读取文件:
        reader.readAsDataURL(file);
    });
    ```

3. 异步回调<br>
上面的代码还演示了JavaScript的一个重要的特性就是单线程执行模式。在JavaScript中，浏览器的JavaScript执行引擎在执行JavaScript代码时，总是以单线程模式执行，也就是说，任何时候，JavaScript代码都不可能同时有多于1个线程在执行。


<h1>AJAX(Asynchronous Javascript And XML)</h1>

1. 使用<br>

    这里了解的已经比较多了，就不详细写了，AJAX是异步执行的所以需要设置回调函数，在现代浏览器上写AJAX主要依靠XMLHttpRequest对象
    <a name="ajax"></a>
    ```js
    var request = new XMLHttpRequest();

    request.onreadystatechange = function () {
        if (request.readyState === 4) { 
            if (request.status === 200) {
                var text = request.responseText;
                console.log(text);
            } else {
                console.log("failed");
            }
        } else {
            // HTTP请求还在继续...
        }
    }
    request.open("GET","https://www.baidu.com");
    request.send();
    console.log("其实这是个异步请求");
    ```

2. 安全限制<br>
    上面的代码实际上若不是再百度的域名下访问，是会收到如下报错的
    ```
   Access to XMLHttpRequest at 'https://www.baidu.com/' from origin 'https://www.liaoxuefeng.com' has been blocked by CORS policy: No 'Access-Control-Allow-Origin' header is present on the requested resource.
    ```
    意思就是请求被浏览器的同源策略所拦截，如图
    ![](https://www.liaoxuefeng.com/files/attachments/1027024093709472/l)
    但是默认情况下，JavaScript在发送AJAX请求时，URL的域名必须和当前页面完全一致。完全一致的意思是，域名要相同（www.example.com和example.com不同），协议要相同（http和https不同），端口号要相同（默认是:80端口，它和:8080就不同）。有的浏览器口子松一点，允许端口不同，大多数浏览器都会严格遵守这个限制。<br>
    所以为了解决这个问题，我们引入一个叫做跨域请求的概念。
    在日常使用中，比如我之前用nodejs写的[前后端分离应用](https://github.com/ayang818/Backstage-Management)
    用的就是aros跨域方法，具体的可以参考[aros文档](https://www.w3.org/TR/cors/)。

# Promise
在JavaScript的世界中，所有代码都是单线程执行的。

由于这个“缺陷”，导致JavaScript的所有网络操作，浏览器事件，都必须是异步执行。异步执行可以用回调函数实现

参考上面的<a href="#ajax">代码</a>，AJAX就是个典型的异步操作，但是有没有比上面的代码更好用的代码呢？
可以写成这样
```js
var ajax = ajaxGet("https://www.baidu.com");
ajax.ifSuccess(success);
    .ifFail(fail);
```
这种链式写法的好处在于，先统一执行AJAX逻辑，不关心如何处理结果，然后，根据结果是成功还是失败，在将来的某个时候调用success函数或fail函数。

古人云：“君子一诺千金”，这种“承诺将来会执行”的对象在JavaScript中称为Promise对象。
代码示例
```js
function test(resolve, reject) {
    var timeOut = Math.random() * 2;
    log('set timeout to: ' + timeOut + ' seconds.');
    setTimeout(function () {
        if (timeOut < 1) {
            log('call resolve()...');
            resolve('200 OK');
        }
        else {
            log('call reject()...');
            reject('timeout in ' + timeOut + ' seconds.');
        }
    }, timeOut * 1000);
}

new Promise(test).then(function (result) {
    console.log('成功：' + result);
})
    .catch(function (reason) {
    console.log('失败：' + reason);
});
```
可见Promise最大的好处是在异步执行的流程中，把执行代码和处理结果的代码清晰地分离了，用图来表示就是![](https://www.liaoxuefeng.com/files/attachments/1027242914217888/l)

再举个栗子，执行网络请求时也是这样
```js
function ajax(method, url, data) {
    var request = new XMLHttpRequest();
    return new Promise(function (resolve, reject) {
        request.onreadystatechange = function () {
            if (request.readyState === 4) {
                if (request.status === 200) {
                    resolve(request.responseText);
                } else {
                    reject(request.status);
                }
            }
        };
        request.open(method, url);
        request.send(data);
    });
}
var request = ajax("GET", "https://www.baidu.com");
request.then(function(text){
    console.log("请求内容为"+text);
}).catch(function(result){
    console.log("错误代码："+result);
});
```
Promise还可以并行执行两个任务，使用
```Promise.all([promise1,promise2]).then()```

---
浏览器章节到这里就结束了，花了差不多五天的时间，仔细读了下javascript使用。和我学过的java还有python进行比较，不得不说有惊喜的地方，也有失望的地方，js更像python和java的结合，它没有java那么繁琐，但是也没有python那么简洁，然后下一步我打算学习Vue了，当可以比较熟练的使用常用的前端技能的时候，我就又做回我的老本行后端啦~



    










        











    