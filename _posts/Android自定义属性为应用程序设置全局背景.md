---
title: 'Android自定义属性为应用程序设置全局背景'
date: 2016-04-22 16:00:40
tags: [Android]
categories: [Android,自定义View]
---
关于自定义属性，我们用的比较多的时候就是在自定义view的时候了，其实自定义属性还有一些其余的妙用。

### 这里讲解一个利用自定义的属性为应用程序全局的替换背景的例子。

### 1.Android里面使用自定义属性的实例
可能我们在使用ToolBar的时候见过很多次的这种使用方式了。
```
<android.support.v7.widget.Toolbar
    style="@style/ToolBarStyle"
xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="?attr/actionBarSize"
    android:background="?attr/colorPrimary" />
```
在确定ToolBar的宽度和背景色的时候，并没有在xml文件里面去写死，而是读取的Google自定义的属性值。
我们会好奇，这样怎么就能获取到值呢，这个属性是在哪里有定义的呢，关于怎么拿到值是另外一个问题了，现在来说一下这个值是在哪里定义的，并且在哪里赋的初始值的。
首先找到我们当前项目的Theme，我的当前的项目的Theme的parent是Theme.AppCompat.Light.DarkActionBar，找到它的parent，点进去，这里面就是Android内部的res\values里面的一些主题，属性等等。
在里面搜索actionBarSize和colorPrimary这个属性。
![actionBarSize](http://img.blog.csdn.net/20160422144636416)

![colorPrimary](http://img.blog.csdn.net/20160422144726150)

然后再看这个属性的declare-styleable的名字
![declare-styleable](http://img.blog.csdn.net/20160422144928307)

好吧，其实不需要知道declare-styleable的名字，跟这个没什么关系。

这里我们知道Android的values里面确实有这些属性的定义，并不是凭空想象出来的，然后有了定义我们使用之前肯定要对这些属性赋值啊，指望开发者对其赋值有可能开发者并不知道这个东西的呢，所以呢，values里面肯定有赋值的操作，我们继续找。

从哪找起呢，当然是从我们的Theme的parent找起了，前面说了当前的程序的主题是继承Theme.AppCompat.Light.DarkActionBar的。
我们就去找到对应的赋值操作。最终发现主题是继承Theme.Light，当然中间还继承了很多层，可以自行查看。
![colorPrimary](http://img.blog.csdn.net/20160422145836795)
可以看到这里面是对colorPrimary进行了赋值操作的。
actionBarSize的赋值在Base.V7.Theme.AppCompat.Light里面。
![actionBarSize](http://img.blog.csdn.net/20160422150129296)

所以综上所述的一些知识我们知道，如果我们想在xm里面像android:background="?attr/colorPrimary"这样的使用自定义的一些属性做一些操作的话也非常简单，只需要三步就可以了。

步骤:
### 第一步，定义属性
### 第二步，给自定义的属性赋值
### 第三步，使用自定义的属性值

<!-- more -->

### 2.实例讲解一个关于自定义属性的妙用
现在有一个需求是这样的，当前的程序为了风格统一，所有的界面需要一个统一的背景图或者背景色，并且这个背景图或者背景色是可以自由定制的，就是说可以在style文件里面自由配置。那这个时候自定义属性就非常方便的可以完成这个效果了。
三个步骤：
### 2.1.定义自定义的属性
在values里面新建attrs.xml文件，里面定义自定义属性。
![定义自定义的属性](http://img.blog.csdn.net/20160422151050347)
代码也贴这里:
```
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <declare-styleable name="App_Attrs">
        <!-- 默认背景图 -->
        <attr name="baseBackground" format="color|reference"/>
    </declare-styleable>
</resources>
```

### 2.2.给自定义的属性赋值
需要我们在当前程序的Theme里面对这个属性赋值。
```
<resources>
    <style name="AppTheme" parent="Theme.AppCompat.Light.DarkActionBar">
        <item name="baseBackground">@android:color/holo_green_light</item>
        <item name="colorPrimary">@color/colorPrimary</item>
        <item name="colorPrimaryDark">@color/colorPrimaryDark</item>
        <item name="colorAccent">@color/colorAccent</item>
    </style>
</resources>

```
赋值也完成了。

### 2.3.使用自定义的属性值
由于我们当前的程序的所有的界面都需要使用这样的背景色，那么我们可以定义一个自己的baseActivity，里面对setContentView作一个简单的处理。
#### 2.3.1.创建base_layout的xml布局
base_layout.xml
```
<?xml version="1.0" encoding="utf-8"?>
<FrameLayout
    android:id="@+id/base_layout_container"
xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="?attr/baseBackground">
</FrameLayout>
```
里面引用了自定义的属性值。

#### 2.3.2.baseActivity的代码:
```
public class BaseActivity extends AppCompatActivity {
    FrameLayout mBaseContainerView;
    @Override
    public void setContentView(@LayoutRes int layoutResID) {
setContentView(getLayoutInflater().inflate(layoutResID, null));
    }

    @Override
    public void setContentView(View view) {
        setContentView(view, new ViewGroup.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT,
                ViewGroup.LayoutParams.MATCH_PARENT));
    }

    @Override
    public void setContentView(View view, ViewGroup.LayoutParams params) {
        mBaseContainerView = (FrameLayout) getLayoutInflater().inflate(R.layout.base_layout, null);
        Drawable bg = view.getBackground();
        if (bg != null) {
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN) {
                findViewById(R.id.base_layout_container).setBackground(bg);
                view.setBackground(null);
            } else {
                findViewById(R.id.base_layout_container).setBackgroundDrawable(bg);
                view.setBackgroundDrawable(null);
            }
        }
        FrameLayout.LayoutParams p = new FrameLayout.LayoutParams(
                FrameLayout.LayoutParams.MATCH_PARENT, FrameLayout.LayoutParams.MATCH_PARENT);
        mBaseContainerView.addView(view, p);
        super.setContentView(mBaseContainerView, params);
    }
}
```
代码里面对setContentView方法作了简单的处理。

#### 2.3.3.MainActivity里面直接使用就可以看到背景色已经发生改变
```

public class MainActivity extends BaseActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }
}
```
有图有真相:
![效果](http://img.blog.csdn.net/20160422152034411)

Demo地址:[http://download.csdn.net/detail/qy274770068/9499356](http://download.csdn.net/detail/qy274770068/9499356)
