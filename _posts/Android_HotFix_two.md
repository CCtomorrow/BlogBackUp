---
title: '热修复探究（二）'
date: 2016-08-02 23:43:40
tags: [HotFix]
categories: [Android,HotFix]
---

这次继续介绍热修复相关的知识，前一篇文章有提到这篇会讲补丁文件生成和CLASS_ISPREVERIFIED问题的相关操作，那其实这个两个实现现在主流的实现方式是通过groovy编写Gradle插件来实现的，所以呢，很有必要研究一下gradle和groovy相关的知识。不过我不会介绍groovy语言以及怎么使用Android Studio来开发Gradle的脚本这两个方面的知识，但是我会推荐几篇文章。

这篇文章会着重介绍gradle相关的知识，所以估计两篇文章很难写完了，最少也是三篇。

### groovy相关的文章推荐
[http://jiajixin.cn/2015/08/07/gradle-android/](http://jiajixin.cn/2015/08/07/gradle-android/)
这篇文章是讲Gradle相关的知识，但是里面有提到需要学习的groovy和gradle的资源，值得一看。
[http://www.ibm.com/developerworks/cn/education/java/j-groovy/j-groovy.html](http://www.ibm.com/developerworks/cn/education/java/j-groovy/j-groovy.html)
IBM社区的精通groovy，浅显易懂。
[http://blog.csdn.net/innost/article/details/48228651](http://blog.csdn.net/innost/article/details/48228651)
深入理解Android之gradle。

我们了解groovy的基本语法也就够用了。

### 使用Android Studio开发gradle插件文章推荐
[http://blog.csdn.net/sbsujjbcy/article/details/50782830](http://blog.csdn.net/sbsujjbcy/article/details/50782830)
这篇文章介绍的很详细，同时这个博主还写了一篇关于热修复的文章，[Android 热修复使用Gradle Plugin1.5改造Nuwa插件](http://blog.csdn.net/sbsujjbcy/article/details/50839263)，大家可以去看一下。
我自己也写了一个关于使用Android Studio创建并开发gradle插件的Demo，在我的Github上面[https://github.com/qingyongai/GradlePluginDemo](https://github.com/qingyongai/GradlePluginDemo)。可以下载下来运行，参考。

<!-- more -->

### 闭包
还是说一下闭包，也方便自己以后复习。
闭包，英文叫Closure。闭包，是一种数据类型，它代表了一段可执行的代码。
```
def closure = {//闭包是一段代码，所以需要用花括号括起来
    String p, int p -> //这个箭头很关键。箭头前面是参数定义，箭头后面是代码
    println"this is code" // 这是代码，最后一句是返回值
   //也可以使用return，和Groovy中普通函数一样
}
```

Closeure定义格式:
```
def xxx = {paramters -> code}

def xxx = {无参数,纯code} 这种情况不需要->符号
```

调用:
```
closure.call("string", 100)

closure("string", 100)
```

** 注意:如果闭包没定义参数的话，则隐含有一个参数，这个参数名字叫 it，和this 的作用类似。it代表闭包的参数。 **

无参闭包
```
def noParamClosure = { -> true }
```
这个时候，我们就不能给 noParamClosure 传参数了！

** 使用注意: **
1.Groovy中，当函数的最后一个参数是闭包的话，可以省略圆括号。
```
def testClosure(int a, String b, Closure closure){
      // dosomething
     closure() //调用闭包
}
那么调用的时候，就可以免括号！
testClosure (4, "test", {
   println "i am in closure"
} )  //括号可以不写
```

2.确定groovy的参数
多练，看API文档

### gradle知识
gradle是一个编程框架，相当于对groovy封装的一个框架。
[https://docs.gradle.org/current/dsl/](https://docs.gradle.org/current/dsl/)
这个是Gradle的API文档，没事多翻翻。

Gradle中，每一个待编译的工程都叫一个Project。每一个Project在构建的时候都包含一系列的Task。
在Android开发的时候，每个Library和每个App都是一个Project。我们一般建立的Android工程都是Multi-Projects Build的。
配置就是会在Root Project的根目录放置一个build.gradle文件来配置其他子Project，这个build.gradle可以没有。
还有一个settings.gradle，配置这个multiprojects包含多少个子Project。

一些好用的命令:
```
gradlew projects 查看当前工程有多少Project
```
![gradle peojects](/images/gradlew_task_peojects.png)

gradle tasks 当前工程包含哪些task
```
gradle project-path:tasks 查看某个Project的tasks
```
![gradle tasks](/images/gradlew_task_tasks.png)
这里这个图并没有列完。

```
gradle task name 执行某个任务
```
![gradle task name](/images/gradlew_task_clean.png)

gradle工作流程:
![gradle work](/images/gradlew_work.png)
图片取自[http://blog.csdn.net/innost/article/details/48228651](http://blog.csdn.net/innost/article/details/48228651)

过程:
1.初始化阶段，对multi-project build而言，就是执行settings.gradle
2.Configration阶段的目标是解析每个project中的build.gradle，Configuration阶段完了后，整个build的project以及内部的Task关系就确定了，就是task之间的依赖也解析完成了，知道先执行哪个，后执行哪个了
3.执行任务了，任务执行完后，我们还可以加Hook

Gradle主要有三种对象:
Gradle:当前项目的gradle环境信息，可以拿到当前使用的gradle工具的信息。
![gradle环境](/images/gradlew_env.png)

Project:每一个build.gradle文件都会转换成一个Project对象。
嗯，这个区看一下文档比较好，英语也不是太难。
[https://docs.gradle.org/current/dsl/org.gradle.api.Project.html](https://docs.gradle.org/current/dsl/org.gradle.api.Project.html)
Project包含若干Tasks，Project对应具体的工程，需要为Project加载所需要的插件，比如为Java工程加载Java插件。其实，一个Project包含多少Task往往是插件决定的。

Setting:setting.gradle文件就是了。

#### Task
Task是Gradle中的一种数据类型，它代表了一些要执行或者要干的工作。不同的插件可以添加不同的Task。每一个Task都需要和一个Project关联。
Task是和Project关联的，所以，我们要利用Project的task函数来创建一个Task。

Android Studio创建项目一般目录结构是这样的:
![初始化](/images/android_project_stuct.png)

**setting.gradle解析**
首先被解析的是setting.gradle，该文件将会在初始化时期执行，并且定义了哪一个模块将会被构建。举个例子，上述setting.gradle包含了app模块，setting.gradle是针对多模块操作的，所以单独的模块工程完全可以删除掉该文件。在这之后，Gradle会为我们创建一个Setting对象。

**根目录的build.gradle**
![根目录](/images/root_build_gradle.png)
buildscript方法是定义了全局的相关属性，repositories定义了jcenter作为仓库。一个仓库代表着你的依赖包的来源，例如maven仓库。dependencies用来定义构建过程。这意味着你不应该在该方法体内定义子模块的依赖包，你仅仅需要定义默认的Android插件就可以了，因为该插件可以让你执行相关Android的tasks。

allprojects方法可以用来定义各个模块的默认属性，你可以不仅仅局限于默认的配置，你可以自己创造tasks在allprojects方法体内，这些tasks将会在所有模块中可见。

**模块内的build.gradle**
![模块的gradle文件](/images/module_build_gradle.png)
模块内的gradle文件只对该模块起作用，而且其可以重写任何的参数来自于根目录下的gradle文件。

**基本的tasks**
android插件依赖于Java插件，而Java插件依赖于base插件。
base插件有基本的tasks生命周期和一些通用的属性。
base插件定义了例如assemble和clean任务，Java插件定义了check和build任务，这两个任务不在base插件中定义。
这些tasks的约定含义：
* assemble: 集合所有的output
* clean: 清除所有的output
* check: 执行所有的checks检查，通常是unit测试和instrumentation测试
* build: 执行所有的assemble和check
Java插件同时也添加了source sets的概念。

**Android tasks**
android插件继承了这些基本tasks,并且实现了他们自己的行为：
* assemble 针对每个版本创建一个apk
* clean 删除所有的构建任务，包含apk文件
* check 执行Lint检查并且能够在Lint检测到错误后停止执行脚本
* build 执行assemble和check
默认情况下assemble tasks定义了assembleDebug和assembleRelease，当然你还可以定义更多构建版本。除了这些tasks,android 插件也提供了一些新的tasks:
* connectedCheck 在测试机上执行所有测试任务
* deviceCheck 执行所有的测试在远程设备上
* installDebug和installRelease 在设备上安装一个特殊的版本
* 所有的install task对应有uninstall 任务
build task依赖于check任务，但是不依赖于connectedCheck或者deviceCheck，执行check任务的使用Lint会产生一些相关文件，这些报告可以在app/build/outputs中查看

gradle插件开发的话，其实需要看gradle的文档以及Android gradle工具的文档，这样才知道里面有什么属性，什么时候去拦截什么方法。