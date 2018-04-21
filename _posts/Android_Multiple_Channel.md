---
title: '多渠道打包，生成不同包名的包'
date: 2016-04-21 22:58:40
tags: [Gradle]
categories: [Android,Gradle]
---

### package name:AndroidManifest文件中的包名
也可以称之为资源包名，这个包名和真正安装到用户手机上面的apk的包名可能会不是同一个，这个包名仅用来命名资源类的包。
```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
          package="com.cctomorrow.plat"/>
```
对应的module的R.java文件的package即:
```java
package com.mmc.plat.pay;
public final class R {
}
```

### applicationId:Android真正的包名 
市场上面唯一的标识即是这个，这个一般我们在项目的build.gradle文件里面配置。
```groovy
android {
    defaultConfig {
        applicationId "com.cctomorrow.plat.beta"
    }
}
```
那如果我们没有配置的情况下，会默认为AndroidManifest文件中的package了。

### manifestPlaceholders:AndroidManifest文件中的变量表示
通过${PlaceHolder}表示PlaceHolder是可以被赋值的变量，如友盟统计中的渠道。
```xml
<meta-data
    android:name="UMENG_CHANNEL"
    android:value="${channel}"/>
```
然后里面的值，我们需要在build.gradle文件里面赋值。
```groovy
android {
    defaultConfig {
        manifestPlaceholders = [channel:"default"]
    }
}
```
当然不仅仅是渠道，Manifest里面的很多东西都可以这样赋值的。

### signingConfigs:签名配置
```groovy
signingConfigs {
    debug {
        storeFile file(DEBUG_STOREFILE)
    }
    beta {
        storeFile file(DEBUG_STOREFILE)
    }
    release {
        storeFile file(RELEASE_STOREFILE)
        storePassword RELEASE_STOREPASSWORD
        keyAlias RELEASE_KEY_ALIAS
        keyPassword RELEASE_KEY_PASSWORD
    }
}
```
这里面的简写是因为我把真正的key配置到了gradle.properties文件里面了
```
#调试签名路径
DEBUG_STOREFILE=/Users/qingyong/Dev/keystore/debug.keystore
```
当然其实一般情况下，我们的gradle.properties文件会上传到git，其实这样并不好，所以我们可以在用户目录下面新建全局的gradle.properties，然后把配置放到里面。
还有这里可以配置自己想配置的签名，比如beta，但是感觉签名配置那么多好像没有什么意义。
![全局gradle.properties文件](/images/global_gradle_properties.png)

### buildTypes:用于生成不同编译类型的包
```groovy
 buildTypes {
        debug {
            debuggable true
            minifyEnabled false
            signingConfig signingConfigs.debug
        }
        beta {
            debuggable true
            minifyEnabled false
            signingConfig signingConfigs.beta
        }
        release {
            debuggable false
            minifyEnabled true
            signingConfig signingConfigs.release
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
            // 下面代码只是用来改变最后生成的apk的名字
            applicationVariants.all { variant ->
                variant.outputs.all { output ->
                    if (output.outputFile != null && output.outputFile.name.endsWith('.apk') && 'release'.equals(variant.buildType.name)) {
                        // 2018_02_02_app_cn_V1.0.0_xiaomi_release
                        def time = new Date().format('yyyy_MM_dd')
                        def name = "app_cn"
                        def version = "V" + variant.versionName
                        def channel = variant.flavorName
                        def type = variant.buildType.name
                        def baseName = time + "_" + name + "_" + version + "_"
                        def finalName
                        // 不一定配置了渠道，所以这里要做处理
                        if (channel.length() > 0) {
                            finalName = baseName + channel + "_" + type
                        } else {
                            finalName = baseName + type
                        }
                        outputFileName = finalName + ".apk"
                    }
                }
            }
        }
    }
```

### productFlavors:用于生成不同渠道的包 
```groovy
flavorDimensions "default"
productFlavors {
    dev {
        applicationId "com.cctomorrow.plat.dev"
        versionCode 180520
        versionName "18.05.20"
        dimension "default"
    }
    xiaomi {
        dimension "default"
    }
    baidu {
        dimension "default"
    }
    wandoujia {
        dimension "default"
    }
    _360 {
        dimension "default"
    }
    vivo {
        dimension "default"
    }
}
// 这里需要这个配置是因为，我们上面虽然建立了渠道，但是并没有处理Manifest里面的meta-data信息。
productFlavors.all { flavor ->
    flavor.manifestPlaceholders = [channel: name] //动态地修改AndroidManifest中的渠道名
}
```
最后的代码即是处理下面的赋值。
```xml
<meta-data
    android:name="UMENG_CHANNEL"
    android:value="${channel}"/>
```

<!-- more -->

### 生成不同包名的包
上面1.6中，我们配置了不同的渠道，这样配置了不同的渠道之后其实会有个操作，如果在src下面建立了相同的文件夹，会去读取里面的信息，然后和main里面的信息作合并。一般的应用为了渠道的aso都会要求打出不同的包名的包的，这个我们就可以利用这一点来处理。
比如现在vivo渠道生成的包要和其他的渠道的不一样，并且还要在欢迎页面怼上vivo商店的logo。
![建立渠道文件夹](/images/create_channel_dir.png)
这个文件夹里面可以放置的内容和main里面可以放置的东西一样，源代码，assets，res，manifest文件，然后如果res里面放置了和main的res对应的目录一样的东西，这个里面的会覆盖main的res里面的东西。

这里我们需要改应用的名称，即application的label,即可如下操作。
![修改应用名称](/images/rename_app_name.png)
这样即可，到时候资源工具会把这个和main里面的manifest作合并。

如果要修改或者覆盖main里面的其他的东西，比如layout文件，图片文件，新建对应的相同的文件做覆盖即可。

如果要覆盖Application也是一样的，但是记得如下操作。
```xml
<application
    android:name="com.cctomorrow.plat.vivo.Application"
    android:label="@string/app_name_vivo"
    tools:replace="android:label,android:name">
</application>
```