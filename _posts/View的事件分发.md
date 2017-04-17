---
title: 'View的事件分发'
date: 2016-07-12 00:17:40
tags: [Android]
categories: [Android,自定义View]
---
### 前言:
最近写这两篇文章的原因是因为在自定义View的时候遇到有关事件分发的问题，然后自己怎么也想不通，最后决定自己看一遍源代码，那其实自己以前也有看过应该说很多的关于事件分发的文章，大概的东西也知道，但是对于有的事情还是模棱两可吧。
先说我遇到的问题吧，自定义ViewGroup里面有ImageView，我是想在ViewGroup里面去移动ImageView，发现当前的ViewGroup只能接收到DOWN事件(当然这个事件必须能接收到)，后续的事件都接收不到，这时候我就真的百思不得姐了，不是子View不消费事件，事件会让ViewGroup处理么，没问题啊，为什么会这样呢？
后来发现当给子View即ImageView设置Click事件之后，神奇的事情发生了，ImageView可以移动了，这两个问题围绕着我，那就想好好的把事件分发搞懂。

首先我们要知道onTouchEvent事件是不需要我们处理的，系统以及帮我们处理了，当然我们可以重写这个方法。
#### 事件分发到View，肯定先走到dispatchTouchEvent中
```
// 代码是Android 6.0的
public boolean dispatchTouchEvent(MotionEvent event) {
    boolean result = false;
    final int actionMasked = event.getActionMasked();
    if (onFilterTouchEventForSecurity(event)) {
    ListenerInfo li = mListenerInfo;
    if (li != null && li.mOnTouchListener != null
    && (mViewFlags & ENABLED_MASK) == ENABLED
    && li.mOnTouchListener.onTouch(this, event)) {
            result = true;
        }
        if (!result && onTouchEvent(event)) {
            result = true;
        }
    }
    return result;
}
```
上面的代码是经过简化的。
最终返回的是result，所以看哪几个地方result的值发生变化就好。

<!-- more -->

**由于代码量很小，可以很轻易的看出端倪，默认result是false的，就是不消费事件，
这里有两种情况result会返回true从而消费事件。**

#### 1.第一个条件
    li != null && li.mOnTouchListener != null
    && (mViewFlags & ENABLED_MASK) == ENABLED
    && li.mOnTouchListener.onTouch(this, event)
    这几个条件都要满足才会返回true。

1.1 li != null
ListenerInfo表示的是当前的View所有的一些监听，OnFocusChangeListener，OnTouchListener等等，一般不会为null。
1.2 mOnTouchListener != null
这个需要调用者自己设置，设置了才不为null。
1.3 (mViewFlags & ENABLED_MASK) == ENABLED
当前的View的状态是Enable的，一般情况下是的，调用者可调用setEnabled来设置。
1.4 li.mOnTouchListener.onTouch(this, event)
OnTouch的返回值是否为true。

第一个条件分析:一般这个条件比较难的达到，主要是1.2和1.4需要调用者来控制。
如果条件达到，会返回true，消费掉事件，从而后面的代码就执行不到了，onTouchEvent就不会执行了，OnClick事件也不会执行了。

#### 2.(!result && onTouchEvent(event))
既然都走到这里了result肯定是false的，!result为true了，然后事件消费与否看onTouchEvent的返回值了，这里说一下，一般onTouchEvent返回值都是为true的，下面会具体的分析，总的来说就是事件分发到View了，一般情况下会消费掉的。


#### onTouchEvent事件分析
```
// 代码是Android 6.0的
public boolean onTouchEvent(MotionEvent event) {
    final int action = event.getAction();
    if ((viewFlags & ENABLED_MASK) == DISABLED) {
    return (((viewFlags & CLICKABLE) == CLICKABLE
    || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
    || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE);
    }
    if (((viewFlags & CLICKABLE) == CLICKABLE
    ||(viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
    ||(viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE) {
    // 省略代码
    return true;
    }
    return false;
}
```
总体只有两个地方有返回，一个一个分析。
#### (viewFlags & ENABLED_MASK) == DISABLED
若一个View是disable的，如果它是CLICKABLE或者LONG_CLICKABLE或CONTEXT_CLICKABLE的就返回true，表示消耗掉了Touch事件。但是并不会执行到Click事件去。

#### ((viewFlags & CLICKABLE) == CLICKABLE||(viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)||(viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE
和第一个条件一模一样的，除了当前View是enable的。

不同的子View的clickable属性不同
比如：Button的clickable默认为true，但是TextView的clickable属性默认为false。
View的longClickable属性默认为false。
当然，我们可以通过代码修改这些默认的属性。
比如：setClickable()和setLongClickListener()可以改变View的CLICKABLE和LONG_CLICKABLE属性。
除此以外，通过设置监听器也可改变某些属性。
比如：setOnClickListener()会将View的CLICKABLE设置为true；setOnLongClickListener()会将View的LONG_CLICKABLE设置为true。

图示:
![View事件分发](http://img.blog.csdn.net/20160718120107410)

#### 最后说一下总结
1. OnTouch事件执行的条件是当前的控件是Enable的，并且设置了OnTouchListener。
2. OnTouch事件优先于OnTouchEvent事件，OnTouch返回true消费掉事件了，OnTouchEvent就不会执行了。
3. Action_up的时候处理点击事件。
4. 怎么让View不消费事件:从上面分析到让View不消费事件就是OnTouchEvent返回fasle就可以了，如ImageView，如果放在一个ViewGroup里面，默认就不消费事件。