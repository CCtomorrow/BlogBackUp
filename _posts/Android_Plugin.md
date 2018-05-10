---
title: '四大组件插件化总结'
date: 2018-04-28 11:17:00
tags: [插件化]
categories: [Android,插件化]
---

### 前言
前面的文章都是讲四大组件的插件化的，然而虽然讲了四大组件的插件化，不过都是每个组件单独分析，并没有关联在一起，并不会让我们在一个比较高的维度上对插件化产生一个比较清晰的概述，这篇文章将会在一个比较高的维度上面来分析这个。

### 四大组件的区别
要针对四大组件插件化，那么到底要采取什么措施，是要看四大组件之间的区别的。
#### Activity
![activity_start](/images/activity_start_progress.png)
- 必须显式声明
必须在AndroidManifest.xml中显示声明使用的Activity。
- Activity生命周期
当然Service其他组件如Service也有生命周期，但是其他组件没有Activity那么复杂的生命周期。
- 启动模式
由于启动模式的存在，导致Activity即可以是每次打开Activity的时候都创新新的对象，也可以是复用已经存在的Activity而不创建对象。
- Activity栈
正因为有Activity栈这个东西的存在才会有，点击返回键可以返回上一个Activity的功能。

<!-- more -->

#### BroadcastReceiver
- BroadcastReceiver可以动态注册，也可以动态注册
比较大的区别是静态注册的BroadcastReceiver在应用关闭掉以后也可以收到BroadcastReceiver(静态BroadcastReceiver必须在AndroidManifest.xml中显示声明)，不过这个在国内其实并不能，因为国内的开发者各种利用BroadcastReceiver保活，然后厂商出各种策略，应用关闭以后，一般我们的应用是收不到的。
- BroadcastReceiver与IntentFilter
BroadcastReceiver有一个IntentFilter的概念(Activity也有，但是我们比较少的使用IntentFilter去启动Activity)，也就是说，每一个BroadcastReceiver只对特定的Broadcast感兴趣；而且，AMS在进行广播分发的时候，也会对这些BroadcastReceiver与发出的广播进行匹配，只有Intent匹配的Receiver才能收到广播。

#### Service
- Service无法拥有多实例
Service组件与Activity组件另外一个不同点在于，对同一个Service调用多次startService并不会启动多个Service实例，而非特定Flag的Activity是可以允许这种情况存在的。
- 生命周期不复杂
Service的生命周期相当简单：整个生命周期从调用 onCreate() 开始起，到 onDestroy() 返回时结束。对于非绑定服务，就是从startService调用到stopService或者stopSelf调用。对于绑定服务，就是bindService调用到unbindService调用。
- 可以启动足够多个
但是Service组件不一样，理论情况下，可以启动的Service组件是无限的——除了硬件以及内存资源，没有什么限制它的数目。
- 运行在不同的进程
当然这个其他组件也会，只是Service运行在其他的进程的场景和使用多一些。

#### ContentProvider
- ContentProvider本身是用来共享数据的
ContentProvider本身是用来共享数据的，因此它提供一般的CURD服务；它类似HTTP这种无状态的服务，没有Activity，Service所谓的生命周期的概念，服务要么可用，要么不可用；对应着ContentProvider要么启动，要么随着进程死亡；而通常情况下，死亡之后还会被系统启动。所以，ContentProvider，只要有人需要这个服务，系统可以保证是永生的；这是与其他组件的最大不同；完全不用考虑生命周期的概念。
- 原理
ContentProvider被设计为共享数据，这种数据量一般来说是相当大的；熟悉Binder的人应该知道，Binder进行数据传输有1M限制，因此如果要使用Binder传输大数据，必须使用类似socket的方式一段一段的读，也就是说需要自己在上层架设一层协议；ContentProvider并没有采取这种方式，而是采用了Android系统的匿名共享内存机制，利用Binder来传输这个文件描述符，进而实现文件的共享；这是第二个不同，因为其他的三个组建通信都是基于Binder的，只有ContentProvider使用了Ashmem。

### 四大组件的插件化方案
#### Activity
- Manifest问题
针对Activity的一些特性，比如，需要在Manifest里面注册，我们可以在AndroidManifest.xml里面声明一个替身Activity，然后在合适的时候把这个假的替换成我们真正需要启动的Activity就OK了；
- 生命周期
同时由于Activity生命周期特别复杂的原因，我们不能自己去管理Activity的生命周期，必须要让系统管理，这个可以有两种解决方案，一种是我们使用一个代理的Activity去调用真实Activity的生命周期，另一种就是使用存在的Activity；
- 启动模式
在Android中，Activity有不同的启动模式，我们声明了一个替身StubActivity，肯定没有满足所有的要求，因此，我们需要在AndroidManifest.xml中声明一系列的有不同launchMode的Activity，还需要完成替身与真正Activity launchMode的匹配过程，这样才能完成启动各种类型Activity的需求，同时由于我们的应用不可能会存在特别多连续的相同的启动模式的Activity，所以可以保证这个是可行的；
- Activity栈
如果启动的是系统认为真实的Activity那么这个问题就不存在了；

所以插件化方案:
当启动插件Activity时会在启动的时候被替换成占坑的Activity，然后Hook ActivityThread的H和Instrumentation类，在对应的回调中替换回插件的Activity，达到欺骗系统的效果。
下面说一个Replugin的方案:
RePlugin 使用的方式比较新颖，在启动的 Activity 时候替换成了合适的占坑的Activity ，同时记录占坑 Activity 和 真实 Activity 的映射关系，然后ClassLoader的 loadClass方法中，load 占坑 Activity 类的的时候，根据占坑组件跟真实组件的映射关系, 加载真实的组件. 这样的好处是，避免了过多的 Hook 系统组件。

#### BroadcastReceiver
BroadcastReceiver有一个IntentFilter的概念，这个就会导致我们使用Activity的方案，使用替身BroadcastReceiver方案不可行了。但是它存在可以动态注册这个功能。

所以插件化方案:
把静态BroadcastReceiver当作动态BroadcastReceiver处理。

#### Service
可以启动足够多个以及无法多实例导致我们不可能去使用Activity的方案，预埋。但是我们可以通过代理分发的方式来管理生命周期这种事。

所以插件化方案:
既然我们希望插件的Service具有一定的运行时优先级，那么一个货真价实的Service组件是必不可少的——只有这种被系统认可的真正的Service组件才具有所谓的运行时优先级。

因此，我们可以注册一个真正的Service组件ProxyService，让这个Service承载一个真正的Service组件所具备的能力（进程优先级等）；当启动插件的服务比如PluginService的时候，我们统一启动这个ProxyService，当这个ProxyService运行起来之后，再在它的onStartCommand等方法里面进行分发，执行PluginService的onStartCommond等对应的方法；我们把这种方案形象地称为「代理分发技术」

代理分发技术也可以完美解决插件Service可以运行在不同的进程的问题——我们可以在AndroidManifest.xml中注册多个ProxyService，指定它们的process属性，让它们运行在不同的进程；当启动的插件Service希望运行在一个新的进程时，我们可以选择某一个合适的ProxyService进行分发。也许有童鞋会说，那得注册多少个ProxyService才能满足需求啊？理论上确实存在这问题，但事实上，一个App使用超过10个进程的几乎没有；因此这种方案是可行的。

#### ContentProvider
插件化方案:
我们在宿主程序里面注册一个货真价实、被系统认可的StubContentProvider组件，把这个组件共享给第三方App；然后通过代理分发技术把第三方App对于插件ContentProvider的请求通过这个StubContentProvider分发给对应的插件。

但是这还存在一个问题，由于第三方App查阅的其实是StubContentProvider，因此他们查阅的URI也必然是StubContentProvider的authority，要查询到插件的ContentProvider，必须把要查询的真正的插件ContentProvider信息传递进来。这个问题的解决方案也很容易，我们可以制定一个「插件查询协议」来实现。
将真实的 ContentProvider 信息放在Uri 中, 通过解析Uri 获取插件的真实的 ContentProvider 信息。