---
title: 'Arouter源码拆解'
date: 2020-11-21 14:10:20
tags: [组件化]
categories: [Android,组件化]
---

#### 说明
Arouter其实只有三个注解，Autowired处理参数的，Interceptor处理拦截器的，Route处理路由和Service的。


#### 1.关于参数的Autowired的注册和初始化
先说最简单的关于参数的。MainActivity的参数，我们希望生成如下类，然后在MainActivity调用`inject`方法即可。实现通用接口是方便为了调用。
```
public class MainActivity$$ARouter$$Autowired implements ISyringe {

    @Override
    public void inject(Object target) {
        //生成的时候判断被注解的类是Activity还是Fragment，然后这里生成的时候可以直接转换
        MainActivity act = (MainActivity) target;
        Intent intent = act.getIntent();
        Bundle bundle = intent.getExtras();
        //这里的key为name也是生成类的时候解析被注解的类得到
        act.name = intent.getStringExtra("name");
        act.age = intent.getIntExtra("age", 0);
    }
}
```
说明:
##### 包名
需要说明的是，在生成这个类的时候，包名是直接以MainActivity所在的包，这也是MainActivity里面的参数不加访问修饰符也可以访问的原因。默认不写可以在同一个包里面访问。
##### 类名
生成的类的类名是固定的，这里举例是`要注入的类名+$$ARouter$$Autowired`，这样调用`inject`才能通过固定类名反射new对象，还是会有反射的，不过反射一次之后会缓存下来。
##### inject方法的调用
根据固定的包名，类名直接反射调用。当然，反射一次之后会有缓存。


#### 2.关于拦截器的Interceptor的注册和初始化
收集拦截器也比较简单，先定义通用接口，然后收集。
```
public interface IInterceptorGroup {
    void loadInto(Map<Integer, Class<? extends IInterceptor>> interceptor);
}
```
然后我们希望当前模块所有的被注解的拦截器都能放入map里面去，生成如下类即可。
```
public class ARouter$$Interceptors$$news implements IInterceptorGroup {
  @Override
  public void loadInto(Map<Integer, Class<? extends IInterceptor>> interceptors) {
    interceptors.put(100, SampleInterceptor.class);
  }
}
```
说明:
##### 包名
固定包名`com.alibaba.android.arouter.routes`。
##### 类名
`ARouter$$Interceptors$$+模块名`，模块名的获取是在build.gradle里面配置的。
```
javaCompileOptions {
    annotationProcessorOptions {
        arguments = [AROUTER_MODULE_NAME: project.getName(), AROUTER_GENERATE_DOC: "enable"]
    }
}
```
##### 初始化
核心就是初始化了，首先要在sdk里面声明一个`interceptorMaps`用来存放项目所有的拦截器，然后根据接口写好调用方法。
```
private static void registerInterceptor(IInterceptorGroup interceptorGroup) {
        markRegisteredByPlugin();
        if (interceptorGroup != null) {
            interceptorGroup.loadInto(interceptorMaps);
        }
    }
```
然后使用gradle plugin注册transform，扫描所有的jar包，找到固定的包`com.alibaba.android.arouter.routes`，以及该包下面所有的继承了指定接口`IInterceptorGroup`的类。

然后使用gradle plugin生成代码调用，生成代码到指定的类指定的方法里面，大概和下面一样。
```
private static void loadRouterMap() {
    registerByPlugin = false;
    //auto generate register code by gradle plugin: arouter-auto-register
    // looks like below:
    // registerInterceptor(new ARouter$$Interceptors$$news());
}
```
Arouter是生成代码到`com.alibaba.android.arouter.core.LogisticsCenter`类，然后方法是上面的方法`loadRouterMap`。
我反编译了LogisticsCenter，如下，他其实是调用了一个通用的register方法，看他注释是牺牲了一点性能来让maindex不至于太大，因为如果直接调用上面的方法`registerInterceptor(new ARouter$$Interceptors$$news);`的话，会关联到太多的类，很多类都会被打入到maindex里面。
![LogisticsCenter](/images/arouter_logistics_center.png)


#### 3.关于路由的Route的注册和初始化
路由方面包含了比较多的信息，ARouter把路由分成了几种种类，Activity，Fragment，甚至是模块间通信的IProvider也是用路由注解。
```
public enum RouteType {
    ACTIVITY(0, "android.app.Activity"),
    SERVICE(1, "android.app.Service"),
    PROVIDER(2, "com.alibaba.android.arouter.facade.template.IProvider"),
    CONTENT_PROVIDER(-1, "android.app.ContentProvider"),
    BOARDCAST(-1, ""),
    METHOD(-1, ""),
    FRAGMENT(-1, "android.app.Fragment"),
    UNKNOWN(-1, "Unknown route type");
}
```
思路也是一样，先定义通用接口。
```
public interface IRouteGroup {
    void loadInto(Map<String, RouteMeta> atlas);
}
```
然后收集，所有被Route注解的Activity，Provider，Fragment都会按照不同分组收集到不同的类里面。类似下面这样:
```
public class ARouter$$Group$$groupa implements IRouteGroup {
  @Override
  public void loadInto(Map<String, RouteMeta> atlas) {
    atlas.put("/groupa/act2", RouteMeta.build(RouteType.ACTIVITY, TestModule2Activity.class, "/groupa/act2", "groupa", new java.util.HashMap<String, Integer>(){{put("nickName", 8); put("age", 3); }}, -1, -2147483648));
    atlas.put("/groupa/bservice", RouteMeta.build(RouteType.PROVIDER, BServiceImpl.class, "/groupa/bservice", "groupa", null, -1, -2147483648));
  }
}

public class ARouter$$Group$$groupb implements IRouteGroup {
  @Override
  public void loadInto(Map<String, RouteMeta> atlas) {
    atlas.put("/groupb/act1", RouteMeta.build(RouteType.ACTIVITY, TestModuleActivity.class, "/groupb/act1", "groupb", null, -1, -2147483648));
    atlas.put("/groupb/aservice", RouteMeta.build(RouteType.PROVIDER, AServiceImpl.class, "/groupb/aservice", "groupb", null, -1, -2147483648));
  }
}
```
这里特别需要注意的是路由是按组分开收集的，也就是说一个模块里面的Route可以定义多个分组，会生成多个路由表，分组名是`/news/login`里面的news，当然换成别的就是别的分组了。这也是为什么不同的模块不能使用同一个分组的原因。

说明:
##### 包名
固定包名`com.alibaba.android.arouter.routes`。
##### 类名
`ARouter$$Group$$+分组名`，分组名称是在Route注解的参数里面定义的。
##### 初始化
初始化对路由来说是一个需要处理的事情，因为一般我们用工具去查看市面的应用，会发现对应的ActivityInfo大概一般都会在250个以上，并且这几百个路由分散在不同的模块里面了，如果同时去加载这几百个路由，再加上还有的拦截器和Provider，那么启动应用的速度可想而知。
所以Arouter才有分组的概念，对同个模块的路由分组，不同模块的当然分组也不同了，每个模块定义一个分组管理类，然后对该模块分组的所有路由类进行收集，初始化的时候只初始化这个分组类。具体要使用到某个路由，先初始化分组类，再加载该分组的所有路由。
初始化步骤和拦截器和Provider一样。先定义接口，然后收集当前模块的所有分组类进行添加。
```
public interface IRouteRoot {
    void loadInto(Map<String, Class<? extends IRouteGroup>> routes);
}
```
然后收集，所有当前模块的对应的`ARouter$$Group$$+分组名`类，以分组名为key。
```
public class ARouter$$Root$$news implements IRouteRoot {
  @Override
  public void loadInto(Map<String, Class<? extends IRouteGroup>> routes) {
    routes.put("groupb", ARouter$$Group$$groupb.class);
    routes.put("groupa", ARouter$$Group$$groupa.class);
  }
}
```
然后是初始化，和拦截器和Provider是一样的。首先要在sdk里面声明一个`routerMaps`用来存放项目所有的IRouteGroup，然后根据接口写好调用方法。
```
private static void registerRouteRoot(IRouteRoot routeRoot) {
    markRegisteredByPlugin();
    if (routeRoot != null) {
        routeRoot.loadInto(routerMaps);
    }
}
```
然后使用gradle plugin注册transform，扫描所有的jar包，找到固定的包`com.alibaba.android.arouter.routes`，以及该包下面所有的继承了指定接口`IRouteRoot`的类。

然后使用gradle plugin生成代码调用，生成代码到指定的类指定的方法里面，大概和下面一样。
```
private static void loadRouterMap() {
    registerByPlugin = false;
    //auto generate register code by gradle plugin: arouter-auto-register
    // looks like below:
    // registerRouteRoot(new ARouter$$Root$$news());
}
```
Arouter是生成代码到`com.alibaba.android.arouter.core.LogisticsCenter`类，然后方法是上面的方法`loadRouterMap`。具体也是和拦截器一样


关于Provider的说明:
Provider也和路由一样，先定义接口然后收集，当前模块所有的Provider会被收集到一个单独的类里面。
public interface IProviderGroup {
    void loadInto(Map<String, RouteMeta> providers);
}
然后收集，所有被Route注解的IProvider的子类都会被收集到。
```
public class ARouter$$Providers$$news implements IProviderGroup {
  @Override
  public void loadInto(Map<String, RouteMeta> providers) {
    providers.put("com.alibaba.android.arouter.demo.module.BService", RouteMeta.build(RouteType.PROVIDER, BServiceImpl.class, "/module/bservice", "groupb", null, -1, -2147483648));
    providers.put("com.alibaba.android.arouter.demo.module.AService", RouteMeta.build(RouteType.PROVIDER, AServiceImpl.class, "/module/aservice", "groupa", null, -1, -2147483648));
  }
}
```
说明:
##### 包名
固定包名`com.alibaba.android.arouter.routes`。
##### 类名
`ARouter$$Providerss$$+模块名`，模块名的获取是在build.gradle里面配置的。
```
javaCompileOptions {
    annotationProcessorOptions {
        arguments = [AROUTER_MODULE_NAME: project.getName(), AROUTER_GENERATE_DOC: "enable"]
    }
}
```
##### 初始化
这个流程和拦截器一样的，首先要在sdk里面声明一个`providerMaps`用来存放项目所有的Provider，然后根据接口写好调用方法。
```
private static void registerProvider(IProviderGroup providerGroup) {
    markRegisteredByPlugin();
    if (providerGroup != null) {
        providerGroup.loadInto(providerMaps);
    }
}
```
然后使用gradle plugin注册transform，扫描所有的jar包，找到固定的包`com.alibaba.android.arouter.routes`，以及该包下面所有的继承了指定接口`IProviderGroup`的类。

然后使用gradle plugin生成代码调用，生成代码到指定的类指定的方法里面，大概和下面一样。
```
private static void loadRouterMap() {
    registerByPlugin = false;
    //auto generate register code by gradle plugin: arouter-auto-register
    // looks like below:
    // registerProvider(new ARouter$$Providers$$news());
}
```
Arouter是生成代码到`com.alibaba.android.arouter.core.LogisticsCenter`类，然后方法是上面的方法`loadRouterMap`。具体也是和拦截器一样
![LogisticsCenter](/images/arouter_logistics_center.png)


#### 3.关于路由的使用
这里要说明的是拦截器sdk会自动使用。所以我们一般查找的对象都是RouteMeta对象，我们会在使用一次之后存储下来。
```
ARouter.getInstance().build("/test/activity2").navigation();
```
首先build会生成Postcard对象，当然这个对象里面只有path即`/test/activity2`和group即`test`。
然后需要为Postcard填充额外信息，比如类型，是Activity还是Fragment，还是IProvider等其他的参数。

根据分组名称在`routerMaps`找到`ARouter$$Group$$groupb`具体的路由类，反射调用`loadInto`把当前分组的所有路由都初始化一遍并且缓存起来。同时`routerMaps`移除这个分组。

然后判断这个类型如果是Provider，反射调用初始化这个Provider，添加到缓存。

最后根据不同的类型返回对象。

还有一直直接通过IProvider对象去构建的，这个会利用到上面生成的`providerMaps`。