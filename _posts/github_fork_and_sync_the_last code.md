---
title: 'github下fork后如何同步源库的新更新内容?'
date: 2016-07-23 21:35:40
tags: [Git]
categories: [Git]
---

* 使用
```
git remote -v
```
查看远程状态

* 给 fork 添加源库的clone地址
```
git remote add upstream 源库的clone地址
```

* 再次查看状态确认是否配置成功

* 从上源仓库 fetch 分支和提交点，并会被存储在一个本地分支 upstream/** 而不是origin /**
```
git fetch upstream
```

* 切换到本地分支

* 把 upstream/分支 分支合并到本地对应的分支上，这样就完成了同步，并且不会丢掉本地修改的内容，比如master分支的。
```
git merge upstream/master
```

* push
```
git push origin master
```
就好了。

<!-- more -->
