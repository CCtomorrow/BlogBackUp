---
title: 'Android NDK 学习一之开篇'
date: 2018-11-15 23:43:40
tags: [ndk]
categories: [Android,ndk]
---

这篇文章是使用ndk build的方式开始写jni，后面再接触cmake。

文章参考:https://juejin.im/post/5a67dcdb518825732c53b338

基础:https://blog.csdn.net/lfdfhl/column/info/20840

结构体:如下几种方式

```C++
struct st {
    char name[32];
    int age;
};
typedef st Person;


typedef struct book {
    int page;
} Book;


typedef struct {
    int day;
} Date;
```

<!-- more -->

结构体:与指针
```C++
void initPerson(Person *p) {
    (*p).age = 18;
    p->age = 23;
}
```

#### 1.使用as的External Tools的功能添加好javah命令
```
Pragram             $JDKPath$/bin/javah
Arguments           -d $ModuleFileDir$/src/main/cpp $FileClass$
Working Directory   $ModuleFileDir$/src/main/java
```

#### 2.编写java里面的jni方法
```java
package com.ai.hellosample.tools;
public class Util {
    public static native String getDeviceId();
}
```
然后使用刚才的javah命令生成好头文件在src/main/cpp里面。

#### 3.实现jni方法
在src/main/cpp里面创建Util.cpp类。
```C++
#include "com_ai_hellosample_tools_Util.h"
jstring Java_com_ai_hellosample_tools_Util_getDeviceId(JNIEnv *env, jclass thiz) {
    jstring js = env->NewStringUTF("nj");
    return (*env).NewStringUTF("id");
}
```

#### 4.创建Android.md文件
```
LOCAL_PATH := $(call my-dir)

include $(CLEAR_VARS)

LOCAL_MODULE    := Hello-Util
LOCAL_SRC_FILES := Util.cpp

include $(BUILD_SHARED_LIBRARY)
```
Application.mk
```
# 原生库名称
APP_MODULES := Hello-Util

# 指定机器指令集
APP_ABI := armeabi armeabi-v7a arm64-v8a
```

#### 5.build.gradle配置
```
apply plugin: 'com.android.application'

android {
    compileSdkVersion 28

    defaultConfig {
        applicationId "com.ai.hellosample"
        minSdkVersion 18
        targetSdkVersion 28
        versionCode 100
        versionName "1.0.0"
    }

    externalNativeBuild {
        ndkBuild {
            path 'src/main/cpp/Android.mk'
        }
    }

}
```