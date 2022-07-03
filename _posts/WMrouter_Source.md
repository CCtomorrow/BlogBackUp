---
title: 'WMrouter源码拆解'
date: 2020-11-28 17:10:20
tags: [组件化]
categories: [Android,组件化]
---

#### 说明
之前分析了Arouter的源码，解析完了发现其实Arouter挺简单的，现在来看WMRouter。WMRouter提供了5个注解，从注解数量上面来看就会比Arouter复杂很多，这5个注解分别是`RouterPage`，`RouterProvider`，`RouterRegex`，`RouterService`，`RouterUri`。下面来一个个的分析这五个不同的注解。其实一个路由框架也基本是围绕一系列的注解展开的。所以从注解下手比较简单，然后了解了一个注解了之后可以触类旁通。


#### 1.RouterUri
WMRouter的源码比较Arouter的复杂很多，这里的注解从最简单使用开始说起。
```java
@RouterUri(path = "/test/schemehost", scheme = "test", host = "test.demo.com")
public static class TestSchemeHostHandler extends EmptyHandler {

}

@RouterUri(path = "/test/interceptor", interceptors = UriParamInterceptor.class)
public static class TestInterceptorHandler extends EmptyHandler {

}

@RouterUri(path = {DemoConstant.JUMP_ACTIVITY_1, DemoConstant.JUMP_ACTIVITY_2})
public class TestBasicActivity extends BaseActivity {

}
```

这个定义就比Arouter复杂很多，路由里面可以配置必须的path，还有一些其他的参数，scheme，host，是否允许外部跳转，要添加的interceptors。WMRouter有个特性，可以配置多个url对应一个目标。

然后来看对应`RouterUri`的注解生成的代码是什么样子的。
```java
public class UriAnnotationInit_72565413b8384a4bebb02d352762d60d implements IUriAnnotationInit {
  public void init(UriAnnotationHandler handler) {
    handler.register("", "", "/show_toast_handler", new ShowToastHandler(), false);
    handler.register("", "", "/nearby_shop_with_location", "com.sankuai.waimai.router.demo.advanced.location.NearbyShopActivity", false, new LocationInterceptor());
    handler.register("", "", "/account_with_login", "com.sankuai.waimai.router.demo.advanced.account.UserAccountActivity", false, new LoginInterceptor());
    handler.register("test", "test.demo.com", "/test/schemehost", new TestSchemeAnnotation.TestSchemeHostHandler(), false);
    handler.register("", "", "/test/interceptors", new TestSchemeAnnotation.TestInterceptorsHandler(), false, new UriParamInterceptor(), new ChainedInterceptor());
    handler.register("", "", "/jump_with_request", "com.sankuai.waimai.router.demo.basic.TestUriRequestActivity", false);
    handler.register("demo_scheme", "demo_host", "/exported", "com.sankuai.waimai.router.demo.basic.ExportedActivity", true);
    handler.register("", "", "/jump_activity_2", "com.sankuai.waimai.router.demo.basic.TestBasicActivity", false);
  }
}
```
跟Arouter的套路是一样的，先定义接口，然后收集。
```
public interface IUriAnnotationInit extends AnnotationInit<UriAnnotationHandler> {
    @Override
    void init(UriAnnotationHandler handler);
}
```
不过这里有点不同的是，这里没有直接使用一个map去put，而是使用一个专门的处理类`UriAnnotationHandler`来处理，当然这个类里面肯定有map去存储这些数据。

这里还有一点需要关注，这里不同的注解，采用了不同的`AnnotationHandler`来处理。比如这里的`RouterUri`注解使用`UriAnnotationHandler`来处理，`RouterPage`注解使用`PageAnnotationHandler`来处理。其实跟Arouter的不同类型的数据采用不同类型的map来存放是一样的，比如`Route`注解的采用`Map<String, RouteMeta> atlas`来存储，`Interceptor`注解的采用`Map<Integer, Class<? extends IInterceptor>> interceptors`来存储。

<!-- more -->

当然这里也有在Arouter源码分析里面提到的问题，所以WMRouter也是采用了单独的管理类，来管理上面的`UriAnnotationInit_72565413b8384a4bebb02d352762d60d`，生成如下类。
```
public class ServiceInit_eb71854fbd69455ef4e0aa026c2e9881 {
  public static void init() {
    ServiceLoader.put(IUriAnnotationInit.class, "com.sankuai.waimai.router.generated.UriAnnotationInit_72565413b8384a4bebb02d352762d60d", com.sankuai.waimai.router.generated.UriAnnotationInit_72565413b8384a4bebb02d352762d60d.class, false);
  }
}
```
这里的实现和Arouter有所不同，里面直接是静态方法，有单独的ServiceLoader来统一管理，ARouter还是使用的接口，直接用map来添加，当然静态方法和接口，在写gradle插件初始化调用都会比较简单。

说明:
##### RouterUri生成类对应的包名
固定包名`com.sankuai.waimai.router.generated`。
管理类的包名`com.sankuai.waimai.router.generated.service`。

##### RouterUri生成类对应的类名
不同的注解生成不同的前缀比如`UriAnnotationInit_`，`PageAnnotationInit_`，`RegexAnnotationInit_`。

管理类前缀`ServiceInit_`，不管什么注解。

##### RouterUri生成类对应的初始化流程
上面我们讲过，RouterUri收集的所有的类，在同一个模块里面会生成一个对应的管理类，来管理，所以我们初始化管理类，然后通过管理类来初始化次模块的路由就行。

现在的问题是我们怎么初始化这个管理类，一般来说使用gradle插件把对应的注册代码写入到某个指定的class文件里面的指定方法即可。

WMRouter采用了一种不同的方案，生成了一个单独的`ServiceLoaderInit`类在`com.sankuai.waimai.router.generated`包名下面，我们反编译看看这个类是什么样子的。
![ServiceLoaderInit](/images/wmrouter_service_loader.png)
我们只需要在合适的时机去初始化这个类就可以加载到所以的路由管理类，service等等。

WMRoter采用了懒加载的方式去加载这个类，可以在应用启动后开启新线程池后台加载也可以在使用的时候自动加载。

主动加载的方法：Router类里面
```
/**
  * 此初始化方法的调用不是必须的。
  * 使用时会按需初始化；但也可以提前调用并初始化，使用时会等待初始化完成。
  * 本方法线程安全。
  */
public static void lazyInit() {
    //加载管理类
    ServiceLoader.lazyInit();
    //加载路由等分组细分类，需要在上面一个方法执行完成
    getRootHandler().lazyInit();
}
```


#### 2.`RouterRegex`和`RouterPage`
关于`RouterRegex`和`RouterPage`，这两个注解其实收集和初始化的方式和`RouterUri`一样，这里就没什么好说的，这里说一下这几个注解的作用。

先说一下，这些注解能添加到Handler类上面的原因是，不管是Activity还是其他，最后都是调用特定的Handler来处理的，你直接给个Handler，那么更方便，直接调用对应的`handle`方法即可。其实这里如果直接注解到Handler上面相当于自己实现跳转，这样和CC框架有点差不多了。

`RouterUri`注解一个类用于跳转，这个类可以是Activity以及UriHandler的子类，跳转的时候由`UriAnnotationHandler`处理。判断类型，如果是UriHandler，则直接调用handle的处理类，如果是字符串，则表示是Activity class的名称，如果是Activity，表示一个Activity类，根据不同的情况生成不同的处理规则。
处理规则在`UriTargetTools`里面：
```java
private static UriHandler toHandler(Object target) {
    if (target instanceof UriHandler) {
        return (UriHandler) target;
    } else if (target instanceof String) {
        return new ActivityClassNameHandler((String) target);
    } else if (target instanceof Class && isValidActivityClass((Class) target)) {
        //noinspection unchecked
        return new ActivityHandler((Class<? extends Activity>) target);
    } else {
        return null;
    }
}
```
这里需要说明的是`RouterUri`可以自定义scheme以及host，你可以直接指定https+hosts直接从网页跳过来。或者如果export是true的话，可以直接从别的app打开此页面。所以WMRouter对他的定义是外部跳转，当然App内部跳转也行。

`RouterPage`注解一个类用于跳转，这个类可以是Activity以及UriHandler的子类，跳转的时候由`PageAnnotationHandler`处理。处理过程和上面一样，不一样的是这个处理且只处理所有格式为 wm_router://page/* 的URI，根据path匹配。所以WMRouter对他的定义是App内部跳转。

`RouterRegex`注解一个类用于跳转，这个类可以是Activity以及UriHandler的子类，跳转的时候由`RegexAnnotationHandler`处理。处理过程和上面一样，不一样的是这个在使用url发起请求之后，会根据正则匹配，如果匹配合适会直接跳转到这个注解所在的目标页面。
```java
public class RegexWrapperHandler extends WrapperHandler {

    private final Pattern mPattern;
    private final int mPriority;

    public RegexWrapperHandler(Pattern pattern, int priority, UriHandler delegate) {
        super(delegate);
        ...
    }

    @Override
    protected boolean shouldHandle(@NonNull UriRequest request) {
        return mPattern.matcher(request.getUri().toString()).matches();
    }
}
```

上面三个注解处理的顺序先`RouterPage`处理，如果能处理会到`RouterPage`注解注解的对应的页面，然后是`RouterRegex`，最后是`RouterUri`。


#### 3.`RouterService`
声明一个Service，通过interface和key加载实现类。此注解可以用在任意静态类上。

被`RouterService`注解的类，会直接生成在`com.sankuai.waimai.router.generated.service`包下面，生成对应的`ServiceInit_`类，
```java
package com.sankuai.waimai.router.generated.service;

public class ServiceInit_b57118238b4f9112ddd862e55789c834 {
  public static void init() {
    ServiceLoader.put(Context.class, "/application", DemoApplication.class, true);
    ServiceLoader.put(ILocationService.class, "/singleton", FakeLocationService.class, true);
    ServiceLoader.put(Func0.class, "/method/get_version_code", GetVersionCodeMethod.class, true);
    ServiceLoader.put(IAccountService.class, "/singleton", FakeAccountService.class, true);
    ServiceLoader.put(Fragment.class, "/fragment/test", TestFragment.class, false);
  }
}
```
管理也是在`ServiceLoaderInit`里面。没什么多说的。


#### 4.路由初始化流程图
![WMRouter init](/images/wmrouter_init.png)

#### 5.路由的使用
![WMRouter start](/images/wmrouter_start.png)
这里先分析路由的使用，再说说服务的调用。
```java
Router.startUri(this, uri);
```
路由的使用这样一行代码就可以搞定。
```java
public static void startUri(Context context, String uri) {
    getRootHandler().startUri(new UriRequest(context, uri));
}
```
后面也是调用`RootUriHandler`来处理，`RootUriHandler`里面会添加几个不同的Uri处理器，`PageAnnotationHandler`，`UriAnnotationHandler`，`RegexAnnotationHandler`来分别处理对应的三个注解`RouterPage`，`RouterUri`，`RouterRegex`注解的对象。是一个责任链的设计模式。

处理顺序就是上面的顺序，当然处理过程中如果`ServiceLoaderInit`还未调用，会先调用init。这个过程在`xxxAnnotationHandler`的`handle`过程中，举例先根据顺序`PageAnnotationHandler`来处理。

我们会先调用所有在`com.sankuai.waimai.router.generated`包下面的`PageAnnotationInit_xxx`的init方法加载所有的被`RouterPage`注解的类。

这个时候是先需要去初始化`com.sankuai.waimai.router.generated.service`包下面的`ServiceInit_xxx`的init方法加载所有的管理类找到如下的特征的类。
```java
public class ServiceInit_xxx {
  public static void init() {
    ServiceLoader.put(IPageAnnotationInit.class, "com.sankuai.waimai.router.generated.PageAnnotationInit_xxx", com.sankuai.waimai.router.generated.PageAnnotationInit_xxx.class, false);
  }
}
```

WMRouter实际调用如下：这里只说PageAnnotationHandler的过程。
`RootUriHandler`分发到`PageAnnotationHandler`调用它的`handle`方法。
```java
@Override
public void handle(@NonNull UriRequest request, @NonNull UriCallback callback) {
    mInitHelper.ensureInit();
    super.handle(request, callback);
}
```
这里的`mInitHelper.ensureInit();`很重要，`super.handle(request, callback);`就是个处理的先不看，这个就是上面说的`ServiceLoaderInit`的初始化和`PageAnnotationInit_xxx`的初始化过程，如果他们都没出初始化的话。
```java
//PageAnnotationHandler class
protected void initAnnotationConfig() {
    //指定我们要查找IPageAnnotationInit的对应的所有的PageAnnotationHandler处理的所有的被注解的类
    RouterComponents.loadAnnotation(this, IPageAnnotationInit.class);
}

//DefaultAnnotationLoader class
@Override
public <T extends UriHandler> void load(T handler,
        Class<? extends AnnotationInit<T>> initClass) {
    //找到所有PageAnnotationInit_xxx implements IPageAnnotationInit
    List<? extends AnnotationInit<T>> services = Router.getAllServices(initClass);
    for (AnnotationInit<T> service : services) {
        //调用init
        service.init(handler);
    }
}

//ServiceLoader class
public static <T> ServiceLoader<T> load(Class<T> interfaceClass) {
    //这个会调用ServiceLoaderInit的init
    sInitHelper.ensureInit();
    //找到IPageAnnotationInit对应的ServiceLoader
    ServiceLoader service = SERVICES.get(interfaceClass);
    if (service == null) {
        synchronized (SERVICES) {
            service = SERVICES.get(interfaceClass);
            if (service == null) {
                service = new ServiceLoader(interfaceClass);
                SERVICES.put(interfaceClass, service);
            }
        }
    }
    //这个service的map里面存放的就是<IPageAnnotationInit PageAnnotationInit_xxx>
    return service;
}

//ServiceLoader class
public <T extends I> List<T> getAll(IFactory factory) {
    Collection<ServiceImpl> services = mMap.values();
    if (services.isEmpty()) {
        return Collections.emptyList();
    }
    List<T> list = new ArrayList<>(services.size());
    for (ServiceImpl impl : services) {
        T instance = createInstance(impl, factory);
        if (instance != null) {
            list.add(instance);
        }
    }
    //得到所有的PageAnnotationInit_xxx并且创建实例
    return list;
}
```
至此，整个初始化完成。

handle的过程其实上面也说了，这里再说一下，`PageAnnotationInit_xxx`生成的注解类在调用init的时候会调用如下处理：
```java
public void register(String path, Object target, boolean exported,
        UriInterceptor... interceptors) {
    if (!TextUtils.isEmpty(path)) {
        //path
        path = RouterUtils.appendSlash(path);
        //这里会生成对应的Activity或者对应的Handler处理
        UriHandler parse = UriTargetTools.parse(target, exported, interceptors);
        //这里直接添加进去了，处理的时候直接拿出来处理即可
        UriHandler prev = mMap.put(path, parse);
        if (prev != null) {
            Debugger.fatal("[%s] 重复注册path='%s'的UriHandler: %s, %s", this, path, prev, parse);
        }
    }
}
```
具体的处理类在放进map的时候就生成好了，处理某个path的时候直接调用即可。

具体的Service也是一样，不过被`RouterService`注解的类里面会再有一个被`RouterProvider`注解的方法，用来标注生成这个Service的方式，例如下面：
```
@RouterService(interfaces = IAccountService.class, key = DemoConstant.SINGLETON, singleton = true)
public class FakeAccountService implements IAccountService {

    @RouterProvider
    public static FakeAccountService getInstance() {
        return new FakeAccountService(Router.getService(Context.class, "/application"));
    }
}
```

#### 5.WMRouter的优缺点
优点:
1.封装的很好，整体代码的质量比Arouter好非常多，包括各种设计模式的使用都有很好的学习的意义。
2.url可以提供多个url对应一个页面

缺点:
一个uri的处理过程，目前看来是不太好的，WMRouter采用了`PageAnnotationHandler`，`UriAnnotationHandler`，`RegexAnnotationHandler`来一一处理，这里会有几个问题。
1.假如我们每个类型的页面都有100个以上，那第一次启动某个页面的时间会非常慢，因为需要每一个类型的都初始化一遍，当然也仅限第一次。
2.三个注解对应的url什么的，没有区分开，会拖慢整个的处理速度，当然，由于存在Regex的类型，分开也不太行。

解决方案：
解决方案参考Arouter的/xxx/yyy，xxx认为分组的形式，但是这样的话不能利用到Regex了，还有一种方案是WMRouter提供了`Router.lazyInit()`方法，异步初始化。但是这样存在另一个问题，假如Application里面就需要用到Service，那就还是得主线程初始化。这样应用启动速度就可能比较慢了。

所以得尽量避免Application里面使用到Service，然后子线程里面去调用`Router.lazyInit()`。

如果想要在Application里面就使用Service，可以先调用`ServiceLoader.lazyInit();`，这个方法只会加载所有的`ServiceInit_xxx`。然后在异步开线程初始化`getRootHandler().lazyInit();`。

