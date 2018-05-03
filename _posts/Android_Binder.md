---
title: 'Binder知识点记录'
date: 2018-05-03 11:17:00
tags: [插件化]
categories: [Android,插件化]
---

### 前言
对Binder的知识点做一些记录，以便日后复习。其中大部分的文字都可以从下面的文章中找到。
[Android Bander设计与实现](https://blog.csdn.net/universus/article/details/6211589)
[彻底理解Android Binder通信架构](http://gityuan.com/2016/09/04/binder-start-service/)
[图文详解Binder机制原理](https://blog.csdn.net/carson_ho/article/details/73560642)

### 进程知识
每个Android的进程，只能运行在自己进程所拥有的虚拟地址空间。对于用户空间，不同进程之间彼此是不能共享的，而内核空间却是可共享的。通过系统调用，用户空间可以访问内核空间。

两个运行在用户空间的进程A和进程B如何完成通信呢？
内核可以访问A和B的所有数据，所以，最简单的方式是通过内核做中转。
![跨进程通信的基本原理](/images/process_base_comm.png)
图片取自===>[图文详解Binder机制原理.](https://blog.csdn.net/carson_ho/article/details/73560642)

Linux内核实际上没有从一个用户空间到另一个用户空间直接拷贝的函数，需要先用copy_from_user()拷贝到内核空间，再用copy_to_user()拷贝到另一个用户空间。为了实现用户空间到用户空间的拷贝，mmap()分配的内存除了映射进了接收方进程里，还映射进了内核空间。所以调用copy_from_user()将数据拷贝进内核空间也相当于拷贝进了接收方的用户空间，这就是Binder只需一次拷贝的`秘密`。
![Binder跨进程通信的原理](/images/binder_process_comm.png)

<!-- more -->

### Binder通信模型
![IPC-Binder](/images/IPC-Binder.jpg)
可以看出无论是注册服务和获取服务的过程都需要ServiceManager，需要注意的是此处的Service Manager是指Native层的ServiceManager（C++），并非指framework层的ServiceManager(Java)。ServiceManager是整个Binder通信机制的大管家，是Android进程间通信机制Binder的守护进程，Client端和Server端通信时都需要先获取Service Manager接口，才能开始通信服务, 当然查找到目标信息可以缓存起来则不需要每次都向ServiceManager请求。

#### Binder驱动
和路由器一样，Binder驱动虽然默默无闻，却是通信的核心。尽管名叫‘驱动’，实际上和硬件设备没有任何关系，只是实现方式和设备驱动程序是一样的：它工作于内核态，提供open()，mmap()，poll()，ioctl()等标准文件操作，以字符驱动设备中的misc设备注册在设备目录/dev下，用户通过/dev/binder访问该它。驱动负责进程之间Binder通信的建立，Binder在进程之间的传递，Binder引用计数管理，数据包在进程之间的传递和交互等一系列底层支持。驱动和应用程序之间定义了一套接口协议，主要功能由ioctl()接口实现，不提供read()，write()接口，因为ioctl()灵活方便，且能够一次调用实现先写后读以满足同步交互，而不必分别调用write()和read()。Binder驱动的代码位于linux目录的drivers/misc/binder.c中。

#### ServiceManager
和DNS类似，ServiceManager的作用是将字符形式的Binder名字转化成Client中对该Binder的引用，使得Client能够通过Binder名字获得对Server中Binder实体的引用。注册了名字的Binder叫实名Binder，就象每个网站除了有IP地址外还有自己的网址。Server创建了Binder实体，为其取一个字符形式，可读易记的名字，将这个Binder连同名字以数据包的形式通过Binder驱动发送给ServiceManager，通知ServiceManager注册一个名叫张三的Binder，它位于某个Server中。驱动为这个穿过进程边界的Binder创建位于内核中的实体节点以及ServiceManager对实体的引用，将名字及新建的引用打包传递给ServiceManager。ServiceManager收数据包后，从中取出名字和引用填入一张查找表中。

细心的读者可能会发现其中的蹊跷：ServiceManager是一个进程，Server是另一个进程，Server向ServiceManager注册Binder必然会涉及进程间通信。当前实现的是进程间通信却又要用到进程间通信，这就好象蛋可以孵出鸡前提却是要找只鸡来孵蛋。Binder的实现比较巧妙：预先创造一只鸡来孵蛋：ServiceManager和其它进程同样采用Binder通信，ServiceManager是Server端，有自己的Binder对象（实体），其它进程都是Client，需要通过这个Binder的引用来实现Binder的注册，查询和获取。ServiceManager提供的Binder比较特殊，它没有名字也不需要注册，当一个进程使用BINDER_SET_CONTEXT_MGR命令将自己注册成ServiceManager（会用到ioctl(fd, cmd, arg)函数，cmd为BINDER_SET_CONTEXT_MGR）时Binder驱动会自动为它创建Binder实体（这就是那只预先造好的鸡）。其次这个Binder的引用在所有Client中都固定为0而无须通过其它手段获得。也就是说，一个Server若要向ServiceManager注册自己Binder就必需通过0（即NULL指针）这个引用号和ServiceManager的Binder通信。类比网络通信，0号引用就好比域名服务器的地址，你必须预先手工或动态配置好。要注意这里说的Client是相对ServiceManager而言的，一个应用程序可能是个提供服务的Server，但对ServiceManager来说它仍然是个Client。

#### Client获得实名Binder的引用
Server向ServiceManager注册了Binder实体及其名字后，Client就可以通过名字获得该Binder的引用了。Client也利用保留的0号引用向ServiceManager请求访问某个Binder：我申请获得名字叫张三的Binder的引用。ServiceManager收到这个连接请求，从请求数据包里获得Binder的名字，在查找表里找到该名字对应的条目，从条目中取出Binder的引用，将该引用作为回复发送给发起请求的Client。从面向对象的角度，这个Binder对象现在有了两个引用：一个位于ServiceManager中，一个位于发起请求的Client中。如果接下来有更多的Client请求该Binder，系统中就会有更多的引用指向该Binder，就象java里一个对象存在多个引用一样。而且类似的这些指向Binder的引用是强类型，从而确保只要有引用Binder实体就不会被释放掉。通过以上过程可以看出，ServiceManager象个火车票代售点，收集了所有火车的车票，可以通过它购买到乘坐各趟火车的票-得到某个Binder的引用。

#### 匿名Binder
并不是所有Binder都需要注册给SMgr广而告之的。Server端可以通过已经建立的Binder连接将创建的Binder实体传给Client，当然这条已经建立的Binder连接必须是通过实名Binder实现。如果我们是从事application开发，跨进程的自己手写AIDL文件，或者相同进程的bindService自己添加一个继承Binder的子类，那么这个Binder没有向ServiceManager注册名字，所以是个匿名Binder。Client将会收到这个匿名Binder的引用，通过这个引用向位于Server中的实体发送请求。匿名Binder为通信双方建立一条私密通道，只要Server没有把匿名Binder发给别的进程，别的进程就无法通过穷举或猜测等任何方式获得该Binder的引用，向该Binder发送请求。

这里很重要，了解了这个知识我们才会知道，AIDL，以及四大组件通信的时候用到的`ApplicationThread`，`IIntentReceiver`，`IServiceConnection`，`IContentProvider`，其实都是匿名Binder，依赖AMS这个实名Binder的匿名Binder。


待续...