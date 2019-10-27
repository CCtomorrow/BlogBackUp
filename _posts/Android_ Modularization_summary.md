---
title: '安卓组件化总结'
date: 2018-12-01 23:30:00
tags: [Router]
categories: [Android,Router]
---
这是一篇迟来的安卓的组件化文章的总结，文章主要讲述组件化过程中，需要解决的问题，以及目前的组件化框架是如何解决这个问题的，这篇文章预计会很长，因为想详细分析一下目前存在的比较知名的组件化框架。

组件化框架太多，预计，文章会分析如下组件化框架:
[ARouter](https://github.com/alibaba/ARouter)
[WMRouter](https://github.com/meituan/WMRouter)
[CC](https://github.com/luckybilly/CC)
[JIMU](https://github.com/mqzhangw/JIMU)
个人精力实在有限，而且上面的四个组件化框架已经很有代表性了。

下面是一些比较好的组件化的文章:
// todo
1. [Android 模块化探索与实践](https://zhuanlan.zhihu.com/p/26744821)
2. []()
3. []()
4. []()
5. []()

组件化需要解决的问题:
1. 路由实现
2. 参数传递
3. 组件通信

当然这篇文章[AndroidComponentizeLibs](https://github.com/luckybilly/AndroidComponentizeLibs)里面描述了各种问题，其实最重要的还是上面列出的。本文也只分析上面提出的问题，如果文章篇幅实在过长，会考虑分篇。

<!-- more -->

### 路由实现
一个路由框架大概要解决下面的几个问题:
1. 模块之间页面跳转
2. 模块之间数据传递
3. 模块初始化处理

路由实现总体来说其实只有两种思路
1. Activity在Manifest里面配置intent-filter然后直接跳转
2. 给Activity加上注解，然后使用apt收集url和Activity的对应关系跳转

当然目前看到的很多的框架都是使用第二种方式，可能就在网页调起原生这里使用了第一种Scheme的形式。

使用第一种方式的框架:
[Floo](https://github.com/drakeet/Floo)
[Router](https://github.com/BaronZ88/Router)
当然第一种方案由于每个页面都需要在Manifest里面配置数据，所以其实并不是特别好，这也是目前大部分的框架没有使用这种方式的原因，那么现在开始分析上面列举的四种框架的路由实现方案。

