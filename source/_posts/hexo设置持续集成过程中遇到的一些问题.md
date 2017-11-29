---
title: hexo设置持续集成过程中遇到的一些问题
date: 2017-11-26 19:42:55
tags: [hexo,CI,git subtree,來必力]
categories:
- 博客
---

由于hexo d的方式来布置hexo博客不能满足多台计算机编辑以及备份的需求，所以参考[Hexo的版本控制与持续集成](https://formulahendry.github.io/2016/12/04/hexo-ci/)的博客，进行了持续集成的配置，效果显著， 终于可以随时随地的编辑博客了。

但是这个过程中也遇到一个问题，就是当我们使用第三方主题的时候（我使用的是next）主题文件夹不能上传到source repository，原因也很简单，因为我是直接使用`git clone https://github.com/iissnan/hexo-theme-next themes/next`克隆的next主题，所以next文件夹下有.git，是一个独立的repo，当然无法上传，删掉.git后就一切正常了，但这也带来另一个问题，就是我们想更新主题的时候比较麻烦，因为已经是一个独立的本地文件夹了，后来经过搜索，确定了使用git subtree来解决这个问题。

关于git subtree的内容大家可以自行网上搜索，简单来说就是git用来管理子项目的一种方式，整个操作过程如下

```shell
# 添加next仓库，命名为next
git remote add next https://github.com/iissnan/hexo-theme-next
#添加next的master分支到子树
git subtree add --prefix=themes/next  next master --squash
#看看有哪些稳定版本
git ls-remote --tags next
#测试切换到5.1.2版本
git subtree pull --prefix=themes/next  next tags/v5.1.2 --squash
#测试切换到5.1.3版本
git subtree pull --prefix=themes/next  next tags/v5.1.3 --squash
```

在切换到5.1.2版本后，我修改了主题配置文件，添加了livere_uid(來必力评论配置，在next主题下配置这一行就可以开启，简单易配置，不翻墙就可以评论，推荐大家使用)，然后尝试切换到5.1.3版本，发现很智能保留了这个配置（两个版本的配置文件是不同的，merge的过程很顺利），所以通过这个方案切换新版本应该是可行的。

这里需要注意的即使代码的第4行，我一开始写的是`git subtree add --prefix=themes/next  next tags/v5.1.2 --squash`，在这样的情况下5.1.2版本的下载倒是没有问题，但是更新到5.1.3时，会提示已经是最新的代码无法更新，相信大多数人还是以稳定版为主，如果不是想折腾一下源码的话，所以推荐按照现在的顺序来配置。