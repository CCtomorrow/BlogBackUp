﻿---
title: 'Activity插件化(三)'
date: 2017-10-22 21:35:40
tags: [插件化]
categories: [Android,插件化]
---
上面一篇文章已经实现了Intent解析和targetClass的还原工作，这篇文章会来说说:
- 插件apk的解析
- ClassLoader的问题
- 资源的问题
- 插件的下载，加载机制的问题


### 插件apk的解析
说到这个我们很容易想到一般的apk的安装过程也是需要解析apk的信息的，下面贴一篇比较好的文章。
[Android APK安装过程分析](http://www.jianshu.com/p/953475cea991)

图片取自上面的博客，apk的安装过程与pms有很大的关系，很多操作都是由pms完成的，有兴趣的可以去了解。

<!-- more -->

```java
private void installPackageLI(InstallArgs args, PackageInstalledInfo res) {
    final int installFlags = args.installFlags;
    final String installerPackageName = args.installerPackageName;
    final String volumeUuid = args.volumeUuid;
    final File tmpPackageFile = new File(args.getCodePath());
    final boolean forwardLocked = ((installFlags & PackageManager.INSTALL_FORWARD_LOCK) != 0);
    final boolean onExternal = (((installFlags & PackageManager.INSTALL_EXTERNAL) != 0)
            || (args.volumeUuid != null));
    boolean replace = false;
    int scanFlags = SCAN_NEW_INSTALL | SCAN_UPDATE_SIGNATURE;
    res.returnCode = PackageManager.INSTALL_SUCCEEDED;
    final int parseFlags = mDefParseFlags | PackageParser.PARSE_CHATTY
            | (forwardLocked ? PackageParser.PARSE_FORWARD_LOCK : 0)
            | (onExternal ? PackageParser.PARSE_EXTERNAL_STORAGE : 0);
    PackageParser pp = new PackageParser();
    pp.setSeparateProcesses(mSeparateProcesses);
    pp.setDisplayMetrics(mMetrics);
    final PackageParser.Package pkg;
    try {
        pkg = pp.parsePackage(tmpPackageFile, parseFlags);
    } catch (PackageParserException e) {
        res.setError("Failed parse during installPackageLI", e);
        return;
    }
}
```
解析apk文件调用了`PackageParser`的`parsePackage`方法。
现在一般的插件话框架也是这样做的，解析apk也是通过调用这个方法，当然除了这个，还有一些开源的解析apk的框架例如:[apk-parser](https://github.com/shwenzhang/apk-parser)

这里要说明的是PackageParser这个类在不同的安卓SDK版本里面有不少的改动，所以需要做兼容。
这里比较一下滴滴的VirtualAPK和DroidPlugin的做法，他们的做法基本一样，比较不同的是VirtualAPK框架把安卓framework把安卓里面插件化需要的类提取出来了，作为一个lib，插件化框架VirtualAPK provide的形式依赖这个库，就可以直接调用里面的方法，有效避免了反射带来的性能方面的损失。
![droidplugin_package_parser](/images/droidplugin_package_parser.png)
DroidPlugin里面的兼容，下图是VirtualAPK里面的做法，抽出framework里面的类，做成一个库，插件化框架provide形式依赖。
![virtualapk_package_parser](/images/virtualapk_package_parser.png)
```java
public final class PackageParserCompat {

    /**
    * 解析插件apk
    *
    * @param context context
    * @param apk    插件apk位置，必须是apk文件
    * @param flags  flag
    * @return {@link PackageParser.Package}
    * @throws PackageParser.PackageParserException
    */
    public static final PackageParser.Package parsePackage(final Context context, final File apk, final int flags)
            throws PackageParser.PackageParserException {
        if (Build.VERSION.SDK_INT >= 24) {
            return PackageParserV24.parsePackage(context, apk, flags);
        } else if (Build.VERSION.SDK_INT >= 21) {
            return PackageParserLollipop.parsePackage(context, apk, flags);
        } else {
            return PackageParserLegacy.parsePackage(context, apk, flags);
        }
    }

    /**
    * 7.0及以后的兼容处理
    */
    private static final class PackageParserV24 {
        static final PackageParser.Package parsePackage(Context context, File apk, int flags)
                throws PackageParser.PackageParserException {
            PackageParser parser = new PackageParser();
            PackageParser.Package pkg = parser.parsePackage(apk, flags);
            ReflectUtil.invokeNoException(PackageParser.class, null, "collectCertificates",
                    new Class[]{PackageParser.Package.class, int.class}, pkg, flags);
            return pkg;
        }
    }

    /**
    * 5.0,5.1,6.0的处理
    */
    private static final class PackageParserLollipop {
        static final PackageParser.Package parsePackage(final Context context, final File apk, final int flags)
                throws PackageParser.PackageParserException {
            PackageParser parser = new PackageParser();
            PackageParser.Package pkg = parser.parsePackage(apk, flags);
            try {
                parser.collectCertificates(pkg, flags);
            } catch (Throwable e) {
                // ignored
            }
            return pkg;
        }
    }

    /**
    * 低版本的处理
    */
    private static final class PackageParserLegacy {
        static final PackageParser.Package parsePackage(Context context, File apk, int flags) {
            PackageParser parser = new PackageParser(apk.getAbsolutePath());
            PackageParser.Package pkg = parser.parsePackage(apk, apk.getAbsolutePath(), context.getResources()
                    .getDisplayMetrics(), flags);
            ReflectUtil.invokeNoException(PackageParser.class, parser, "collectCertificates",
                    new Class[]{PackageParser.Package.class, int.class}, pkg, flags);
            return pkg;
        }
    }

}
```
代码处理，对了，我最近在给VirtualAPK框架添加注释，代码在这里:[VirtualAPK](https://github.com/elderSister/VirtualAPK)


### ClassLoader的问题
ClassLoader问题之前已经提过了，插件是不是能使用宿主的类，如果要能使用的话，插件是需要继承宿主的ClassLoader的，宿主是不是能加载到插件的类，如果能，插件的dex是要放到宿主的dex数组里面滴。
VirtualAPK在处理这个问题的时候分了两种情况，一种是宿主有价值插件类的能力，一种是没有，实现方式那是相当的简单。(PS:插件是一直可以加载到宿主类的)
```java
/**
    * 创建插件ClassLoader
    *
    * @param context host的classLoader
    * @param apk    插件文件位置
    * @param libsDir native lib的文件夹，按照他的写法和files，cache同级的app_valibs目录
    * @param parent  host的ClassLoader
    * @return 插件ClassLoader
    */
private static ClassLoader createClassLoader(Context context, File apk, File libsDir, ClassLoader parent) {
    // data/data/name/app_dex
    File dexOutputDir = context.getDir(Constants.OPTIMIZE_DIR, Context.MODE_PRIVATE);
    String dexOutputPath = dexOutputDir.getAbsolutePath();
    DexClassLoader loader = new DexClassLoader(apk.getAbsolutePath(), dexOutputPath, libsDir.getAbsolutePath(), parent);
    if (Constants.COMBINE_CLASSLOADER) {
        try {
            DexUtil.insertDex(loader);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    return loader;
}
```
这里的处理，就很简单了，如果宿主有加载插件类的能力，那么，把插件的dex文件放到宿主的dex列表里面(至于为什么要这样做，可以了解一下安卓的BaseDexClassLoader)。


### 资源的问题
关于资源的问题和代码的问题差不多，如果想要共享资源，那么需要把插件的资源添加到宿主的资源里面，当然也可以插件就自己管理自己的资源，每个插件一个资源管理器。

这里需要注意的是如果把插件的资源放到宿州的资源里面那么需要解决一个问题，文件描述的明白点就是，宿主是一个正常的apk文件，资源id是0x7f开头，插件也是apk文件，资源id也是0x7f开头，那么这样的话，插件和宿主的资源id不是就存在重复的可能了么，答案是肯定的，这种可能很大呢，为了解决这个问题也有两种解决方案:(这里只讨论修改aapt的方法)
1.修改aapt，让插件资源id不再以0x7f开头
2.使用gradle plugin插件修改资源产物.ap_里面的资源id，然后重新压缩回去

VirtualAPK在处理这个问题上也是分两种情况，一种是把插件的资源添加到宿州的资源里面，一种是插件自己管理自己的资源，当然实现的代码依旧非常简单。
```java
/**
    * 创建插件的AssetManager
    *
    * @param context 宿主context
    * @param apk    插件apk文件
    * @return 插件AssetManager
    */
private static AssetManager createAssetManager(Context context, File apk) {
    try {
        AssetManager am = AssetManager.class.newInstance();
        ReflectUtil.invoke(AssetManager.class, am, "addAssetPath", apk.getAbsolutePath());
        return am;
    } catch (Exception e) {
        e.printStackTrace();
        return null;
    }
}

/**
    * 创建插件的资源管理器
    *
    * @param context 宿主host
    * @param apk    插件apk文件
    * @return 插件Resources
    */
@WorkerThread
private static Resources createResources(Context context, File apk) {
    if (Constants.COMBINE_RESOURCES) {
        Resources resources = ResourcesManager.createResources(context, apk.getAbsolutePath());
        ResourcesManager.hookResources(context, resources);
        return resources;
    } else {
        Resources hostResources = context.getResources();
        AssetManager assetManager = createAssetManager(context, apk);
        return new Resources(assetManager, hostResources.getDisplayMetrics(), hostResources.getConfiguration());
    }
}
```
这里就分两种情况，如果资源要合并的话，就把插件资源放入宿主里面，不然插件就使用自己的资源。


### 插件的下载，加载机制的问题
插件的下载要考虑一些问题，首先是后台的设计，根据选择的插件化框架的不同，有不同的处理。
这些问题是我的一些关于插件化问题的一些思考。
![plugin_questions](/images/plugin_questions.png)
插件的后台也要考虑到这些问题。
插件的加载，插件加载是需要释放dex优化dex文件的，那么这个过程的管理升级的一些问题。


### 后续
后续就不说Activity插件化相关的事情了，等把代码补充完成之后，就开始其他三大组件的插件化讲解了。
关于代码:整理中

其实这个系统的插件化基本都是基于VirtualAPK讲解的，个人更喜欢360团队的插件化框架Replugin，等这系列结束之后会去研究他。
个人打算写一个插件化的后台[PluginServer](https://github.com/elderSister/PluginServer)，这个后台是为Replugin服务的，当然，也会考虑到我上面说的一些问题，版本的问题，争取做一个通用的插件化平台的后台。