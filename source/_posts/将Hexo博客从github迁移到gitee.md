---
title: 将Hexo博客从github迁移到gitee
date: 2019-07-31 13:35:18
tags: [随笔]
---
## 将Hexo博客从github迁移到gitee
由于原来基于Github Page的Hexo博客访问速度实在是太慢了，我把博客迁移到了Gitee上，加载速度算是大大提高了。下一步可能就是等这段时间忙完后，再把博客迁移到我的新的企鹅云的服务器上。

## 小插曲
在把SSH公钥部署到Gitee上的时候，不小心把自己的私钥给修改了。导致我GitHub都没法Remote Push。这里记录一下重新创建SSH Key的命令
```
ssh-keygen -t rsa -C "your_email@example.com"
```