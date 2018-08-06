---
title: 'Replugin插件通信不得不了解的知识'
date: 2018-08-05 22:13:00
tags: [插件化]
categories: [Android,插件化]
---
### 前言
[`Replugin`](https://github.com/Qihoo360/RePlugin)其实比较强调的是各个插件以及插件与宿主之间不共用同一套代码资源，因为这样不好维护，特别是涉及到应用版本升级就更麻烦。
但是目前这样的需求其实是比较多的，因为这样才不会说一个插件的大小很大。
这篇文章探讨几个方面，一个是插件使用宿主的代码，二是插件与宿主之间的通信，三是资源相关的问题。
本文代码:[RepluginSample](https://github.com/CCtomorrow/RepluginSample)

### 插件使用宿主的代码
#### 反射方式
Replugin推荐的方式，Replugin不建议直接使用宿主的代码，而是建议通过反射的方式调用宿主的代码。
![项目概览](/images/replugin_project_show.png)
我的项目大概是这样子的，具体可以看Github上面的配置。然后我宿主里面有这样的一个类。
![dateutil](/images/replugin_dateutil.png)
我想要在插件里面使用到这个类的话，可以通过反射的方式直接调用到这个类。
宿主要开启:
```java
RePluginConfig c = new RePluginConfig();
//允许“插件使用宿主类”。默认为“关闭”
c.setUseHostClassIfNotFound(true);
```
插件里面使用:
```java
private void initTime() {
    String t = "现在时间是:";
    ClassLoader c = RePlugin.getHostClassLoader();
    if (c != null) {
        try {
            Class cz = c.loadClass("com.cctomo.hostapp.util.DateUtil");
            Method m = cz.getDeclaredMethod("getNow");
            String o = (String) m.invoke(null);
            t += o;
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    timeInfo.setText(t);
}
```
目前调试是直接使用内置插件调试的，所以写一个任务直接把生成的debug的apk拷贝到宿主的assets里面更方便。在插件的build.gradle文件里面加如下的代码即可。
```groovy
task copyToHost(type: Copy, dependsOn: ['assembleDebug']) {
    from('build/outputs/apk/debug/pluginapp-debug.apk')
    into('../hostapp/src/main/assets/plugins')
    rename('pluginapp-debug.apk', 'pluginapp_one.jar')
}
copyToHost.group = 'copy'
```
这里就不得不感叹groovy确实有时候很方便。
然后这里提一个`task.group`，我们给一个任务指定了这个属性，可以让任务归属到某一个组里面，然后会更方便我们在图形界面的as上面使用。
![task.group](/images/replugin_task_group.png)

<!-- more -->

#### 直接使用的形式
上面介绍的反射使用对工具类这种当然是很方便，但是如果需要比较多的操作，那么反射写起来就很痛苦了，所以虽然不建议直接使用，但是这个需求量并不小。我认为可以在直接使用的情况下，使用接口约束，这样可以减小由于版本的变迁而导致的维护越来越复杂的问题。
![用户模块](/images/replugin_user_module.png)
我们新建用户模块，然后宿主直接引用此模块，插件使用`compileOnly project(':user')`的形式引用。
注意:这里要说一个问题，app类型的module是不支持providedAar的(3.X以后provided被compileOnly和runtimeOnly替代)，所以要能这样使用要用一个别人已经写好的库。
就是这个库:[AndroidGradlePluginCompat](https://github.com/lizhangqu/AndroidGradlePluginCompat).
然后在插件app的build.gradle里面配置一下:
```java
apply plugin: 'com.android.application'
apply plugin: 'com.android.compat'
//启用providedAar功能
providedAarCompat()
android {
    //...
}
```
然后在插件里面直接使用即可。
```java
private void initUser() {
    String u = "用户信息:\n";
    UserModel userModel = new UserModel("嘻嘻", 18);
    u += userModel.toString();
    userInfo.setText(u);
}
```