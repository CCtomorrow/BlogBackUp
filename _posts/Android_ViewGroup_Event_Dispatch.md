---
title: 'ViewGroup事件分发'
date: 2016-07-14 00:40:40
tags: [Custom View]
categories: [Android,Custom View]
---

这次分析ViewGroup的事件分发了。

这次的博客是看了[http://blog.csdn.net/lfdfhl/article/details/51603088](http://blog.csdn.net/lfdfhl/article/details/51603088)的分析的,这篇文章写的很好，值得一看，其次，对于View的事件分发比ViewGroup简单太多。

事件一般先传到Activity的dispatchTouchEvent。
```java
// Android 6.0 代码
public boolean dispatchTouchEvent(MotionEvent ev) {
    if (ev.getAction() == MotionEvent.ACTION_DOWN) {
        onUserInteraction();
    }
    if (getWindow().superDispatchTouchEvent(ev)) {
        return true;
    }
    return onTouchEvent(ev);
}
```
可以看到调用Window的事件分发，最后会调用到DecorView(FrameLayout)进行分发，如果消费了返回true，表示事件已经被消费，不然会调用到Activity的onTouchEvent方法，这里很简单啦。

<!-- more -->

下面说ViewGroup对事件分发的处理。
```java
// 代码是Android6.0的
if (actionMasked == MotionEvent.ACTION_DOWN) {
    cancelAndClearTouchTargets(ev);
    resetTouchState();
}
```
首先在ACTION_DOWN做一些初始化操作，清除状态，这个不重要。继续看后面。
```java
final boolean intercepted;
// ACTION_DOWN或者mFirstTouchTarget不为null，很明显第一次ACTION_DOWN
// 的时候肯定是为null的
// 从这里也可看出ACTION_DOWN事件是一定可以收到的
if (actionMasked == MotionEvent.ACTION_DOWN
  || mFirstTouchTarget != null) {
    // 禁止拦截事件
    final boolean disallowIntercept = (mGroupFlags &
    FLAG_DISALLOW_INTERCEPT) != 0;
    if (!disallowIntercept) {
        // 拦截的话，还要看当前的onInterceptTouchEvent是否拦截了
        // 如果虽然子View调用了拦截，但是ViewGroup不拦截，最终的结果
        // 还是不拦截
        intercepted = onInterceptTouchEvent(ev);
        ev.setAction(action);
  } else {
        // 禁止拦截
        intercepted = false;
    }
} else {
        intercepted = true;
}
```
检查是否需要ViewGroup拦截事件。
mFirstTouchTarget是TouchTarget类型的，TouchTarget里面封装了被触摸的View以及手指对应的id，该类主要用于多点触控。
在第一次ACTION_DOWN的时候，明显mFirstTouchTarget是为null的。
mFirstTouchTarget的两种情况:
(1) mFirstTouchTarget不为null
表示ViewGroup没有拦截Touch事件并且子View消费了Touch
(2) mFirstTouchTarget为null
表示ViewGroup拦截了Touch事件或者虽然ViewGroup没有拦截Touch事件但是子View也没有消费Touch。总之，此时需要ViewGroup自身处理Touch事件。
有个情况要注意:
如果不是ACTION_DOWN事件并且mFirstTouchTarget为null，那么直接将intercepted设置为true，表示ViewGroup拦截Touch事件。
更加直白地说：如果ACTION_DOWN没有被子View消费(mFirstTouchTarget为null)那么当ACTION_MOVE和ACTION_UP到来时ViewGroup不再去调用onInterceptTouchEvent()判断是否需要拦截而是直接的将intercepted设置为true表示由其自身处理Touch事件

```java
if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
  mLastTouchDownTime = ev.getDownTime();
    if (preorderedList != null) {
  for (int j = 0; j < childrenCount; j++) {
            if (children[childIndex] == mChildren[j]) {
                mLastTouchDownIndex = j;
                break;
            }
        }
    } else {
        mLastTouchDownIndex = childIndex;
    }
    mLastTouchDownX = ev.getX();
    mLastTouchDownY = ev.getY();
    newTouchTarget = addTouchTarget(child, idBitsToAssign);
    alreadyDispatchedToNewTouchTarget = true;
    break;
}
```
如果事件没有被拦截并且也没有取消。分发ACTION_DOWN事件:
首先需要找到当前用户按的是哪个View，如果找到就调用dispatchTransformedTouchEvent去分发ACTION_DOWN事件，这个时候就要注意了
```java
private TouchTarget addTouchTarget(View child, int pointerIdBits) {
    TouchTarget target = TouchTarget.obtain(child, pointerIdBits);
    target.next = mFirstTouchTarget;
    mFirstTouchTarget = target;
    return target;
}
```
如果子View消费了ACTION_DOWN事件就会把当前的View添加到mFirstTouchTarget的链表的表头。
mFirstTouchTarget = target;
并为其设置值，mFirstTouchTarget就不会为null了，这里要记住是子View消费了事件的。
```java
if (mFirstTouchTarget == null) {
  handled = dispatchTransformedTouchEvent(ev, canceled, null,
            TouchTarget.ALL_POINTER_IDS);
} else {
    TouchTarget predecessor = null;
    TouchTarget target = mFirstTouchTarget;
    while (target != null) {
        final TouchTarget next = target.next;
        if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
            handled = true;
        } else {
            final boolean cancelChild = resetCancelNextUpFlag(target.child)
                    || intercepted;
            if (dispatchTransformedTouchEvent(ev, cancelChild,
                    target.child, target.pointerIdBits)) {
                handled = true;
            }
            if (cancelChild) {
                if (predecessor == null) {
                    mFirstTouchTarget = next;
                } else {
                    predecessor.next = next;
                }
                target.recycle();
                target = next;
                continue;
            }
        }
        predecessor = target;
        target = next;
    }
}
```
上个步骤是基本是对mFirstTouchTarget进行赋值操作的，这里就继续事件的分发了。
1.mFirstTouchTarget==null
它表示Touch事件被ViewGroup拦截了根本就没有派发给子view或者虽然派发了但是在第四步中没有找到能够消费Touch事件的子View。
2.mFirstTouchTarget != null，表示找到了能够消费Touch事件的子View。
1.处理ACTION_DOWN
```java
if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
    handled = true;
}
```
如果mFirstTouchTarget！=null则说明在第四步中Touch事件已经被消费，所以不再做其他处理
2.  处理ACTION_MOVE和ACTION_UP
```java
else {
    final boolean cancelChild = resetCancelNextUpFlag(target.child)
            || intercepted;
    if (dispatchTransformedTouchEvent(ev, cancelChild,
            target.child, target.pointerIdBits)) {
        handled = true;
    }
    if (cancelChild) {
        if (predecessor == null) {
            mFirstTouchTarget = next;
        } else {
            predecessor.next = next;
        }
        target.recycle();
        target = next;
        continue;
    }
}
```
调用dispatchTransformedTouchEvent()将事件分发给子View处理

```java
if (canceled|| actionMasked == MotionEvent.ACTION_UP
  || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
        resetTouchState();
    } else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
        final int actionIndex = ev.getActionIndex();
        final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);
        removePointersFromTouchTargets(idBitsToRemove);
    }
}
```
最后清理数据和状态还原，在手指抬起或者取消Touch分发时清除原有的相关数据


还有一点，事件分发给子View的时候调用下面的方法
```java
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,View child, int desiredPointerIdBits) {
    final boolean handled;
    // 代码简化
  if (child == null) {
        handled = super.dispatchTouchEvent(transformedEvent);
    } else {
        handled = child.dispatchTouchEvent(transformedEvent);
    }
    return handled;
}
```
上面多次调用这个方法，如果ViewGroup拦截了Touch事件或者子View不能消耗掉Touch事件，那么ViewGroup会在其自身的onTouch()，onTouchEvent()中处理Touch如果子View消耗了Touch事件父View就不能再处理Touch.

图示:
![ViewGroup事件分发](/images/viewgroup_event_dispatch.png)

问题:
ViewGroup将ACTION_DOWN分发给子View，如果子View没有消费该事件，那么当ACTION_MOVE和ACTION_UP到来的时候系统还会将Touch事件派发给该子View么？
答案:不会，ACTION_DOWN事件不消费的话，mFirstTouchTarget就为null，执行到后面直接执行
    handled = dispatchTransformedTouchEvent(ev, canceled, null,
    TouchTarget.ALL_POINTER_IDS);
那传递的子view是null，则会直接调用当前ViewGroup的事件处理方法。

问题:
ViewGroup将ACTION_DOWN分发给子View，如果子View没有消费该事件(默认就ViewGroup消费事件了)，那么当ACTION_MOVE和ACTION_UP到来的时候系统还会将Touch事件派发给该ViewGroup么？(也是我遇到的问题)
答案:不会，事件会传递到Activity里面去处理，紧接着上面的答案，ACTION_DOWN事件会交给当前ViewGroup去处理，即调用View的dispatchTouchEvent去处理事件，关于View的事件分发，上一篇文章有分析，当前的ViewGroup一般情况下肯定没有设置mOnTouchListener，那最终事件会传递到当前ViewGroup的onTouchEvent，其实呢ViewGroup是没有实现onTouchEvent方法的，最终调用的是View的onTouchEvent方法，那其实要满足
    if (((viewFlags & CLICKABLE) == CLICKABLE ||
        (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) ||
        (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE)
这些条件，才能消费事件，ViewGroup一般不会打上这几个FLAG的，所以呢，后续的ACTION_MOVE和ACTION_UP到来的时候系统不会将事件派发给该ViewGroup。

问题:
ViewGroup将ACTION_DOWN分发给子View，如果子View和ViewGroup没有消费该事件(默认会Activity消费)，那么当ACTION_MOVE和ACTION_UP到来的时候系统还会将Touch事件派发给Activity么？
答案:会。
```
public boolean dispatchTouchEvent(MotionEvent ev) {
    if (ev.getAction() == MotionEvent.ACTION_DOWN) {
        onUserInteraction();
    }
    if (getWindow().superDispatchTouchEvent(ev)) {
        return true;
    }
    return onTouchEvent(ev);
}
```
上面是Activity的dispatchTouchEvent处理，第五行代码肯定是返回false的，因为ViewGroup并没有消费事件，就会调用当前Activity的onTouchEvent方法。
```
public boolean onTouchEvent(MotionEvent event) {
    if (mWindow.shouldCloseOnTouch(this, event)) {
        finish();
        return true;
    }
    return false;
}
```
很明显返回true，消费了ACTION_DOWN事件，所以后续的ACTION_MOVE和ACTION_UP，当前的Activity可以收到。
