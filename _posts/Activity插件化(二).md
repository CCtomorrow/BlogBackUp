﻿---
title: 'Activity插件化(二)'
date: 2017-09-20 21:35:40
tags: [插件化]
categories: [Android,插件化]
---
上一篇文章里面我们分析了一下Activity插件化并提出了5个问题，然后有的问题给出了解决方案，有的问题没有给出解决方案，不用担心，会有一系列的文章来循序渐进的把Activity插件化过程中遇到的问题慢慢讲清楚。
本文代码:[PluginDemo/activity_plugin](https://github.com/qingyongai/PluginDemo/tree/activity_plugin_01)
### 这篇文章只讲一个问题，Activity插件化占坑实现方式
后续文章将要讲到的，现阶段不讲的:
1. 外部apk的解析
2. ClassLoader问题
3. 资源文件问题

#### 1.最简单Activity完全插件化实现方式
了解Activity的启动过程就应该知道启动Activity调用的是`Instrumentation`的`execStartActivity`方法完成的。等到AMS完成校验，以及在需要的时候创建进程等等一系列的操作之后会回到App进程，最后依旧调用`Instrumentation`的另外一个方法`newActivity`。所以我们Hook掉`ActivityThread`的`Instrumentation`的实例`mInstrumentation`即可。

<!-- more -->

```
public class HookManager {

    private static volatile HookManager sManager;

    private HookManager() {
    }

    public static HookManager getManager() {
        if (sManager == null) {
            synchronized (HookManager.class) {
                if (sManager == null) {
                    sManager = new HookManager();
                }
            }
        }
        return sManager;
    }

    /**
    * {@link android.app.ActivityThread} Class
    */
    private Class<?> mActivityThreadClass = null;
    /**
    * {@link android.app.ActivityThread}对象
    */
    private Object mActivityThread = null;
    /**
    * {@link Instrumentation}对象
    */
    private Instrumentation mInstrumentation = null;
    /**
    * {@link Instrumentation} Field
    */
    private Field mInstrumentationField = null;

    /**
    * 获取ActivityThread Class对象
    *
    * @return
    */
    public Class<?> getActivityThreadClass() {
        if (mActivityThreadClass != null) {
            return mActivityThreadClass;
        }
        try {
            mActivityThreadClass = Class.forName("android.app.ActivityThread");
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
        return mActivityThreadClass;
    }

    /**
    * 获取ActivityThread
    *
    * @return
    */
    public Object getActivityThread() {
        if (mActivityThread != null) {
            return mActivityThread;
        }
        Class<?> activityThreadClass = getActivityThreadClass();
        if (activityThreadClass != null) {
            try {
                // 通过反射调用 ActivityThread 的静态方法, 获取 currentActivityThread
                Method currentActivityThreadMethod = activityThreadClass.getDeclaredMethod("currentActivityThread");
                currentActivityThreadMethod.setAccessible(true);
                mActivityThread = currentActivityThreadMethod.invoke(null);
            } catch (NoSuchMethodException e) {
                e.printStackTrace();
            } catch (InvocationTargetException e) {
                e.printStackTrace();
            } catch (IllegalAccessException e) {
                e.printStackTrace();
            }
        }
        return mActivityThread;
    }

    /**
    * 获取 Instrumentation 的 Field
    *
    * @return
    */
    public Field getInstrumentationField() {
        if (mInstrumentationField != null) {
            return mInstrumentationField;
        }
        Class<?> activityThreadClass = getActivityThreadClass();
        if (activityThreadClass != null) {
            // 拿到原始的 mInstrumentation字段
            if (mInstrumentationField == null) {
                try {
                    mInstrumentationField = activityThreadClass.getDeclaredField("mInstrumentation");
                } catch (NoSuchFieldException e) {
                    e.printStackTrace();
                }
            }
        }
        return mInstrumentationField;
    }

    /**
    * 获取Instrumentation
    *
    * @return
    */
    public Instrumentation getInstrumentation() {
        if (mInstrumentation != null) {
            return mInstrumentation;
        }
        Class<?> activityThreadClass = getActivityThreadClass();
        if (activityThreadClass != null) {
            try {
                if (getInstrumentationField() != null) {
                    // 拿到原始的 mInstrumentation 字段
                    getInstrumentationField().setAccessible(true);
                    if (getActivityThread() != null) {
                        mInstrumentation = (Instrumentation) getInstrumentationField().get(getActivityThread());
                    }
                }
            } catch (IllegalAccessException e) {
                e.printStackTrace();
            }
        }
        return mInstrumentation;
    }

    /**
    * 替换{@link android.app.ActivityThread#mInstrumentation}对象为指定对象
    */
    public void replaceInstrumentation() {
        if (getInstrumentation() != null && !(getInstrumentation() instanceof PluginInstrumentation)) {
            PluginInstrumentation instrumentation = new PluginInstrumentation(getInstrumentation());
            if (getInstrumentationField() != null && getActivityThread() != null) {
                // 拿到原始的 mInstrumentation 字段
                getInstrumentationField().setAccessible(true);
                try {
                    getInstrumentationField().set(getActivityThread(), instrumentation);
                } catch (IllegalAccessException e) {
                    e.printStackTrace();
                } catch (IllegalArgumentException e) {
                    e.printStackTrace();
                }
            }
        }
    }

}

```
一个类就搞定了，这里我们就把整个App的`Instrumentation`对象换成了我们计己的`PluginInstrumentation`啦。
```
public class PluginInstrumentation extends Instrumentation {

    private Instrumentation mOrigin;

    private static final String RESOURCES_PACKAGE_NAME = "com.example.activityplugin";

    public PluginInstrumentation(Instrumentation origin) {
        this.mOrigin = origin;
    }

    public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        // 由于这个方法是隐藏的,因此需要使用反射调用;首先找到这个方法
        try {
            Method execStartActivity = Instrumentation.class.getDeclaredMethod(
                    "execStartActivity", Context.class, IBinder.class, IBinder.class,
                    Activity.class, Intent.class, int.class, Bundle.class);
            execStartActivity.setAccessible(true);
            // TODO: 2017/10/27 这里把intent里面的class替换成Manifest里面注册的
            reWarpIntent(who, intent);
            return (ActivityResult) execStartActivity.invoke(mOrigin,
                    who, contextThread, token, target, intent, requestCode, options);
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        }
        return null;
    }

    /**
    * 对Intent做一些修改
    *
    * @param context
    * @param intent
    */
    private void reWarpIntent(Context context, Intent intent) {
        // TODO: 2017/10/27 判断
        if (!intent.getComponent().getClassName().contains("MainActivity")) {
            intent.setClassName(context.getPackageName(), RESOURCES_PACKAGE_NAME + ".A$1");
        }
    }

    @Override
    public Activity newActivity(ClassLoader cl, String className, Intent intent)
            throws InstantiationException, IllegalAccessException, ClassNotFoundException {
        // TODO: 2017/10/27 替换className为真正需要启动的class
        String targetClass = className;
        if (!className.contains("MainActivity")) {
            targetClass = RESOURCES_PACKAGE_NAME + ".PluginSampleActivity";
        }
        return mOrigin.newActivity(cl, targetClass, intent);
    }

}
```
这里说了是最简单的Demo，只是实现了可以使用预注册的Activity代替真实的Activity的目的。
这里我的场景是这样的，定义了一个普通的`PluginSampleActivity`，然后在主界面启动，然后我们不在`Manifest`里面注册这个，使用预注册好的一个`Activity`，大概如下:
![](http://dd089a5b.wiz03.com/share/resources/5306617d-fe7d-4a39-b387-3b1938b5d7d4/index_files/81755721.png)
当然第一步是Hook，我们需要在一个尽可能早的时机Hook。所以我们选择Application的`attachBaseContext`方法。
![](http://dd089a5b.wiz03.com/share/resources/5306617d-fe7d-4a39-b387-3b1938b5d7d4/index_files/82038718.png)
Manifest是这样的，启动Activity的代码就和平时一样的啦。
![](http://dd089a5b.wiz03.com/share/resources/5306617d-fe7d-4a39-b387-3b1938b5d7d4/index_files/82107540.png)
效果如下:
![](http://dd089a5b.wiz03.com/share/resources/5306617d-fe7d-4a39-b387-3b1938b5d7d4/index_files/3867f4d8-f689-486b-ab49-401ae3c89ffe.gif)
是不是so easy的。

#### 2.Manifest 预留 Activity 占坑说明
先看看现阶段各大插件化框架是怎样占坑滴。
[DroidPlugin](https://github.com/DroidPluginTeam/DroidPlugin/blob/master/project/Libraries/DroidPlugin/AndroidManifest.xml)
[Android-Plugin-Framework](https://github.com/limpoxe/Android-Plugin-Framework/blob/master/FairyPlugin/src/main/AndroidManifest.xml)
[VirtualAPK](https://github.com/didi/VirtualAPK/blob/master/CoreLibrary/src/main/AndroidManifest.xml)
[Small](https://github.com/wequick/Small/blob/master/Android/DevSample/small/src/main/AndroidManifest.xml)
##### 1.1.standard
对于standard模式的Activity可以注册一个即可，因为这个模式的Activity每次启动都会生成新的Activity的实例。所以stub并不需要真实存在，只是占个位置，standard的launchmode只需全透明和非全透明各注册1个即可。如果在实际中遇到特别的需求可以再调整的。
当然这个涉及到进程的问题，进程的问题在我看来是这样的，一般的应用插件化开发涉及不到进程的问题，或者说你的插件全都运行在一个新的进程，如果有需要支持的话，可以后续增加，没有必要一开始就整的那么的全。
```
<activity android:name=".A$1" android:launchMode="standard"/>
<activity android:name=".A$2" android:launchMode="standard"
    android:theme="@android:style/Theme.Translucent" />
```
##### 1.2.singleTop
需要注册的stub数量只需 >= 可能 同时 处于运行状态的 singleTop 模式的Activity的数量，最糟糕的情况是所有的 singleTop 模式的Activity都 同时 处于运行状态，那么这种情况下 需要注册的stub数量即为所有插件所有 singleTop 模式的Activity的总和。一般情况，我们应用设计的话应该不会同时有那么多的 singleTop 类型的Activity同时运行的。这里只预注册四个，如果需要更多可自行判断。
```
<activity android:name=".B$1" android:launchMode="singleTop"/>
<activity android:name=".B$2" android:launchMode="singleTop"/>
<activity android:name=".B$3" android:launchMode="singleTop"/>
<activity android:name=".B$4" android:launchMode="singleTop"/>
```
##### 1.3.singleTask
需要注册的stub数量只需 >= 可能 同时 处于运行状态的 singleTask 模式的Activity的数量，最糟糕的情况是所有的 singleTask 模式的Activity都 同时 处于运行状态，那么这种情况下 需要注册的stub数量即为所有插件所有 singleTask 模式的Activity的总和。一般情况，我们应用设计的话应该不会同时有那么多的 singleTask 类型的Activity同时运行的。这里只预注册四个，如果需要更多可自行判断。
```
<activity android:name=".C$1" android:launchMode="singleTask"/>
<activity android:name=".C$2" android:launchMode="singleTask"/>
<activity android:name=".C$3" android:launchMode="singleTask"/>
<activity android:name=".C$4" android:launchMode="singleTask"/>
```
##### 1.4.singleInstance
需要注册的stub数量只需 >= 可能 同时 处于运行状态的 singleInstance 模式的Activity的数量，最糟糕的情况是所有的 singleInstance 模式的Activity都 同时 处于运行状态，那么这种情况下 需要注册的stub数量即为所有插件所有 singleInstance 模式的Activity的总和。一般情况，我们应用设计的话应该不会同时有那么多的 singleInstance 类型的Activity同时运行的。这里只预注册四个，如果需要更多可自行判断。
```
<activity android:name=".D$1" android:launchMode="singleInstance"/>
<activity android:name=".D$2" android:launchMode="singleInstance"/>
<activity android:name=".D$3" android:launchMode="singleInstance"/>
<activity android:name=".D$4" android:launchMode="singleInstance"/>
```

#### 3.启动 Activity Intent解析
Intent对象里面有一个ComponentName对象，这个用来描述一个组件的信息，可以用来描述四大组件，里面有两个对象。
```
private final String mPackage;
private final String mClass;
```
其中mPackage表示包名，mClass表示要启动的对象的类名，一般我们启动一个Activity，如果不是隐式的启动，那么Intent里面的ComponentName对象都是不为空的，所以针对隐式的启动需要做一些处理，但是这个处理不一定是完全的，因为通过action这种隐式启动，很随意，如果用户自定义了action，那么也很难处理的。
```
public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, Activity target,
        Intent intent, int requestCode, Bundle options) {
    // 隐式Intent的转换
    mHookManager.getComponentResolver().implicitToExplicit(intent);
    // Intent的重新解析
    if (intent.getComponent() != null) {
        mHookManager.getComponentResolver().reMakeIntent(intent);
    }
    // 由于这个方法是隐藏的,因此需要使用反射调用,首先找到这个方法
    try {
        Method execStartActivity = Instrumentation.class.getDeclaredMethod(
                "execStartActivity", Context.class, IBinder.class, IBinder.class,
                Activity.class, Intent.class, int.class, Bundle.class);
        execStartActivity.setAccessible(true);
        return (ActivityResult) execStartActivity.invoke(mOrigin,
                who, contextThread, token, target, intent, requestCode, options);
    } catch (Exception e) {
        e.printStackTrace();
    }
    return null;
}
```
我们要在启动Activity的时候替换找到合适的预注册的Activity用来替换插件中的，当然这里只是模拟，启动的和真实的Activity在同一个apk里面。
然后我们来想想`reMakeIntent`方法的实现，我们需要根据类名，包名(对应的插件包)，启动模式来确定一个插件Activity，这里就不涉及到包名，因为是在同一个里面的。
```
/**
    * Intent重新解析，可能需要加上标记，标记是一个需要替换的Activity
    *
    * @param intent
    */
public void reMakeIntent(Intent intent) {
    if (intent.getComponent() == null) {
        return;
    }
    String targetPackageName = intent.getComponent().getPackageName();
    String targetClassName = intent.getComponent().getClassName();
    // 包名不是宿主的，并且能在插件列表找到对应包名的插件就打上是插件Activity的标记
    // 2017/11/7 插件Activity判断，这里做最简单的判断，如果不是主界面就认为是插件Activity
    if (!targetClassName.contains("MainActivity")) {
        intent.putExtra(Constants.KEY_IS_PLUGIN, true);
        intent.putExtra(Constants.KEY_TARGET_PACKAGE, targetPackageName);
        intent.putExtra(Constants.KEY_TARGET_ACTIVITY, targetClassName);
        dispatchStubActivity(intent);
    }
}

/**
    * 找出最合适的可以替换的Activity
    *
    * @param intent
    */
private void dispatchStubActivity(Intent intent) {
    PackageManager pm = mContext.getPackageManager();
    ComponentName component = intent.getComponent();
    if (component == null) {
        return;
    }
    try {
        String targetClassName = component.getClassName();
        ActivityInfo activityInfo = pm.getActivityInfo(intent.getComponent(), 0);
        int launchMode = activityInfo.launchMode;
        String stubActivity = getStubActivity(targetClassName, launchMode);
        if (!TextUtils.isEmpty(stubActivity)) {
            Log.i(TAG, String.format("dispatchStubActivity,[%s -> %s]", targetClassName, stubActivity));
            intent.setClassName(mContext, stubActivity);
        }
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```
首先我们对是插件的Activity打上标记，表示是插件Activity，这里注释里面写的比较清楚，然后根据启动模式找到对应的Activity。
```
/**
 * 获取对应的stub Activity，这里用了一种很巧妙的处理方式，框架一般认为不存在说超过8个特殊
 * 启动模式的Activity在运行，所以这里使用了%8的方式，8个用完就又从第一个开始
 *
 * @param className  要启动的origin Activity
 * @param launchMode 启动模式
 * @param theme      主题，需要靠这个来判断是不是透明的
 * @return 合适的stub Activity
 */
public String getStubActivity(String className, int launchMode, Theme theme) {
    String stubActivity = mCachedStubActivity.get(className);
    if (stubActivity != null) {
        return stubActivity;
    }
    TypedArray array = theme.obtainStyledAttributes(new int[]{
            android.R.attr.windowIsTranslucent,
            android.R.attr.windowBackground
    });
    boolean windowIsTranslucent = array.getBoolean(0, false);
    array.recycle();
    Log.i("StubActivityInfo", "getStubActivity, is transparent theme : " + windowIsTranslucent);
    stubActivity = format(STUB_ACTIVITY_STANDARD, corePackage, usedStandardStubActivity);
    switch (launchMode) {
        case ActivityInfo.LAUNCH_MULTIPLE: {
            stubActivity = format(STUB_ACTIVITY_STANDARD, corePackage, usedStandardStubActivity);
            if (windowIsTranslucent) {
                stubActivity = format(STUB_ACTIVITY_STANDARD, corePackage, 2);
            }
            break;
        }
        case ActivityInfo.LAUNCH_SINGLE_TOP: {
            usedSingleTopStubActivity = usedSingleTopStubActivity % MAX_COUNT_SINGLETOP + 1;
            stubActivity = format(STUB_ACTIVITY_SINGLETOP, corePackage, usedSingleTopStubActivity);
            break;
        }
        case ActivityInfo.LAUNCH_SINGLE_TASK: {
            usedSingleTaskStubActivity = usedSingleTaskStubActivity % MAX_COUNT_SINGLETASK + 1;
            stubActivity = format(STUB_ACTIVITY_SINGLETASK, corePackage, usedSingleTaskStubActivity);
            break;
        }
        case ActivityInfo.LAUNCH_SINGLE_INSTANCE: {
            usedSingleInstanceStubActivity = usedSingleInstanceStubActivity % MAX_COUNT_SINGLEINSTANCE + 1;
            stubActivity = format(STUB_ACTIVITY_SINGLEINSTANCE, corePackage, usedSingleInstanceStubActivity);
            break;
        }
        default:
            break;
    }
    mCachedStubActivity.put(className, stubActivity);
    return stubActivity;
}
```
这个类其实很简单，就是根据启动模式，来匹配预注册的Activity，但是有一个比较巧妙的地方，注释里面也说了，获取对应的stub Activity，这里用了一种很巧妙的处理方式，框架一般认为不存在说超过8个特殊启动模式的Activity在运行，所以这里使用了%8的方式，8个用完就又从第一个开始。

#### 4.启动真实Activity
启动完成从AMS回到App的时候会调用`newActivity`方法，我们要在这个方法里面去启动真正要启动的Activity。实现很简单了，因为前面已经做了标记了的。
```
public Activity newActivity(ClassLoader cl, String className, Intent intent)
        throws InstantiationException, IllegalAccessException, ClassNotFoundException {
    // 2017/10/27 替换className为真正需要启动的class
    String realClass = intent.getStringExtra(Constants.KEY_TARGET_ACTIVITY);
    if (!TextUtils.isEmpty(realClass)) {
        Log.i(TAG, String.format("newActivity[%s : %s]", className, realClass));
        Activity activity = mOrigin.newActivity(cl, realClass, intent);
        activity.setIntent(intent);
        return activity;
    }
    return mOrigin.newActivity(cl, className, intent);
}
```

#### 5.结束语
以上就是启动Activity的Intent的匹配过程，相对来说比较简单的。再次说明一下上面的一个比较巧妙的地方，我们认为一般App里面不存在同时有超过我们预想个数的特殊启动模式的Activity同时运行，如果你觉得你的App可能的话，可以增加这个数量即可，然后我们在启动特殊的启动模式的Activity的时候，使用这个最大的数当一个轮回，使用完这个数目的Activity之后，又从第一个开始，这样就很巧妙的解决了，我们需要去判断，当前有多少个特殊模式的Activity在运行了。
本文代码:[PluginDemo/activity_plugin](https://github.com/qingyongai/PluginDemo/tree/activity_plugin_01)