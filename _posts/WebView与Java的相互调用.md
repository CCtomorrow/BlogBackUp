---
title: 'WebView与Java的相互调用'
date: 2016-04-25 17:20:40
tags: [WebView]
categories: [Android,WebView]
---
本次主要介绍 WebView 和前端的一些交互，前端调用 Java 方法的几种方法，顺便会介绍 Java 调用 JS 的方式。

按如下的顺序依次讲解
1. 前端需要注意的几个地方
2. Java 调用 JS 函数，以及传递参数给 JS 函数
3. JS 调用 Java 代码不需要 Java 函数的返回值的两种方式
4. JS 调用 Java 代码需要获取 Java 函数的返回值的两种方式

##一、前端需要注意的几个地方
###1.a标签的问题
在和客户端交互的过程中，往往都有跳转的，在 Web 开发中，默认的 href 属性通常是 #，然后通过获取标签绑定动作触发事件，这里会出现问题。
```
<a href="#">click here</a>
$(function () {
    $('a').on('click', function () {
        // 调用Android的方法代码
        return true;
    });
});
```
这个时候，会发现，当客户端第一次点击的时候，调用时正常的，当客户端再次点击的时候，就会调用客户端的方法两次，每点击依次，调用客户端的方法次数就会 +1。
解决方案:
```
<a href="javascript:void(0);">click here</a>
```

<!-- more -->

###2.关于传递参数的问题
Android 中一般都只能接受一些 String，int 之类的，不能接收 JS 传递过来的对象，同时 Android 也无法将一个对象传递给 JS 函数，但是 IOS 可以，哈哈。
在开发中，我们制定客户端和前端的交互的时候，一般采用的是 Json，所以这里就需要一个转换了，前端可以通过UA判断当前是 Android 还是 IOS 然后如果是 Android 的话，进行下面的转换。
```
encode: function (data) {
     if ("" == data) {
         return {};
     }
     return JSON.stringify(data).replace(/"/g, '\'');
 },
 decode: function (data) {
     if ("" == data) {
         return "{}";
     }
     return JSON.parse(data.replace(/\'/ig,'\"'));
 }
```
其中 decode 方法是解析 Android 传递过来的 Json 字符串，转换成 JS 里面的 Json 对象。
encode 是传递 Json 参数给 Android 客户端的时候，把 JS 里面的 Json 对象转换成字符串，然后再传递给 Android 客户端。

###3.判断当前的页面是否在自家产品内的浏览器打开的
这个其实很简单啦，客户端和前端商量好指定一个协议，追加一个标识在客户端的内置浏览器的 UA 上面，前端去按照制定的协议解析即可。

###4.判断应用是否安装，如果安装了就打开，不然就跳去下载页面
其实有很多时候有这种需求，有时候不止是打开应用，更需要打开应用里面的某个页面。
为了适用 Android 和 IOS 一般会采用 Scheme 来做，和前端定好 Scheme 的协议，这里前端的判断也是很麻烦的。
协议可以这样定义:
```
前缀://host?content=base64(json)
```
Android 客户端把指定接收这个 Scheme 协议的 Activity 加上这个的             intent-filter:
```
            <intent-filter>
                <action android:name="android.intent.action.VIEW"/>
                <category android:name="android.intent.category.DEFAULT"/>
                <category android:name="android.intent.category.BROWSABLE"/>

                <data android:scheme="前缀"/>
                // host
            </intent-filter>
```
指定 Scheme 和 host 就可以打开对应的 Activity 页面了，然后客户端需要在这个 Activity 的 onCreate 方法和 onNewIntent 里面获取 Intent，然后解析里面的数据，如果按照上面的协议，解析代码应该是这样的。
```
    public void handleIntent() {
        if (mIntent == null) {
            Log.i(Constants.TAG + "WebIntent", "Web Intent is NULL");
            return;
        Uri uri = mIntent.getData();
        if (uri == null) {
            Log.i(Constants.TAG + "WebIntent", "Web Intent URI is NULL");
            return;
        }
        LG.i(Constants.TAG + "WebIntent", "URI String is " + uri.toString());
        String query = uri.getQuery();
        if (TextUtil.isEmpty(query)) {
            LG.i(Constants.TAG + "WebIntent", "Query is NULL ");
            return;
        }
        LG.i(Constants.TAG + "WebIntent", "Query：" + query);
        handleQueryString(query);
    }
```


##二、Java 调用 JS 函数，以及传递参数给 JS 函数
WebView 调用 JS 方法，代码一般是这样的:
```
mWebView.loadUrl("javascript:getShareContent();");
```
WebView 调用 JS 方法并传递参数给 JS，代码一般是这样的:

```
mWebView.loadUrl("javascript:putData('" + mData + "')");
```

##三、JS 调用 Java 代码不需要 Java 函数的返回值
###3.1.对 shouldOverrideUrlLoading 做处理
前端和客户端定好协议，传递协议数据，客户端拦截 shouldOverrideUrlLoading 做处理。

```
<a href="openUserInfo//user/532177" />
```
客户端可以先给 WebView 设置 mWebView.setWebViewClient 然后重写里面的方法。

```
public boolean shouldOverrideUrlLoading(WebView webView, String url) {
    if (url.startsWith("openUserInfo://")) {
        RedirectActivity.redirect(this.activity, url);
        return true;
    }
    if ((url.toLowerCase().startsWith("http://")) || (url.toLowerCase().startsWith("https://"))) {
        return false;
    }
    try {
        this.activity.startActivity(new Intent("android.intent.action.VIEW", Uri.parse(url)));
        return true;
    } catch (ActivityNotFoundException localActivityNotFoundException) {
        localActivityNotFoundException.printStackTrace();
    }
    return true;
}
```
RedirectActivity 可以是一个空的 Activity 专门做跳转的，只需要根据 host 判断当前应该干什么，这里是 openUserInfo 打开用户信息界面。



###3.2.利用 Scheme 做跳转
前面讲解过了使用 Scheme 做跳转的例子。


##四、JS 调用 Java 代码需要获取 Java 函数的返回值
###4.1.使用 addJavascriptInterface 方式
```
webView.addJavascriptInterface(new WebAppJs(), "WebJava");
```
然后实现 WebAppJs 类:

```
    private class WebAppJs implements Serializable {
        private static final long serialVersionUID = 1L;

        /**
         * 在UI线程里运行
         */
        void runOnUiThread(Runnable runnable) {
            mHandler.post(runnable);
        }

        /**
         * 获取用户信息
         *
         * @return
         */
        @JavascriptInterface
        public String getUserInfo() {
            LG.i(Constants.TAG, "------js-java------获取用户信息");
            return "用户信息字符串";
        }
    }
```
这样就可以把字符串信息传递给前端。

###4.2.使用 onJsPrompt 方式
由于在 Android 4.2 以下使用上面的方式会造成注入漏洞，所以才有了这个实现。
利用WebChromeClient的 onJsPrompt 也可以达到目的，当然实现比较麻烦，不过GitHub上面已经有一个比较好的开源的这样的实现。
地址:[https://github.com/pedant/safe-java-js-webview-bridge](https://github.com/pedant/safe-java-js-webview-bridge)
其中他这个实现是这样的，先把要使用的方法在一个类里面写好，然后在网页加载的过程中在 onProgressChanged 里面把写好的类里面的方法生成一些信息，转换成JS代码，注入到当前的网页上面，这样就相当于告诉了网页，我这里已经有了这些方法，你可以自由调用。
这个想要看的可以自行查看，作者有篇博客也有详细的讲解过程。

博客就介绍到这里，上面的四种方法，可酌情采用。