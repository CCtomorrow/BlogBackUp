---
title: '插件化总结'
date: 2018-04-28 11:17:00
tags: [插件化]
categories: [Android,插件化]
---

### 前言
前面的文章都是讲四大组件的插件化的，然而虽然讲了四大组件的插件化，不过都是每个组件单独分析，并没有关联在一起，并不会让我们在一个比较高的维度上对插件化产生一个比较清晰的概述，这篇文章将会在一个比较高的维度上面来分析这个。

### 四大组件的区别
要针对四大组件插件化，那么到底要采取什么措施，是要看四大组件之间的区别的。
| Activity | BroadcastReceiver | Service | ContentProvider |
| -------- | ----------------- | ------- | --------------- |
Activity
![activity_start](/images/activity_start_progress.png)
- 启动模式
由于启动模式的存在，导致Activity即可以是每次打开Activity的时候都创新新的对象，也可以是复用已经存在的Activity而不创建对象。
- Activity栈
正因为有Activity栈这个东西的存在才会有，点击返回键可以返回上一个Activity的功能。
- Activity生命周期
当然Service其他组件如Service也有生命周期，但是其他组件没有Activity那么复杂的生命周期。
- 必须显式声明
必须在AndroidManifest.xml中显示声明使用的Activity。

<!-- more -->

BroadcastReceiver
- BroadcastReceiver可以动态注册，也可以动态注册
比较大的区别是静态注册的BroadcastReceiver在应用关闭掉以后也可以收到BroadcastReceiver(静态BroadcastReceiver必须在AndroidManifest.xml中显示声明)，不过这个在国内其实并不能，因为国内的开发者各种利用BroadcastReceiver保活，然后厂商出各种策略，应用关闭以后，一般我们的应用是收不到的。
- BroadcastReceiver与IntentFilter
BroadcastReceiver有一个IntentFilter的概念(Activity也有，但是我们比较少的使用IntentFilter去启动Activity)，也就是说，每一个BroadcastReceiver只对特定的Broadcast感兴趣；而且，AMS在进行广播分发的时候，也会对这些BroadcastReceiver与发出的广播进行匹配，只有Intent匹配的Receiver才能收到广播。

Service
- Service无法拥有多实例
Service组件与Activity组件另外一个不同点在于，对同一个Service调用多次startService并不会启动多个Service实例，而非特定Flag的Activity是可以允许这种情况存在的。
- 生命周期不复杂
Service的生命周期相当简单：整个生命周期从调用 onCreate() 开始起，到 onDestroy() 返回时结束。对于非绑定服务，就是从startService调用到stopService或者stopSelf调用。对于绑定服务，就是bindService调用到unbindService调用。
- 运行在不同的进程
当然这个其他组件也会，只是Service运行在其他的进程的场景和使用多一些。

ContentProvider
- ContentProvider本身是用来共享数据的
ContentProvider本身是用来共享数据的，因此它提供一般的CURD服务；它类似HTTP这种无状态的服务，没有Activity，Service所谓的生命周期的概念，服务要么可用，要么不可用；对应着ContentProvider要么启动，要么随着进程死亡；而通常情况下，死亡之后还会被系统启动。所以，ContentProvider，只要有人需要这个服务，系统可以保证是永生的；这是与其他组件的最大不同；完全不用考虑生命周期的概念。
- 原理
ContentProvider被设计为共享数据，这种数据量一般来说是相当大的；熟悉Binder的人应该知道，Binder进行数据传输有1M限制，因此如果要使用Binder传输大数据，必须使用类似socket的方式一段一段的读，也就是说需要自己在上层架设一层协议；ContentProvider并没有采取这种方式，而是采用了Android系统的匿名共享内存机制，利用Binder来传输这个文件描述符，进而实现文件的共享；这是第二个不同，因为其他的三个组建通信都是基于Binder的，只有ContentProvider使用了Ashmem。
