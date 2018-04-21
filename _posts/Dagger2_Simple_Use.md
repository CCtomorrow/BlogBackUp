---
title: 'Dagger2入门使用'
date: 2017-03-03 23:43:40
tags: [Dagger2]
categories: [Android,other]
---

dagger 1.x 以前只是听说过，没使用过，也不打算再研究啦。
最近在看有关组件化的文章，讲道理，组件化如果约定好了，唯一需要解决的可能就是组件之间的通信了。
按照我的理解，组件化大概是这样:基础库+组件+组件+...+主module，基础库可以按照功能分成style库，util库，自定义控件库，这些库可以发布到内部maven里面，各个组件可以引用其中的某个库，或者引用所有的，或者不引用。
各个组件之间的通信分为两个部分，一个是组件之间的跳转，一个是组件的数据交换，对于组件之间的跳转可以使用Router之类的库，组件之间的数据交换可能就会比较麻烦了。
组件间的数据交换，我的初步想法是通过接口，需要交换的数据写一些接口也做成一个库，其他的组件可以引用这个库，做处理然后返回对应的数据。
有大神提到他们公司的组件之间的通信是使用的一个二次开发的dagger2库，所以这次主要来写篇daager2的入门使用文章。

dagger2库的地址:[https://github.com/google/dagger](https://github.com/google/dagger)

### 依赖配置
```groovy
dependencies {
    // 你的其他依赖库
    compile 'com.google.dagger:dagger:2.9'
    annotationProcessor 'com.google.dagger:dagger-compiler:2.9'
}
```
dagger2采用的是编译时注解时注解，以前可能用的是apt，Gradle从2.2版本开始支持annotationProcessor功能来代替Android-apt。android-apt插件作者近期已经发表声明表示后续不会再继续维护该插件。
关于如何从apt切换到annotationProcessor可以参考下图。
![apt](/images/dagger2_apt.png)
图取自 [拓展篇：注解处理器最佳实践](http://blog.csdn.net/dd864140130/article/details/53957691)

### 概念
好吧，其实看了官方Demo，和很多文章之后，感觉dagger2就是用来生成对象的，当然它用编译时注解做了一些自动化的事情（妈蛋，不要看一些文章说什么依赖注入，听着好像挺厉害的样子，很多时候不是自己不够聪明，而是别人说的不够明白）。以前学习过spring，了解过一些这方面的知识。

daager2的好处呢，一是因为生成对象在一个独立的地方，所以修改的时候方便些，测试会简单些，还有就是如果一个对象需要很多其他对象的引用，使用dagger2生成会方便很多的。

- @Inject
既然dagger2是用来生成对象的，我们肯定需要知道哪些对象是需要生成的啦，这个注解其中一个作用就是用来告诉dagger2，这个对象需要被生成。
还有一个作用是给构造方法添加此注解，标记构造函数，dagger2通过@Inject注解可以在需要这个对象的时候，找到这个构造函数并生成对象，给被@Inject标记了的变量提供对象。

- @Module
@Module用于标注提供对象的类。上面已经提供了@Inject，这里还提供@Module，其实呢，当需要生成的对象是第三方的jar包，或者对象有构造参数的时候，@Inject并不会特别的方便。

- @Provides
用于标注@Module标注的类中，提供对象的方法。

- @Component
用于标注接口，是@Inject和@Module之间的桥梁。被Component标注的接口在编译时会生成该接口的实现类（如果@Component标注的接口为CarComponent，则编译期生成的实现类为DaggerCarComponent）,我们通过调用这个实现类的方法完成对象的赋值。

- @Qualifier
dagger里面在同一个Module里面提供两个相同的对象是不允许的，这样dagger无法分辨，这个时候就需要使用Qualifier去自定义不同的注解限定需要提供的是哪种类型的对象。

- @Scope
用于自定义注解，可以通过@Scope自定义的注解来限定注解作用域，实现局部的单例。

- @Singleton
一个通过Scope定义的注解，一般通过它来实现全局单例。但实际上它并不能提前全局单例，是否能提供全局单例还要取决于对应的Component是否为一个全局对象。

<!-- more -->

### 基本使用
比如，我们使用dagger生成用户User对象。
#### 1.创建User类
```java
public class User {
    private String name;

    public User() {
    }

    public User(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```

#### 2.创建用户生成类，即对应的Module
```java
@Module
public class UserModule {

    public UserModule() {
    }

    @Provides
    User provideUser() {
        return new User();
    }

}
```

#### 3.创建对应的Component连接Module和Inject
这里Inject还没用上，被@Inject标注的对象就表示是需要UserModule提供的。
```java
@Component(modules = UserModule.class)
public interface UserComponent {
    void inject(MainActivity activity);
}
```

#### 4.生成代码，使用
点击As上面的make project按钮，对应的代码就会在项目的对应的module的/build/generated/source/apt里面。
然后在MainActivity里面使用生成的对应UserComponent接口的DaggerUserComponent对象即可，dagger默认生成的Component对象即在接口前面加上Dagger。
```java
UserComponent userComponent = DaggerUserComponent.create();
userComponent.inject(this);
```
上面只是最简单的使用，后面的文章会比较详细的介绍每个关键字的用法，以及结合更多的实例。
