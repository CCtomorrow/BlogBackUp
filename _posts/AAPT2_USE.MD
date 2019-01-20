---
title: 'AAPT2的妙用'
date: 2019-01-20 21:05:40
tags: [插件化]
categories: [Android,插件化]
---

在buildTools 28.0.0以前，aapt2自带了资源分区，通过–package-id参数指定。但是该分区只支持>0x7f的PP段，而在Android 7.0之前，是不支持>0x7f的PP段资源的，运行时会抛异常。但是当指定了一个<0x7f的PP段资源后，编译资源时却会报错。
```
error: invalid package ID 0x28. Must be in the range 0x7f-0xff..
```

**关于插件化为什么需要修改资源id，一定需要的么？可以看之前的一篇介绍文章。**
[Android的资源管理器的创建过程](http://www.qingyongai.com/2017/05/05/Android_Resource_Manager/)

<!-- more -->

目前市面上修改PP字段的方式就是两种，一种是修改arsc二进制文件，一种是修改aapt源码。

在buildTools 28.0.0之后，aapt2支持了<0x7f预留PP段分区的功能，只需要指定一个参数即可，`--allow-reserved-package-id`

buildTools 28.0.0开始的aapt2还新增了一个参数，`--warn-manifest-validation`，这个参数如果你在Manifest里面定义了安卓识别不了的节点不会报错。

使用:
``` groovy
android {
    //记得保证这个在28.0.0以后
    buildToolsVersion "28.0.3"
    aaptOptions {
        additionalParameters "--warn-manifest-validation", "--allow-reserved-package-id", "--package-id", "0x15"
    }
}
```

这个个人感觉意义还是挺大的，做开源项目可能需要兼容考虑一下buildTools在28.0.0之前的情况，但是作为一个团队来说，可以要求都使用buildTools在28.0.0之后的。
