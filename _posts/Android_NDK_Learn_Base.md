---
title: 'Android NDK 学习二之基础'
date: 2018-11-25 22:27:40
tags: [ndk]
categories: [Android,ndk]
---
### define 使用

参考:http://blog.jobbole.com/108624/

如#define MAX 10，编译器在处理这个代码之前会对MAX进行处理，替换为10，或许有些人认为这样的定义看起来和const常量很相似，但是他们还是有区别的，#define的定义其实就是简单的文本的替换，并不是作为一个量来使用。

使用宏进行条件编译--->

格式如下：#ifdef … (#else) … #endif

```
#ifdef HELLO
  #define WORLD 1
#else
  #define WORLD 0
#endif
```

### Android.mk

Android.mk 的语法用于将源文件分组为模块。 模块是静态库、共享库或独立可执行文件。 可在每个 Android.mk 文件中定义一个或多个模块，也可在多个模块中使用同一个源文件。 构建系统只会将共享库放入应用软件包。 此外，静态库可生成共享库。无需在 Android.mk 文件中列出标头文件或生成的文件之间的显式依赖关系。 NDK 构建系统会自动计算这些关系。

详细看:https://developer.android.com/ndk/guides/android_mk?hl=zh-cn



### Application.mk

此文件用于描述应用需要的原生模块。 模块可以是静态库、共享库或可执行文件。

详细看:https://developer.android.com/ndk/guides/application_mk?hl=zh-cn

<!-- more -->

### JNI两种注册过程

详细看:http://gityuan.com/2016/05/28/android-jni/

- 调用System.loadLibrary("media_jni")

- nativeLoad
  调用dlopen函数，打开一个so文件并创建一个handle；
  调用dlsym()函数，查看相应so文件的JNI_OnLoad()函数指针，并执行相应函数。


总之，System.loadLibrary()的作用就是调用相应库中的JNI_OnLoad()方法。


JVM 查找 native 方法有两种方式： 

详细看:https://juejin.im/entry/5885b019128fe1006c3f0149

1)、JNI_OnLoad 调用 JNI 提供的 RegisterNatives 函数，将本地函数注册到 JVM 中（动态注册）

2)、如果JNI Lib实现中没有定义JNI_OnLoad，则dvm调用dvm ResolveNativeMethod进行动态解析，按照 JNI 规范的命名规则（静态注册）



#### 0x00静态注册

基本原理:根据函数名来建立java方法和JNI函数间的一一对应关系

静态有两个非常重要的关键字JNIEXPORT和JNICALL，这两个关键字时宏定义，主要用于说明该函数是JNI函数，在虚拟机加载so库时，如果发现函数含有上面两个宏定义时，就会链接到对应java层的native方法。

##### 步骤(参考[Android NDK 学习一之开篇](http://www.qingyongai.com/2018/11/15/Android_NDK_Learn_Start/))

- 编写native方法

- 生成头文件

- 编写源文件

```java
package com.ai.hellosample.tools;
public class Util {
    static {
        System.loadLibrary("Hello-Util");
    }
    public static native String getDeviceId();
    public static native String dynamicGenerateKey(String name);
}
```

生成的头文件如下:

```C++
/* DO NOT EDIT THIS FILE - it is machine generated */
#include <jni.h>
/* Header for class com_ai_hellosample_tools_Util */

#ifndef _Included_com_ai_hellosample_tools_Util
#define _Included_com_ai_hellosample_tools_Util
#ifdef __cplusplus
extern "C" {
#endif
/*
 * Class:     com_ai_hellosample_tools_Util
 * Method:    getDeviceId
 * Signature: ()Ljava/lang/String;
 */
JNIEXPORT jstring JNICALL Java_com_ai_hellosample_tools_Util_getDeviceId(JNIEnv *, jclass);
#ifdef __cplusplus
}
#endif
#endif
```

可以看出JNI调用函数名称是按照一定的规则去生成的，规则如下：

```
java_完整包名_类名_方法名
```



#### 0x01动态注册

基本原理:直接告诉native函数其在JNI中对应函数的指针

动态注册的原理是这样的：JNI 允许我们提供一个函数映射表，注册给 JVM，这样 JVM 就可以用函数映射表来调用相应的函数，而不必通过函数名来查找相关函数(这个查找效率很低，函数名超级长)。



实现过程：

- 利用结构体JNINativeMethod保存Java Native函数和JNI函数的对应关系；

- 在一个JNINativeMethod数组中保存所有native函数和JNI函数的对应关系；

- 在Java中通过System.loadLibrary加载完JNI动态库之后，调用JNI_OnLoad函数，开始动态注册；

- JNI_OnLoad中会调用AndroidRuntime::registerNativeMethods函数进行函数注册；

- AndroidRuntime::registerNativeMethods中最终调用jniRegisterNativeMethods完成注册。



```C++
#include <assert.h>
#include "com_ai_hellosample_tools_Util.h"

jstring Java_com_ai_hellosample_tools_Util_getDeviceId(JNIEnv *env, jclass thiz) {
    return (*env).NewStringUTF("*");
}

jstring native_dynamic_key(JNIEnv *env, jclass thiz) {
    return env->NewStringUTF("->");
}

static JNINativeMethod methods[] = {
        {"dynamicGenerateKey", "(Ljava/lang/String;)Ljava/lang/String;", (void *) native_dynamic_key},
        //这里可以有很多其他映射函数
};

static int registerNativeMethods(JNIEnv *env, const char *className, JNINativeMethod *gMethods,
                                 int numMethods) {
    jclass clazz;
    clazz = env->FindClass(className);
    if (clazz == NULL) {
        return JNI_FALSE;
    }
    if (env->RegisterNatives(clazz, gMethods, numMethods) < 0) {
        return JNI_FALSE;
    }
    return JNI_TRUE;
}

static int registerNatives(JNIEnv *env) {
    const char *className = "com/ai/hellosample/tools/Util"; //指定注册的类
    return registerNativeMethods(env, className, methods, sizeof(methods) / sizeof(methods[0]));
}

jint JNI_OnLoad(JavaVM *vm, void *reserved) {
    //LOGI("-------------JNI_OnLoad into.--------\n");
    JNIEnv *env = NULL;
    jint result = JNI_ERR;
    if (vm->GetEnv((void **) &env, JNI_VERSION_1_6) != JNI_OK) {
        return result;
    }
    assert(env != NULL);
    //动态注册，自定义函数
    if (!registerNatives(env)) {
        return -1;
    }
    return JNI_VERSION_1_6;
}
```

JNINativeMethod函数映射表

JNINativeMethod这其实是一个结构体，在jni.h头文件中定义,通过这个结构体从而使Java与jni建立联系。

```

typedef struct {

    const char* name;//Java中函数的名字

    const char* signature;//符号签名，描述了函数的参数和返回值

    void* fnPtr;//函数指针，指向一个被调用的函数

} JNINativeMethod;

```

符号签名可以看最前面的一篇文章，也可以使用java命令查看**class,是class后缀文件**文件对应的签名。`javap -s class`


RegisterNatives动态注册
RegisterNatives,动态注册就是在这里完成的，函数原型在jni.h中
```
RegisterNatives(jclass clazz, const JNINativeMethod* methods, jint nMethods);
```
该函数有3个参数
clazz:java类名，通过FindClass得到
methods:JNINativeMethod的结构体指针
mMethods:方法个数
关于数组名和指针，可以看:https://blog.csdn.net/lfdfhl/article/details/83118205
这个讲清楚了为什么，传递了数组还需要传递，数组的个数。

#### 0x02优劣

静态注册弊端：

- 后期类名、文件名改动，头文件所有函数将失效，需要手动改，超级麻烦易出错

- 代码编写不方便，由于JNI层函数的名字必须遵循特定的格式，且名字特别长；

- 会导致程序员的工作量很大，因为必须为所有声明了native函数的java类编写JNI头文件；

- 程序运行效率低，因为初次调用native函数时需要根据根据函数名在JNI层中搜索对应的本地函数，然后建立对应关系，这个过程比较耗时。