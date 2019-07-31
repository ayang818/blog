---
title: '使用express实现注册登陆接口,以及jwt的使用'
date: 2019-07-13 23:18:06
tags: [express,前后端分离]
---
## 首先贴一个[项目地址](https://github.com/ayang818/Express-login-demo)

技术栈是express+mongodb+cors跨域方案解决+jwt token令牌

## 1.需求分析
- 注册(username唯一性)
- 登陆(使用bcrypt加密)
- jwt令牌登陆态控制
<br>
<!-- more -->
## 2.首先是express中mongodb操作
照例是先链接数据库
```js
const mongoose = require("mongoose");
mongoose.connect("mongodb://119.23.240.115:27017/express_login", {
    useCreateIndex: true,    
    useNewUrlParser: true    
})
```
其次是创建User表结构，这里由于只是Demo演示，所以只是写了两个字段
```js
const User = mongoose.models("User", new mongoose.Schema({
    username: {type: String, unique: true},
    // 在这里使用bcrypt加密算法
    password: {type: String, set(val){
        return require("bcrypt").hashSync(val, 10);
    }}
}))
// 导出模型
module.exports = { User }
```

## 3.其次是后端api调用
### 查询所有用户
```js
app.get("/api/users", async (req, res) => {
    const users = await User.find();
    res.send(users);
})
```
### 用户注册
```js
app.post("/api/register", async (req, res) => {
    const user = await User.create({
        username: req.body.username,
        password: req.body.password
    });
})
```
### 用户登陆
```js
app.post("/api/login", async (req, res) => {
    const user = await User.findOne({
        username: req.body.username
    })
    if (!user) {
        return res.status(422).send({
            message: "用户名不存在"
        })
    }
    const isPasswordValid = require("bcrypt").conpareSync(req.body.password, user.password);
    if (!isPasswordValid) {
        return res.status(422).send({
            message: "密码错误"
        })
    }
    // 接下来使用jwt(jsonwebtoken)生成令牌
    const token = require("jsonwebtoken").sign({
        // 注意这里千万不能把密码当作令牌加密
        id: String(user._id)
    }, "secretKey, should be a golbal variable")
    res.send({
        user,
        token: token
    })
})
```
关于jwt，推荐一篇[文章](https://www.tomczhen.com/2017/05/25/5-easy-steps-to-understanding-json-web-tokens-jwt/)，通俗易懂

### 身份认证中间件编写
身份认证中间件的编写的作用是为了减少代码的复用
```js
// 这里默认每一次登陆后的api请求头的authorization字段里都含有jwt
const authorMiddleware = async (req, res, next) => {
    const raw = String(req.header.authorization).split(" ").pop();
    // 校验令牌
    const { id } = require("jsonwebtoken").verify(raw, "secretKey");
    req.user = await User.findById(id);
    next();
}
```

### 请求时身份认证中间件
```js
app.post("/api/profile", authorMiddleware, async (req, res) => {
    res.send(req.user);
})
```

---

这样就实现了一套完整的注册登陆认证的服务，但是显然这套方案是有问题的。

写完我发现的一个问题就是注册时——密码是以明文传输到后端API，的这显然是不可行的，所以应该改为前端就是用bcrypt直接加密，随后将密文传输到后端API。这里没有写前端界面，所以暂且不修改。

还有一个不确定的问题在于jwt的加密方式，从代码提示看jwt像是对称加密，这里看起来在安全性上也有点问题，等过段时间仔细看下jwt的正确使用是否是我现在这样放在请求头里用来验证数据的的。