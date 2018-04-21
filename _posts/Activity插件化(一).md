---
title: 'Activity插件化(一)'
date: 2017-08-22 21:35:40
tags: [插件化]
categories: [Android,插件化]
---
#### 首先说明一下:
本文的编写借鉴参考了大量的文章，有的可能是直接把文字拷贝过来的，我会在文中给出链接，如果有侵权，请联系我删除，谢谢。

我们知道，启动Activity可以是通过Activity或者通过Context，这两种启动没有太大的区别，最终都是调用` Instrumentation`的方法来启动的，当然说是这样说，其实还是有区别滴，Activity的startActivity()方法可使用默认配置的LAUNCH FLAG，而Context的startActivity()须包含`FLAG_ACTIVITY_NEW_TASK`的LAUNCH FLAG，原因是该Context可能没有现存的任务栈供新建的Activity使用，必须显式指定生成一个自己单独的任务栈。

Activity启动发起后，通过Binder，最终由system_server进程中的AMS(ActivityManagerService)启动的。这里不打算说Activity的启动过程了，因为套路就是那样，太多的博客文章也分析过了过程。想看启动过程的可以去看看下面的文章:
[startActivity启动过程分析](http://gityuan.com/2016/03/12/start-activity/)
[Activity启动过程全解析](http://blog.csdn.net/zhaokaiqiang1992/article/details/49428287)

﻿<!-- more -->

如果对上面的文章都不满意，或者还是有细节问题没搞清楚，可以这样:
![](http://dd089a5b.wiz03.com/share/resources/b6b18d42-e833-4f63-82fe-c6213e78cba1/index_files/72736309.png)
嗯，都系你想要滴。
这里直接贴出别人文章里面画的时序图了。
![](http://dd089a5b.wiz03.com/share/resources/b6b18d42-e833-4f63-82fe-c6213e78cba1/index_files/72815267.png)
说明:此图出处为[startActivity启动过程分析](http://gityuan.com/2016/03/12/start-activity/)

下面列出一些重要类:
- ActivityManagerService，简称AMS，服务端对象处于system_server进程，负责系统中所有Activity的生命周期
- ActivityThread，App的真正入口。当开启App之后，会调用main()开始运行，开启消息循环队列，这就是传说中的UI线程或者叫主线程。与ActivityManagerService配合，一起完成Activity的管理工作
- ApplicationThread，用来实现ActivityManagerService与ActivityThread之间的交互。在ActivityManagerService需要管理相关Application中的Activity的生命周期时，通过ApplicationThread的代理对象与ActivityThread通讯，因为App和AMS通信，App是客户端，AMS所在进程为服务端，这个时候一般是客户端调用服务端的方法，但是这个是单向的，如果服务端要调用客户端怎么办呢，通过ApplicationThread，这个时候App是服务端，AMS所在system_server进程为客户端。
- ApplicationThreadProxy，是ApplicationThread在服务器端的代理，负责和App进程的服务端对象ApplicationThread通讯。AMS就是通过该代理与ActivityThread进行通信的
- Instrumentation，每一个应用程序只有一个Instrumentation对象，每个Activity内都有一个对该对象的引用。Instrumentation可以理解为应用进程的管家，ActivityThread要创建或暂停某个Activity时，都需要通过Instrumentation来进行具体的操作。
- ActivityStack，Activity在AMS的栈管理，用来记录已经启动的Activity的先后关系，状态信息等。通过ActivityStack决定是否需要启动新的进程。
- ActivityRecord，ActivityStack的管理对象，每个Activity在AMS对应一个ActivityRecord，来记录Activity的状态以及其他的管理信息。其实就是服务器端的Activity对象的映像。
- TaskRecord，AMS抽象出来的一个“任务”的概念，是记录ActivityRecord的栈，一个“Task”包含若干个ActivityRecord。AMS用TaskRecord确保Activity启动和退出的顺序。如果你清楚Activity的4种launchMode，那么对这个概念应该不陌生。

下面说几个问题:
#### 1.启动`Activity`为什么这么复杂，需要跨进程?
一个原因是安卓的四大组件设计的都是允许某个组件运行在一个单独的进程中的，安卓里面所有的App进程都是`Zygote`进程fork出来的(*你不要想着自己创建进程，你创建出来的进程，他需要的一些系统资源你怎么给*)，如果我们的`Activity`组件配置了新的进程，是需要`Zygote`进程做事的，这就是一个跨进程了吧。这里说一下组件配置进程的方式。
一般是通过在`AndroidManifest.xml`中`android:process`属性来实现的。
当android:process属性值以”:”开头，则代表该进程是私有的，只有该App可以使用，其他应用无法访问；
当android:process属性值不以”:“开头，则代表的是全局型进程，但这种情况需要注意的是进程名必须至少包含“.”字符。

另一个原因是`Activity`的生命周期其实是由`system_server`进程中的`ActivityManagerService(AMS)`管理的，除了onCreate是在new出来之后就本进程调用外，其余的都是AMS管理的。我们看`IActivityManager`接口就知道。
```
public interface IActivityManager extends IInterface {
    public void finishSubActivity(IBinder token, String resultWho, int requestCode) throws RemoteException;
    public boolean finishActivityAffinity(IBinder token) throws RemoteException;
    public void finishVoiceTask(IVoiceInteractionSession session) throws RemoteException;
    public boolean releaseActivityInstance(IBinder token) throws RemoteException;
    public void releaseSomeActivities(IApplicationThread app) throws RemoteException;
    public boolean willActivityBeVisible(IBinder token) throws RemoteException;
    public Intent registerReceiver(IApplicationThread caller, String callerPackage,
            IIntentReceiver receiver, IntentFilter filter,
            String requiredPermission, int userId) throws RemoteException;
    public void unregisterReceiver(IIntentReceiver receiver) throws RemoteException;
    public int broadcastIntent(IApplicationThread caller, Intent intent,
            String resolvedType, IIntentReceiver resultTo, int resultCode,
            String resultData, Bundle map, String[] requiredPermissions,
            int appOp, Bundle options, boolean serialized, boolean sticky, int userId) throws RemoteException;
    public void unbroadcastIntent(IApplicationThread caller, Intent intent, int userId) throws RemoteException;
    public void finishReceiver(IBinder who, int resultCode, String resultData, Bundle map,
            boolean abortBroadcast, int flags) throws RemoteException;
    public void attachApplication(IApplicationThread app) throws RemoteException;
    public void activityResumed(IBinder token) throws RemoteException;
    public void activityIdle(IBinder token, Configuration config,
            boolean stopProfiling) throws RemoteException;
    public void activityPaused(IBinder token) throws RemoteException;
    public void activityStopped(IBinder token, Bundle state,
            PersistableBundle persistentState, CharSequence description) throws RemoteException;
    public void activitySlept(IBinder token) throws RemoteException;
    public void activityDestroyed(IBinder token) throws RemoteException;
}
```
为什么`Activity`的生命周期需要`system_server`来管理么，不是我的人生我做主么，这个问题大概想一下就知道，我们现在在使用一个App，停留在A界面并且正在播放小视频，突然有人来了，so赶紧按了Home键，这个时候切换进程回到了桌面Launcher进程，这个时候我们肯定是希望A界面的视频停止播放啊，这个时候如果是App自己管理生命，App根本不知道现在已经处于桌面了，所以很明显这一个简单的场景就知道`Activity`的生命周期自己回调管理是不存在的。


#### 2.`Activity`是怎么怎么跨进程和`ActivityManagerService`通信的?
这个答案是很明显是通过`Binder`的，但是具体`Binder`怎么通信的，这个要说起来估计一篇文章也远远说不完。我在这里一时半会也说不清，而且，我现在的描述和对`Binder`的理解也没有特别到位，所以这里只说Framework层`Binder`的使用。

Binder使用过程:
##### 2.1.制定协议接口
`Binder`是C/S架构的，对应着`Client`端和`Server`端。要使用`Binder`，首先我们要定一个协议，就是客户端和服务端需要做什么事情，这里对应到Java端就是定一个客户端和服务端通用的接口，这个借口需要实现`IInterface`这个空接口，为什么要实现这个接口呢，这个接口里面定义了一个方法用于返回`Binder`对象，这个对象用于`Binder`通信使用。
```
/**
* Base class for Binder interfaces.  When defining a new interface,
* you must derive it from IInterface.
*/
public interface IInterface
{
    /**
    * Retrieve the Binder object associated with this interface.
    * You must use this instead of a plain cast, so that proxy objects
    * can return the correct result.
    */
    public IBinder asBinder();
}
```
举例:这里直接拿`IApplicationThread`举例了，他是用于`system_server`进程来跨进程调用App方法，嗯，前面说的AMS是App进程跨进程调用`system_server`进程方法，刚好是相反滴，AIDL也是一样哒。
```
public interface IApplicationThread extends IInterface {
    void schedulePauseActivity(IBinder token, boolean finished, boolean userLeaving,
            int configChanges, boolean dontReport) throws RemoteException;
    void scheduleStopActivity(IBinder token, boolean showWindow,
            int configChanges) throws RemoteException;
    void scheduleWindowVisibility(IBinder token, boolean showWindow) throws RemoteException;
    void scheduleSleeping(IBinder token, boolean sleeping) throws RemoteException;
    void scheduleResumeActivity(IBinder token, int procState, boolean isForward, Bundle resumeArgs)
            throws RemoteException;
}
int SCHEDULE_PAUSE_ACTIVITY_TRANSACTION = IBinder.FIRST_CALL_TRANSACTION;
int SCHEDULE_STOP_ACTIVITY_TRANSACTION = IBinder.FIRST_CALL_TRANSACTION+2;
int SCHEDULE_WINDOW_VISIBILITY_TRANSACTION = IBinder.FIRST_CALL_TRANSACTION+3;
int SCHEDULE_RESUME_ACTIVITY_TRANSACTION = IBinder.FIRST_CALL_TRANSACTION+4;
```
并且这里给每个方法编号，来标识每个方法。

##### 2.2.服务端的实现
服务端的实现，继承`Binder`类，实现上面定义的公共接口`IApplicationThread`。然后实现里面的方法。
```
private class ApplicationThread extends ApplicationThreadNative {

    private void updatePendingConfiguration(Configuration config) {
        synchronized (mResourcesManager) {
            if (mPendingConfiguration == null ||
                    mPendingConfiguration.isOtherSeqNewer(config)) {
                mPendingConfiguration = config;
            }
        }
    }

    public final void schedulePauseActivity(IBinder token, boolean finished,
            boolean userLeaving, int configChanges, boolean dontReport) {
        sendMessage(
                finished ? H.PAUSE_ACTIVITY_FINISHING : H.PAUSE_ACTIVITY,
                token,
                (userLeaving ? 1 : 0) | (dontReport ? 2 : 0),
                configChanges);
    }
}
```
这些方法就真正办事情的方法，这里继承`Binder`了，还需要复写另外一个`onTransact`方法，因为都说了是跨进程调用肯定不能直接调用方法的，肯定是客户端和服务端用同样的上面接口定义的标识，然后根据标识调用到对应的方法的。
```
@Override
public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
        throws RemoteException {
    switch (code) {
    case SCHEDULE_PAUSE_ACTIVITY_TRANSACTION:
    {
        data.enforceInterface(IApplicationThread.descriptor);
        IBinder b = data.readStrongBinder();
        boolean finished = data.readInt() != 0;
        boolean userLeaving = data.readInt() != 0;
        int configChanges = data.readInt();
        boolean dontReport = data.readInt() != 0;
        schedulePauseActivity(b, finished, userLeaving, configChanges, dontReport);
        return true;
    }

    case SCHEDULE_STOP_ACTIVITY_TRANSACTION:
    {
        data.enforceInterface(IApplicationThread.descriptor);
        IBinder b = data.readStrongBinder();
        boolean show = data.readInt() != 0;
        int configChanges = data.readInt();
        scheduleStopActivity(b, show, configChanges);
        return true;
    }
}
```

##### 2.3.客户端的实现
客户端的实现，实现上面定义的公共接口`IApplicationThread`。然后实现里面的方法。
```
class ApplicationThreadProxy implements IApplicationThread {
    private final IBinder mRemote;
    
    public ApplicationThreadProxy(IBinder remote) {
        mRemote = remote;
    }
    
    public final IBinder asBinder() {
        return mRemote;
    }
    
    public final void schedulePauseActivity(IBinder token, boolean finished,
            boolean userLeaving, int configChanges, boolean dontReport) throws RemoteException {
        Parcel data = Parcel.obtain();
        data.writeInterfaceToken(IApplicationThread.descriptor);
        data.writeStrongBinder(token);
        data.writeInt(finished ? 1 : 0);
        data.writeInt(userLeaving ? 1 :0);
        data.writeInt(configChanges);
        data.writeInt(dontReport ? 1 : 0);
        mRemote.transact(SCHEDULE_PAUSE_ACTIVITY_TRANSACTION, data, null,
                IBinder.FLAG_ONEWAY);
        data.recycle();
    }
}
```
这里的实现方法只是把*要调用的方法的标识，传递的参数，通过mRemote写入Binder驱动，然后等待远程方法的调用，最后把结果通过Binder驱动写回来。*这里的`mRemote`其实指的是`BinderProxy`这个类，里面有native方法和Binder交互，具体是怎么知道是这个类的，你们还是去看文章吧，一时半会也说不清。

##### 2.4.客户端和服务端的转换
我们可以从这里看出，`ActivityThread.attach`方法，这里呢，我们的App是服务端，AMS是客户端。最终调用的是AMS的代理类`ActivityManagerProxy`。
```
public void attachApplication(IApplicationThread app) throws RemoteException
{
    Parcel data = Parcel.obtain();
    Parcel reply = Parcel.obtain();
    data.writeInterfaceToken(IActivityManager.descriptor);
    data.writeStrongBinder(app.asBinder());
    mRemote.transact(ATTACH_APPLICATION_TRANSACTION, data, reply, 0);
    reply.readException();
    data.recycle();
    reply.recycle();
}
```
对应到服务端`ActivityManagerService`。首先onTransact里面:
```
case ATTACH_APPLICATION_TRANSACTION: {
    data.enforceInterface(IActivityManager.descriptor);
    IApplicationThread app = ApplicationThreadNative.asInterface(
            data.readStrongBinder());
    if (app != null) {
        attachApplication(app);
    }
    reply.writeNoException();
    return true;
}
```
这里`data.readStrongBinder()`得到的是BinderProxy对象，就拿到了`ApplicationThreadProxy`，至于中间的层层转换也是Binder底层的操作。


#### 3.`Activity`可以怎么HOOK?
启动`Activity`，非常的简单，`startActivity`方法即可搞定，但是安卓有一个限制，*必须是在Manifest里面声明*的`Activity`才能被启动。嗯，这个校验过程并不在本地而在`ActivityManagerService`所在的`system_server`进程里面，并不能做什么手脚。
所以现在是衍生出了一些解法，既然要启动的`Activity`必须是在Manifest里面注册，那可以提前注册一些`Activity`以供使用哒。嗯，关于这个也份两种做法。
##### 3.1.代理`Activity`模式
所谓代理`Activity`模式主要特点是这样:
主项目APK注册一个代理Activity（命名为ProxyActivity），ProxyActivity是一个普通的Activity，但只是一个空壳，自身并没有什么业务逻辑。每次打开插件APK里的某一个Activity的时候，都是在主项目里使用标准的方式启动ProxyActivity，再在ProxyActivity的生命周期里同步调用插件中的Activity实例的生命周期方法，从而执行插件APK的业务逻辑。
上面的特点描述出自:[代理Activity模式](http://kaedea.com/2016/06/10/android-dynamical-loading-06-proxy-activity/)
由于现在的插件化`Activity`的方式都是使用的接下来3.2中的第二种，并且代理`Activity`模式也确实不是很方便，所以不是要说的重点。
代理`Activity`模式插件化框架的具体实现就是[dynamic-load-apk](https://github.com/singwhatiwanna/dynamic-load-apk)
关于`Activity`定义了`DLPlugin`接口来表示:[DLPlugin](https://github.com/singwhatiwanna/dynamic-load-apk/blob/master/DynamicLoadApk/lib/src/com/ryg/dynamicload/DLPlugin.java)
把Activity关键的生命周期方法抽象成DLPlugin接口，ProxyActivity通过DLPlugin代理调用插件Activity的生命周期。
![](http://dd089a5b.wiz03.com/share/resources/b6b18d42-e833-4f63-82fe-c6213e78cba1/index_files/5034622.png)
![](http://dd089a5b.wiz03.com/share/resources/b6b18d42-e833-4f63-82fe-c6213e78cba1/index_files/5172087.png)


加载插件的时候，先解析apk文件，然后创建`ClassLoader`，`Resources`，这两个问题也是*特别麻烦*的两个问题，后面会说到，因为一时半会说不清楚。
准备工作代码:[DLPluginManager](https://github.com/singwhatiwanna/dynamic-load-apk/blob/master/DynamicLoadApk/lib/src/com/ryg/dynamicload/internal/DLPluginManager.java)
![](http://dd089a5b.wiz03.com/share/resources/b6b18d42-e833-4f63-82fe-c6213e78cba1/index_files/5599806.png)
启动`Activity`的核心代码也在这个类里面的:
![](http://dd089a5b.wiz03.com/share/resources/b6b18d42-e833-4f63-82fe-c6213e78cba1/index_files/5829112.png)
再看这个:
![](http://dd089a5b.wiz03.com/share/resources/b6b18d42-e833-4f63-82fe-c6213e78cba1/index_files/5875206.png)
哈哈哈，是不是感觉[dynamic-load-apk](https://github.com/singwhatiwanna/dynamic-load-apk)的代码特别简单，轻松看懂，美滋滋，关于代理`Activity`模式的就说到这里，如果想要了解更多去阅读这个项目的源代码吧，说实话代码也特别好看懂，比别的插件化框架好懂太多，因为比较简单。

##### 3.2.动态创建`Activity`模式
其实上面的代码模式的`Activity`是有一定的缺陷的，比如开发要使用that关键字，启动的都是ProxyActivity，LaunchMode的问题等等。所以呢，后面有人继续研究，就出现了现在的动态创建`Activity`模式。
先说一点，动态创建`Activity`的基础:
1.需要对`Activity`的启动过程，Binder机制有一定的认识；
2.基于Hook，动态创建`Activity`模式是通过Hook了系统的部分api实现的，所以需要兼容；
3.需要预注册占坑，前面就分析了，要启动的`Activity`必须已经注册了，所以呢，动态创建也不例外需要先创建好一些`Activity`放在`Manifest`里面使用，可以创建一些不同启动模式，甚至不同进程的以支持多进程；
要Hook掉`Activity`把真正要启动的`Activity`在开始启动的时候替换掉成已经注册的`Activity`，并在`system_server`进程的AMS把事情办完，就是一些`Activity`的管理以及一些校验工作，回到App进程的时候替换回真正的`Activity`，可以想到大致有两种方案。
1.Hook住startActivity方法，在这里替换真正的`Activity`和预注册的`Activity`，当然需要校验一下启动的`Activity`的一些参数，例如LaunchMode以便选取最优的预注册`Activity`，在AMS做完事情回到App进程的时候Hook住`handleLaunchActivity`方法。
Hook startActivity 方法的时候，比较重一点的方式是Hook AMS，比较轻一点的方法可以Hook Instrumentation。

2.直接对ClassLoader动手脚，加载插件类的时候做处理，这个360团队的插件化框架[RePlugin](https://github.com/Qihoo360/RePlugin)就是这样做的，当然这个会更麻烦。

具体操作下一篇说。


#### 4.`ClassLoader`处理?
`ClassLoader`如果不知道嘎哈的，必须先去了解一下咯。
```
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    // ...
    Activity activity = null;
    try {
        java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
        activity = mInstrumentation.newActivity(
                cl, component.getClassName(), r.intent);
        StrictMode.incrementExpectedActivityCount(activity.getClass());
        r.intent.setExtrasClassLoader(cl);
        r.intent.prepareToEnterProcess();
        if (r.state != null) {
            r.state.setClassLoader(cl);
        }
    } catch (Exception e) {
        if (!mInstrumentation.onException(activity, e)) {
            throw new RuntimeException(
                "Unable to instantiate activity " + component
                + ": " + e.toString(), e);
        }
    }
    // ...
    return activity;
}
```
这个问题也可以直接看这篇文章:[插件加载机制](http://weishu.me/2016/04/05/understand-plugin-framework-classloader/)
了解`Activity`的启动流程了，我们知道最后启动`Activity`是由AMS里面调用ATP(ApplicationThreadProxy)，跨进程调用到我们App的AT(ApplicationThread)，然后AT发送消息给Handler H，然后调用`ActivityThread`的`performLaunchActivity`方法，也就是上面我贴出的代码。因为`Activity`也是java对象的嘛，new的时候肯定是需要ClassLoader的，不光是`Activity`，加载插件所有的类都需要ClassLoader的。
这样其实就会遇到一个问题，如果Activity组件存在于独立于宿主程序的文件之中，系统的ClassLoader怎么知道去哪里加载呢？因此，如果不做额外的处理，插件中的Activity对象甚至都没有办法创建出来，谈何启动？
关于插件代码的加载ClassLoader，也有两种方式:
1.自定义ClassLoader加载
自己创建ClassLoader去加载插件，每个插件一个ClassLoader。

2.委托系统ClassLoader加载
可以把我们的插件apk路径放到pathList的对象DexPathList的dexElements字段里面去，然后加载的时候就可以加载到了。
![](http://dd089a5b.wiz03.com/share/resources/b6b18d42-e833-4f63-82fe-c6213e78cba1/index_files/81712625.png)
![](http://dd089a5b.wiz03.com/share/resources/b6b18d42-e833-4f63-82fe-c6213e78cba1/index_files/81764585.png)
上面说的很不具体，详细的可以看我推荐的那篇文章。
第一种方案，每一个插件都有一个自己的ClassLoader，因此类的隔离性非常好，如果不同的插件使用了同一个库的不同版本，就是不同的插件之前可以引用相同库的不同版本，然而这也就意味着，如果采用这种方案的话，插件之间，宿主与插件之间，想使用相同的库，都需要引入，这样会导致插件体积变大的。
他也还有一个好处，如果插件需要升级，直接重新创建一个自定的ClassLoader加载新的插件，然后替换掉原来的版本即可（Java中，不同ClassLoader加载的同一个类被认为是不同的类）。

第二种方案，宿主和插件，插件和插件之间不能存在相同的类。插件升级了之后需要下次启动才能更新。关于这个有看到一个比较好的实现方案，在插件更新了之后也能立即更新的。他是通过替换掉系统的ClassLoader，然后也是每个插件对应一个ClassLoader，可以看看源码[ZeusClassLoader](https://github.com/iReaderAndroid/ZeusPlugin/blob/master/ZeusPlugin/src/main/java/zeus/plugin/ZeusClassLoader.java)。


#### 5.资源处理?
资源的处理，之前有篇文章略微提及了，[Android的资源管理器的创建过程](http://www.jianshu.com/p/db7a9e70cbdc)，这个也确实很麻烦，会再单独写一篇文章来说明。

#### 6.结束语
讲完了？不存在的，因为`Activity`的起点涉及到很多，这里面只是讲了5个问题(第五个问题还没细说，逃)，下篇文章，会参考众多的开源的插件化项目，写一个比较完整的`Activity`的插件化的Demo，写了`Activity`的插件化的Demo之后，对说后面的BroadcastReceiver，Service，ContentProvider也有帮助。
