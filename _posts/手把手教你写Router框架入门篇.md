---
title: '手把手教你写Router框架入门篇'
date: 2017-03-26 23:43:40
tags: [Router]
categories: [Android,Router]
---
### 前言

本文代码在[Github](https://github.com/qingyongai/SimpleRouterDemo)上面，可以自行查看，代码还是挺简单的。

最近项目在组件化，模块内的跳转使用了Router框架，通信暂时还有用的接口，没想到好的方法。关于组件化和Router框架是什么这里不作介绍请自行Google。
Router框架的好处可以解耦，还有在Push的时候可能会比较方便，在后台直接配置Activity的名称硬编码并不好，还有一个是一般现在的客户端，App里面的模块都有Web版的，使用Router框架的话，在跳转的过程中，如果本地模块没有可以去自动做些处理，使用浏览器打开对应的模块的。
这篇文章着重说一下编译时候的注解处理器，其实注解的使用在Router框架中的占比非常小，主要的处理还是在Router库里面。由于我自己并不熟悉编译时的注解的写法，所以开篇的文章先写一下注解处理器的使用。

### 不使用注解处理器的基本Router框架的写法
路由框架，顾名思义就是通过一个路由地址可以跳转到对应的页面去。这里的页面只针对Activity，Fragment其实没必要，个人感觉Fragment最好通过Activity的参数处理。
为了对应起Activity和路由地址，我们需要有一个表来进行记录，这里路由地址最好制定好相关的协议，我是希望一个路由地址大概如下:
```
scheme://module/界面/params
```
路由里面支持参数对于Push或者Web跳转到相应的界面是非常方便的。
一个不使用注解处理器的路由框架，需要自己手动把每个界面和相应的Activity对应起来。这个处理过程可以在Application里面做的。
大致步骤:
1.定义接口，让调用着可以去注册。
2.写RouterManager类，去实现跳转。
首先定义一个接口如下:
```
public interface IRoute {
    void initRouter(Map<String, Class<? extends Activity>> routers);
}
```
使用Map将路由地址和Activity关联起来。

<!-- more -->

然后完成RouterManager大概如下所示。
```
public class RouterManager {

    private static volatile RouterManager sManager;
    private Map<String, Class<? extends Activity>> mTables;
    private String mSchemeprefix;

    private RouterManager() {
        mTables = new HashMap<>();
    }

    public static RouterManager getManager() {
        if (sManager == null) {
            synchronized (RouterManager.class) {
                if (sManager == null) {
                    sManager = new RouterManager();
                }
            }
        }
        return sManager;
    }

    public void init(IRoute route) {
        if (route != null) {
            route.initRouter(mTables);
        }
    }

    public void setSetSchemeprefix(String setSchemeprefix) {
        this.mSchemeprefix = setSchemeprefix;
    }

    public void openResult(Context context, String path) {
        if (!TextUtils.isEmpty(mSchemeprefix)) {
            // router://activity/main
            path = mSchemeprefix + "://" + path;
        }
        try {
            Class aClass = mTables.get(path);
            Intent intent = new Intent(context, aClass);
            if (!(context instanceof Activity)) {
                intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
            }
            context.startActivity(intent);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

}
```
使用者只需要在Application里面调用init方法即可。
```
public class RouterApplication extends Application {

    @Override
    public void onCreate() {
        super.onCreate();
        initRouter();
    }

    private void initRouter() {
        RouterManager manager = RouterManager.getManager();
        manager.setSetSchemeprefix("router");
        manager.init(new IRoute() {
            @Override
            public void initRouter(Map<String, Class<? extends Activity>> routers) {
                routers.put("router://activity/main", MainActivity.class);
                routers.put("router://activity/main2", Main2Activity.class);
                routers.put("router://activity/main3", Main3Activity.class);
            }
        });
    }

}
```
相应的代码在[Github](https://github.com/qingyongai/SimpleRouterDemo/tree/master/normalusage)上面。

### 使用注解处理器自动完成initRouter过程
上面的框架已经基本可以使用了，就是Application里面的注册比较麻烦，Activity比较少还好，多的话就比较麻烦了。下面就说一下使用注解处理器自动完成这个过程。

#### 一些基本概念
首先这里讨论的不是在运行时通过反射去调用的注解(不过在编写API的过程中，看想要调用者怎么使用，可能会用到一些反射的)，而是在编译时，扫描字节码文件，根据相应的注解生成特定的代码。

注解处理器:一个在javac中，用来编译时扫描和处理的注解的工具。你可以为特定的注解(你自己写的注解啦)注册你自己的注解处理器。

##### AbstractProcessor
每一个处理器都是继承于AbstractProcessor。我们可以继承并复写一些里面的方法，这些方法java虚拟机会自动调用的。
```
public class MyProcessor extends AbstractProcessor {
    @Override
    public synchronized void init(ProcessingEnvironment env){ }
    @Override
    public boolean process(Set<? extends TypeElement> annoations, RoundEnvironment env) { }
    @Override
    public Set<String> getSupportedAnnotationTypes() { }
    @Override
    public SourceVersion getSupportedSourceVersion() { }
}
```
`init(ProcessingEnvironment processingEnv)`这个方法会被注解处理工具调用，并传入`ProcessingEnvironment`参数，这个参数携带啦很多有用的信息后面会用到。
`process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv)`处理函数，我们需要在这个函数里面写处理特定注解的代码，并生成相应的java文件，`RoundEnvironment `通过这个参数我们可以拿到带有特定注解的元素(类，字段，方法啦)。
`getSupportedAnnotationTypes`
指定当前的注解处理器处理哪些注解，从返回值可以看到是返回`Set`的，那就是一个注解处理器可以处理多个注解的。
`getSupportedSourceVersion`
指定你使用的java版本，通常这里直接返回`SourceVersion.latestSupported()`，可以看这个方法的具体实现。

在java 7中，可以使用注解来代替最后的两个方法。
```
@SupportedSourceVersion(SourceVersion.RELEASE_7)
@SupportedAnnotationTypes("com.ai.router.anno.Route")
```
不过还是建议使用复写的方式吧。

##### 注册你的处理器
在你提供的jar包中，需要有特定的文件在META-INF/services中，文件名是`javax.annotation.processing.Processor`内容是处理器的路径，多个的就没行一个啦。类似下面。

![注册处理器](http://dd089a5b.wiz03.com/share/resources/ab488f1a-bef5-45db-81bf-4b8f5503f69e/index_files/85465331.png)

##### 处理器的编写
其实编写编译时注解处理器，首先要知道我们要干什么，上面一开始就把我们要干什么事情分析的很清楚了，我们需要写一个类，这个类去实现`IRoute`接口，在`initRouter`方法里面去实现关联Activity和URL。
##### 注意
这个类实现完了，我们怎么调用，这个要想明白先。
- 1.让使用者手动调用，因为这个类在MakeProject的时候是生成了的，并且类名我们都写死了，所以可以让使用者手动调用，这就会比较麻烦一些，我们只生成一个类，还好，如果生成的多了，使用者可能会崩溃。
- 2.在提供对外使用的API中使用反射调用，原因和上面一样，类名是死的，就算不是死的，我们也知道类名的生成规则的。这就是上面提到的可能会用到一些反射的原因。

编写编译时注解的library是有一定套路的，一般编写这种库，会建三个module，一个是只存放注解的库，一个是注解处理库(这个只是处理注解，并不会增加apk的大小啦)，一个是提供对外使用的API库，前面已经说到，其实这种库，注解处理器的作用很小的，只是提供某一个功能，其余的99%的功能都是对外使用的API库做的(就是说可以只有API库，其余的需要编译时注解处理的工作可以手动处理)。这个很重要，要记住。一般项目划分如下。
![项目结构划分](http://dd089a5b.wiz03.com/share/resources/ab488f1a-bef5-45db-81bf-4b8f5503f69e/index_files/85486779.png)

##### 注解库实现
注解库只专注提供注解给API库和注解处理器库使用，本身并不做其他的操作。这里我们只需要一个注解即可，这个注解是使用在Activity上面的，就是类(继承自Activity的类)上面，所以这里就很简单啦。
```
package com.ai.router.anno;

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.CLASS)
public @interface Route {
    String value();
}

```
是不是觉得很简单(当然，这篇文章只是个入门篇，其实就算是只使用一个注解类，应该也不会这么简单的使用一个参数啦，这个其实跟你的路由框架的架构设计有关的，你的路由库准备怎样设计使用的路由，要不要使用scheme，要不要二级path，这个都是需要提前想好整个大体的框架，然后画个图，自己好好研究，写个库哪这么简单😢)。

##### 注解处理库实现
其实代码也特别少，这里先贴出代码，然后慢慢讲解的。
```
package com.ai.router.compiler;

// @AutoService(Processor.class) // 生成META-INF等信息
// @SupportedSourceVersion(SourceVersion.RELEASE_7)
// @SupportedAnnotationTypes("com.ai.router.anno.Route")
public class RouterProcessor extends AbstractProcessor {

    @Override
    public synchronized void init(ProcessingEnvironment processingEnv) {
        super.init(processingEnv);
        UtilManager.getMgr().init(processingEnv);
    }

    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {

        UtilManager.getMgr().getMessager().printMessage(Diagnostic.Kind.NOTE, "process");

        Set<? extends Element> elements = roundEnv.getElementsAnnotatedWith(Route.class);
        List<TargetInfo> targetInfos = new ArrayList<>();
        for (Element element : elements) {
            // 检查类型
            if (!Utils.checkTypeValid(element)) continue;
            TypeElement typeElement = (TypeElement) element;
            Route route = typeElement.getAnnotation(Route.class);
            targetInfos.add(new TargetInfo(typeElement, route.value()));
        }
        if (!targetInfos.isEmpty()) {
            generateCode(targetInfos);
        }
        return false;
    }

    /**
     * 生成对应的java文件
     *
     * @param targetInfos 代表router和activity
     */
    private void generateCode(List<TargetInfo> targetInfos) {
        // Map<String, Class<? extends Activity>> routers

        TypeElement activityType = UtilManager
                .getMgr()
                .getElementUtils()
                .getTypeElement("android.app.Activity");

        ParameterizedTypeName actParam = ParameterizedTypeName.get(ClassName.get(Class.class),
                WildcardTypeName.subtypeOf(ClassName.get(activityType)));

        ParameterizedTypeName parma = ParameterizedTypeName.get(ClassName.get(Map.class),
                ClassName.get(String.class), actParam);

        ParameterSpec parameterSpec = ParameterSpec.builder(parma, "routers").build();

        MethodSpec.Builder methodSpecBuilder = MethodSpec.methodBuilder(Constants.ROUTE_METHOD_NAME)
                .addAnnotation(Override.class)
                .addModifiers(Modifier.PUBLIC)
                .addParameter(parameterSpec);
        for (TargetInfo info : targetInfos) {
            methodSpecBuilder.addStatement("routers.put($S, $T.class)", info.getRoute(), info.getTypeElement());
        }

        TypeElement interfaceType = UtilManager
                .getMgr()
                .getElementUtils()
                .getTypeElement(Constants.ROUTE_INTERFACE_NAME);

        TypeSpec typeSpec = TypeSpec.classBuilder(Constants.ROUTE_CLASS_NAME)
                .addSuperinterface(ClassName.get(interfaceType))
                .addModifiers(Modifier.PUBLIC)
                .addMethod(methodSpecBuilder.build())
                .addJavadoc("Generated by Router. Do not edit it!\n")
                .build();
        try {
            JavaFile.builder(Constants.ROUTE_CLASS_PACKAGE, typeSpec)
                    .build()
                    .writeTo(UtilManager.getMgr().getFiler());
        } catch (Exception e) {
            e.printStackTrace();
        }

    }

    /**
     * 定义你的注解处理器注册到哪些注解上
     */
    @Override
    public Set<String> getSupportedAnnotationTypes() {
        Set<String> annotations = new LinkedHashSet<>();
        annotations.add(Route.class.getCanonicalName());
        return annotations;
    }

    /**
     * java版本
     */
    @Override
    public SourceVersion getSupportedSourceVersion() {
        return SourceVersion.latestSupported();
    }
}
```
##### 详细分析
- 1.init
前面说啦，`ProcessingEnvironment`会携带一些有用的东西，我们后面需要用到，这里就把这些对象取出来，放在一个单独的类里面方便随时调用。
`UtilManager`的实现如下，很简单。
```
public class UtilManager {
    /**
     * 一个用来处理TypeMirror的工具类
     */
    private Types typeUtils;
    /**
     * 一个用来处理Element的工具类
     */
    private Elements elementUtils;
    /**
     * 正如这个名字所示，使用Filer你可以创建文件
     */
    private Filer filer;
    /**
     * 日志相关的辅助类
     */
    private Messager messager;

    private static UtilManager mgr = new UtilManager();

    public void init(ProcessingEnvironment environment) {
        setTypeUtils(environment.getTypeUtils());
        setElementUtils(environment.getElementUtils());
        setFiler(environment.getFiler());
        setMessager(environment.getMessager());
    }

    private UtilManager() {
    }
}
```
上面的注释也很清晰了，具体的怎么用，要到用到的时候才能理解。比如我们可以通过Elements.getTypeElement("android.app.Activity")得到Activity的type element，这个非常有作用，后面就会看到。
还可以通过Messager.printMessage()方法输出一些我们想要的信息。
在注解处理的过程，源码的每个部分都是特定的Element。
如下:
```
package com.example;    // PackageElement
public class Foo {        // TypeElement
    private int a;      // VariableElement
    private Foo other;  // VariableElement
    public Foo () {}    // ExecuteableElement
    public void setA (  // ExecuteableElement
                     int newA   // TypeElement
                     ) {}
}
```

- 2.process
首先，我们要得到被`Route`注解的Activity的Element集合，显然我们规定了Route是作用于类上面的，即上面的`TypeElement `，还有不仅是作用于类，而且是Activity的子类上面。
`Set<? extends Element> elements = roundEnv.getElementsAnnotatedWith(Route.class);`这句代码即是获取所有被`Route`注解的`Element`的。
`if (!Utils.checkTypeValid(element)) continue;`
这个是检测被`Route`修饰的`Element`是不是类(`TypeElement `)，并且是不是`Activity`的字类啦。检测的方法这里就不分析啦，GitHub上面的代码有注释，可以去看看。
```
TypeElement typeElement = (TypeElement) element;
Route route = typeElement.getAnnotation(Route.class);
targetInfos.add(new TargetInfo(typeElement, route.value()));
```
这三句代码，比较重要，我们来分析一下，一个被`Route`注释的类表示一个需要被添加到路由表里面的数据。这里使用List记录起来。`typeElement`就是那个Activity的字类，`route.value()`就是Activity的路由地址。

- 3.generateCode生成java文件
我们其实已经知道我们最终需要的文件是什么样子的啦。
```
package com.ai.router.impl;
public class AppRouter implements IRoute {
  @Override
  public void initRouter(Map<String, Class<? extends Activity>> routers) {
    routers.put("router://activity/main2", Main2Activity.class);
    routers.put("router://activity/main3", Main3Activity.class);
    routers.put("router://activity/main", MainActivity.class);
  }
}
```
然后写相应的代码即可，这里生成java文件，我使用的是[square](https://github.com/square)的开源项目[javapoet](https://github.com/square/javapoet)关于它的用法这里就不说了，这个不重要，可以自行搜索用法。

- 4.Router API库编写
上面一直说要实现的接口，其实定义在这个库里面的。
```
public interface IRoute {
    void initRouter(Map<String, Class<? extends Activity>> routers);
}
```
主要的功能实现类。
```
package com.ai.router;
public class RouterManager {

    private static volatile RouterManager sManager;
    private Map<String, Class<? extends Activity>> mTables;
    private String mSchemeprefix;

    private RouterManager() {
        mTables = new HashMap<>();
    }

    public static RouterManager getManager() {
        if (sManager == null) {
            synchronized (RouterManager.class) {
                if (sManager == null) {
                    sManager = new RouterManager();
                }
            }
        }
        return sManager;
    }

    public void init() {
        try {
            String className = "com.ai.router.impl.AppRouter";
            Class<?> moduleRouteTable = Class.forName(className);
            Constructor constructor = moduleRouteTable.getConstructor();
            IRoute instance = (IRoute) constructor.newInstance();
            instance.initRouter(mTables);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public void setSetSchemeprefix(String setSchemeprefix) {
        this.mSchemeprefix = setSchemeprefix;
    }

    public void openResult(Context context, String path) {
        if (!TextUtils.isEmpty(mSchemeprefix)) {
            // router://activity/main
            path = mSchemeprefix + "://" + path;
        }
        try {
            Class aClass = mTables.get(path);
            Intent intent = new Intent(context, aClass);
            if (!(context instanceof Activity)) {
                intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
            }
            context.startActivity(intent);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```
这里就提供两个最简单的功能
`init`注册路由框架，init使用的是反射，拿到`AppRouter`，它实现了IRoute，调用就很简单了，直接调用initRouter方法即可。当然这个方法需要使用者在Application里面去调用。
`openResult`打开对应的Activity，代码很简单，一看即懂。

- 5.使用
Application里面
```
public class RouterApplication extends Application {
    @Override
    public void onCreate() {
        super.onCreate();
        initRouter();
    }
    private void initRouter() {
        RouterManager manager = RouterManager.getManager();
        manager.setSetSchemeprefix("router");
        manager.init();
    }
}
```
打开对应的页面:
`RouterManager.getManager().openResult(this, "activity/main");`

以上，一个最简单的使用编译时注解的Router框架就完成了，这篇文章着重讲了编译时注解的使用。一个Router框架不会这么简单的啦。

### [本文代码在这里啦。](https://github.com/qingyongai/SimpleRouterDemo)
