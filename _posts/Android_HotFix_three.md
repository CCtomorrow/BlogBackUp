---
title: '热修复探究（三）'
date: 2016-08-10 23:43:40
tags: [HotFix]
categories: [Android,HotFix]
---

这篇文章将会分析一个开源的热更新框架的实现。这里只分析使用QQ空间方案的实现的框架，现在开源的这种框架大致如下:
*   [https://github.com/jasonross/Nuwa](https://github.com/jasonross/Nuwa)
*   [https://github.com/bunnyblue/DroidFix](https://github.com/bunnyblue/DroidFix)
*   [https://github.com/dodola/RocooFix](https://github.com/dodola/RocooFix)
前面两个框架现在基本没有更新了，最后一个框架现在是一直在更新的，我们就来分析最后一个，这个框架的交流群是561394234，这个群很活跃，不少人都在交流，建议加群，总体来说这个框架应该也会存在兼容性问题，但是这并不妨碍我们研究它。

这个框架实现是使用的groovy编写的gradle插件来实现的，关于gradle插件的编写，以及推荐文章，我在上篇文章[热修复探究（二）]里面有推荐，这里不多说。
![app插件](/images/gradle_app_plugin.png)
我们一般对于可运行的app使用这个插件，对于lib的module使用下面的插件。
![lib插件](/images/gradle_library_plugin.png)

我们可以使用这两个插件，是因为，我们在根目录的build.gradle里面的buildscript有配置。类似下面:
![根目录配置](/images/gradle_root_classpath.png)

所以可以想象，在`com.android.tools.build:gradle:2.1.2`，这个jar包里面必然存在这两个插件。去看源代码:
![插件列表](/images/gradle_plugin_list.png)

看了前面的文章的应该知道，我们写的`apply plugin: 'com.android.application`，里面的参数就是插件在META-INF/gradle-plugins里面的名字决定的。
可以清楚的看到，我们经常使用的两个插件，可运行的插件application和library，前面会多几个如application.properties，进去看代码:
![对比](/images/gradle_app_plugin_duibi.png)
其实会发现他们指向同一个插件，并且都是可运行module使用的插件，那这里如果用过早期gradle插件的用户就会知道，以前是`apply plugin: 'android'`，哈哈。

这里继续看源代码会发现很多知识，比如可运行的module对应插件类为AppPlugin，参数类为AppExtension，这个就是用来读取android闭包里面的数据的类，对应的lib的module对应的插件类为LibraryPlugin，参数类为LibraryExtension。

这里有个知识点必须了解，一般的 Java 项目中有一组 task 用于协同处理并最终生成一个输出。相比之下在 Android 项目中这就有点复杂。因为 Android 项目中会有大量相同的 task，并且它们的名字基于_Build Types_ 和 _Product Flavor_ 生成。
所以呢，Android针对这种情况引入了一个属性**Variants**。
这里有详细介绍，非常值得一看:
[操作task](https://chaosleong.gitbooks.io/gradle-for-android/content/advanced_build_customization/manipulating_tasks.html)
![介绍](/images/gradle_variants.png)
怎么拿到这个属性呢，看源代码:
![获取variants](/images/gradle_get_variants.png)
如果在android的节点下面可以直接使用applicationVariants，这个变量，不然则要project.android.applicationVariants，这样才能使用到了。

后面介绍RocooFix这个库

![主代码](/images/rocoofix.png)
分析:
34行，声明当前android工程的所有的变量；
35行，拿到所有应用到当前Project的Plugins，返回的对象是PluginContainer，我们可以去官网看它的API，解释如下
*   A `PluginContainer` is used to manage a set of [`Plugin`](https://docs.gradle.org/current/javadoc/org/gradle/api/Plugin.html "interface in org.gradle.api") instances applied to a particular project.

    Plugins can be specified using either an id or type. The id of a plugin is specified using a META-INF/gradle-plugins/${id}.properties classpath resource.
上面的意思是这是用来管理Plugin的，Plugins可以通过type或者id来指定，id是我们编写gradle插件时候的，META-INF下面的文件的名称，这个在创建插件项目的时候我们应该了解过。
我们来看`PluginContainer`的API:
![是否存在插件API](/images/gradle_has_plugin.png)
这两个方法呢，我们可以判断当前的Project是否含有指定的插件，由于当前的HotFix插件是针对Android应用的，不是别的如Android库啊，java项目的，我们知道插件的类型就是AppPlugin，所以填就好了。

待续。。。