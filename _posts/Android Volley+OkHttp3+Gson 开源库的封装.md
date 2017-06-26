---
title: 'Android Volley + OkHttp3 + Gson 开源库的封装'
date: 2016-04-23 20:52:40
tags: [Volley,OKHttp3]
categories: [Android,other]
---
博客将按照下面的步骤介绍Volley的重新封装:
1.OkHttp3的关于Volley的HttpStack实现
2.HttpRequest的实现和HttpListener回调监听的封装
3.Volley原始的Request的Wrap
4.各种方式的请求的重新实现
5.统一请求的实现
6.使用

所需依赖:
```
compile 'com.android.volley:volley:1.0.0'
compile 'com.squareup.okio:okio:1.7.0'
compile 'com.squareup.okhttp3:okhttp:3.2.0'
compile 'com.google.code.gson:gson:2.6.2'
```

###一、OkHttp3Stack的关于Volley的实现
这个是应该是比较简单的，关于OkHttp3Stack的实现在github上面有实现，本博客里面的实现在我的[Github](https://github.com/qingyongai/VolleyOkExtension/blob/master/volleyokextensionlib/src/main/java/com/yong/volleyok/okhttp/OkHttp3Stack.java)上面。
由于代码比较长，而且这个也不是这篇博客的重点，大家需要的话可以去我的[Github](https://github.com/qingyongai/VolleyOkExtension/blob/master/volleyokextensionlib/src/main/java/com/yong/volleyok/okhttp/OkHttp3Stack.java)查看。

<!-- more -->

###二、HttpRequest的实现和HttpListener回调监听的封装
###2.1.HttpRequest的实现
通过查看Volley的源代码我们会发现，Volley的Request的构造方法是这样写的:
```
    public Request(int method, String url, Response.ErrorListener listener) {
        mMethod = method;
        mUrl = url;
        mErrorListener = listener;
        setRetryPolicy(new DefaultRetryPolicy());
        mDefaultTrafficStatsTag = findDefaultTrafficStatsTag(url);
    }
```
请求的参数有一些通过构造方法传递，而另外一些参数是通过Request里面的很多的get方法得到的，比如post请求的时候的参数是通过Request类里面的getParams()实现的，而我们需要做的就是如果需要post请求，那么就重写请求类，覆盖里面的getParams方法。
```
    protected Map<String, String> getParams() throws AuthFailureError {
        return null;
    }
```
会发现这样并不利于统一的调度，那其实在构造一个请求的时候，参数是不固定的，而且有的需要，有的不需要，这个时候，我们可以通过Builder模式来构造请求，可以进行如下的封装:
```
package com.yong.volleyok;

/**
 * <b>Project:</b> com.yong.volleyok <br>
 * <b>Create Date:</b> 2016/4/22 <br>
 * <b>Author:</b> qingyong <br>
 * <b>Description:</b> 请求的封装，使用Builder模式进行构建 <br>
 */
public class HttpRequest {

    private Builder mBuilder;

    private HttpRequest(Builder builder) {
        this.mBuilder = builder;
    }

    public Map<String, String> getHeaders() {
        return mBuilder.headMaps;
    }

    public int getMethod() {
        return mBuilder.method;
    }

    public Map<String, String> getParams() {
        return mBuilder.params;
    }

    public Request.Priority getPriority() {
        return mBuilder.priority;
    }

    public String getContentType() {
        return mBuilder.contentType;
    }

    public String getParamsEncodeing() {
        return mBuilder.paramsEncodeing;
    }

    public RetryPolicy getRetryPolicy() {
        return mBuilder.retryPolicy;
    }

    public String getUrl() {
        return mBuilder.url;
    }

    public static final class Builder {

        String paramsEncodeing = "UTF-8";
        String url;
        int method = Request.Method.GET;
        Request.Priority priority = Request.Priority.NORMAL;
        String contentType = "application/x-www-form-urlencoded; charset=utf-8";
        // 请求头
        Map<String, String> headMaps = new HashMap<>();
        // 参数
        Map<String, String> params = new HashMap<>();

        // 超时以及重连次数
        RetryPolicy retryPolicy = new DefaultRetryPolicy(10000, 2, 1.0F);

        public Builder(String url) {
            this.url = url;
        }

        /**
         * 增加 Http 头信息
         *
         * @param key   key
         * @param value value
         * @return
         */
        public Builder addHeader(String key, String value) {
            this.headMaps.put(key, value);
            return this;
        }

        /**
         * 增加 Http 头信息
         *
         * @param headers
         * @return
         */
        public Builder addheader(Map<String, String> headers) {
            for (Map.Entry<String, String> entry : headers.entrySet()) {
                this.headMaps.put(entry.getKey(), entry.getValue());
            }
            return this;
        }

        /**
         * 设置 Http 请求方法
         *
         * @param method {@link Request.Method}
         * @return
         */
        public Builder setMethod(int method) {
            this.method = method;
            return this;
        }

        /**
         * 增加请求参数
         *
         * @param key   key
         * @param value value
         * @return
         */
        public Builder addParam(String key, Object value) {
            this.params.put(key, String.valueOf(value));
            return this;
        }

        /**
         * 增加请求参数
         *
         * @param params map<string, object>
         * @return
         */
        public Builder addParam(Map<String, Object> params) {
            for (Map.Entry<String, Object> entry : params.entrySet()) {
                this.params.put(entry.getKey(), String.valueOf(entry.getValue()));
            }
            return this;
        }

        /**
         * 设置请求优先级
         *
         * @param priority {@link Request.Priority}
         * @return
         */
        public Builder setPriority(Request.Priority priority) {
            this.priority = priority;
            return this;
        }

        /**
         * 设置文本类型
         *
         * @param contentType
         * @return
         */
        public Builder setContentType(String contentType) {
            this.contentType = contentType;
            return this;
        }

        /**
         * 设置超时以及重连次数
         *
         * @param initialTimeoutMs  超时时间
         * @param maxNumRetries     重连次数
         * @param backoffMultiplier
         * @return
         */
        public Builder setRetryPolicy(int initialTimeoutMs, int maxNumRetries, float backoffMultiplier) {
            this.retryPolicy = new DefaultRetryPolicy(initialTimeoutMs, maxNumRetries, backoffMultiplier);
            return this;
        }

        /**
         * 构建 HttpRequest
         *
         * @return
         */
        public HttpRequest build() {
            return new HttpRequest(this);
        }

    }

}

```
我们构造这样的一个请求的类，然后在请求的时候可以通过这个类，去构建请求的时候需要的一些参数。这个类很简单就不用详细的讲解了。具体的怎么使用这个类去构造请求，我们会在Wrap Volley的Request的时候详细的说明。

###2.2.HttpListener的封装
其实就是回调的封装，在Volley里面是使用了两个接口来做的，这里统一成一个接口。
```
package com.yong.volleyok;

/**
 * <b>Project:</b> com.yong.volleyok <br>
 * <b>Create Date:</b> 2016/4/22 <br>
 * <b>Author:</b> qingyong <br>
 * <b>Description:</b> 回调响应 <br>
 */
public interface HttpListener<T> {
    /**
     * 服务器响应成功
     *
     * @param result 响应的理想数据。
     */
    void onSuccess(T result);

    /**
     * 网络交互过程中发生错误
     *
     * @param error {@link VolleyError}
     */
    void onError(VolleyError error);
}

```
将成功和失败的回调封装到一个方法里面了。


###三、Request的Wrap
为了以后能更好的升级的考虑，我们最好是不采用直接改源代码的方式，所以我们只能对原始的请求类Request类进行Wrap，然后我们自定义请求类继承这个Wrap的类。先贴代码，然后讲解。
```
package com.yong.volleyok.request;

/**
 * <b>Project:</b> com.yong.volleyok <br>
 * <b>Create Date:</b> 2016/4/22 <br>
 * <b>Author:</b> qingyong <br>
 * <b>Description:</b> 原始请求的包装 <br>
 */
public abstract class RequestWrapper<T> extends com.android.volley.Request<T> {

    /**
     * 请求
     */
    protected HttpRequest mHttpRequest;

    /**
     * 结果
     */
    protected HttpListener<T> mHttpListener;

    public RequestWrapper(HttpRequest httpRequest, HttpListener<T> listener) {
        // 这里不需要错误的监听，下面已经做了处理
        super(httpRequest.getMethod(), httpRequest.getUrl(), null);
        this.mHttpRequest = httpRequest;
        this.mHttpListener = listener;
    }

    /**
     * 得到url，这里get方法作处理，把参数都拼接上去
     *
     * @return
     */
    @Override
    public String getUrl() {
        // 当get的时候做处理，把参数都连接起来
        try {
            if (getMethod() == Method.GET &&
                    (getParams() != null && getParams().size() != 0)) {
                String encodedParams = getEncodedUrlParams();
                String extra = "";
                if (encodedParams != null && encodedParams.length() > 0) {
                    if (!mHttpRequest.getUrl().endsWith("?")) {
                        extra += "?";
                    }
                    extra += encodedParams;
                }
                return mHttpRequest.getUrl() + extra;
            }
        } catch (AuthFailureError e) {
        }
        return mHttpRequest.getUrl();

    }

    /**
     * 拼接get请求的参数的拼接
     *
     * @return
     * @throws AuthFailureError
     */
    public String getEncodedUrlParams() throws AuthFailureError {
        StringBuilder encodedParams = new StringBuilder();
        String paramsEncoding = getParamsEncoding();
        Map<String, String> params = getParams();
        try {
            for (Map.Entry<String, String> entry : params.entrySet()) {
                if (null == entry.getValue()) {
                    continue;
                }
                encodedParams.append(URLEncoder.encode(entry.getKey(), paramsEncoding));
                encodedParams.append('=');
                encodedParams.append(URLEncoder.encode(entry.getValue(), paramsEncoding));
                encodedParams.append('&');
            }
            return encodedParams.toString();
        } catch (UnsupportedEncodingException uee) {
            throw new RuntimeException("Encoding not supported: " + paramsEncoding, uee);
        }
    }

    /**
     * 得到请求头
     *
     * @return
     * @throws AuthFailureError
     */
    @Override
    public Map<String, String> getHeaders() throws AuthFailureError {
        return mHttpRequest.getHeaders();
    }

    /**
     * 请求参数
     *
     * @return
     * @throws AuthFailureError
     */
    @Override
    protected Map<String, String> getParams() throws AuthFailureError {
        return mHttpRequest.getParams();
    }

    /**
     * 请求的ContentType
     *
     * @return
     */
    @Override
    public String getBodyContentType() {
        return mHttpRequest.getContentType();
    }

    /**
     * 请求的优先级，这里RequestQueue里面会根据这个把请求进行排序
     *
     * @return
     */
    @Override
    public Priority getPriority() {
        return mHttpRequest.getPriority();
    }

    /**
     * 设置请求时长，请求失败之后的次数
     *
     * @return
     */
    @Override
    public RetryPolicy getRetryPolicy() {
        return mHttpRequest.getRetryPolicy();
    }

    /**
     * 请求成功
     *
     * @param response The parsed response returned by
     */
    @Override
    protected void deliverResponse(T response) {
        if (mHttpListener != null) {
            mHttpListener.onSuccess(response);
        }
    }

    /**
     * 请求失败
     *
     * @param error Error details
     */
    @Override
    public void deliverError(VolleyError error) {
        if (mHttpListener != null) {
            mHttpListener.onError(error);
        }
    }

}

```
这里最重要的就是这个类了，下面详细的说一下，封装的过程:
**我们在前面定义了HttpRequest和HttpListener类就是为了在这里使用，在构造方法里面把这两个传递进来，然后请求需要的参数通过HttpRequest的一系列的get方法获取，请求最终的回调通过HttpListener传递出去。**

首先来看HttpListener的最终的两个回调的方法:
```
    @Override
    protected void deliverResponse(T response) {
        if (mHttpListener != null) {
            mHttpListener.onSuccess(response);
        }
    }
    @Override
    public void deliverError(VolleyError error) {
        if (mHttpListener != null) {
            mHttpListener.onError(error);
        }
    }
```
这两个方法一个是请求成功的方法，另外一个是请求失败的方法，我们通过HttpListener把最后的结果抛出去，这里可以统一实现，不需要子请求类再去实现了。

再看其余的一些方法，getUrl方法是请求的url，但是我们在封装请求的时候不管是get还是post都是把参数放置getParams方法里面返回的，get请求是直接使用url拼接参数的，所以需要对这个方法进行重写，这样，才能保证get请求能有参数。

getHeaders是请求头，这里直接使用HttpRequest获取到。

getParams时请求的参数，也是直接通过HttpRequest拿到。

其余的方法都是大同小异，总体来说这个封装也比较简单的。


###四、各种方式的请求的重新实现
上面对Request进行了重新的封装之后，我们只需要继承RequestWrapper即可，并且，需要我们实现的方法也只有一个了，parseNetworkResponse。由于我们队Request进行了封装，所以Volley自己带的几个请求，如JsonRequest，StringRequest等，都需要重写，继承RequestWrapper，但是，经过我们的封装，重写不会很麻烦。
这里只举两个例子，一个ByteRequest:
```
package com.yong.volleyok.request;

/**
 * <b>Project:</b> com.yong.volleyok.request <br>
 * <b>Create Date:</b> 2016/4/23 <br>
 * <b>Author:</b> qingyong <br>
 * <b>Description:</b> Byte Request <br>
 */
public class ByteRequest extends RequestWrapper<byte[]> {

    public ByteRequest(HttpRequest httpRequest, HttpListener<byte[]> listener) {
        super(httpRequest, listener);
    }

    @Override
    protected Response<byte[]> parseNetworkResponse(NetworkResponse response) {
        return Response.success(response.data, HttpHeaderParser.parseCacheHeaders(response));
    }
}

```
可以看到经过我们的封装之后，实现变得非常简单了。

GsonRequest:
```
package com.yong.volleyok.request;

/**
 * <b>Project:</b> com.yong.volleyok.request <br>
 * <b>Create Date:</b> 2016/4/23 <br>
 * <b>Author:</b> qingyong <br>
 * <b>Description:</b> Gson Request <br>
 */
public class GsonRequest<T> extends RequestWrapper<T> {

    private static Gson mGson = new Gson();
    private Class<T> mClass;
    private TypeToken<T> mTypeToken;

    public GsonRequest(Class<T> tClass, HttpRequest httpRequest, HttpListener<T> listener) {
        this(tClass, null, httpRequest, listener);
    }

    public GsonRequest(Class<T> tClass, TypeToken<T> typeToken,
                       HttpRequest httpRequest, HttpListener<T> listener) {
        super(httpRequest, listener);
        mClass = tClass;
        mTypeToken = typeToken;
    }

    @SuppressWarnings("unchecked")
    @Override
    protected Response<T> parseNetworkResponse(NetworkResponse response) {
        try {
            String json = new String(
                    response.data, HttpHeaderParser.parseCharset(response.headers, getParamsEncoding()));
            if (mTypeToken == null) {
                return Response.success(
                        mGson.fromJson(json, mClass), HttpHeaderParser.parseCacheHeaders(response));
            } else {
                return (Response<T>) Response.success(
                        mGson.fromJson(json, mTypeToken.getType()), HttpHeaderParser.parseCacheHeaders(response));
            }
        } catch (UnsupportedEncodingException e) {
            return Response.error(new ParseError(e));
        } catch (JsonSyntaxException e) {
            return Response.error(new ParseError(e));
        }
    }
}

```
这里只贴出这两个方法，还有的方法可以看[github](https://github.com/qingyongai/VolleyOkExtension)。


###五、统一请求的实现
请求都封装好了，现在只有调用了，针对封装的6种请求可以封装一个接口。
```
package com.yong.volleyok;

/**
 * <b>Project:</b> com.yong.volleyok <br>
 * <b>Create Date:</b> 2016/4/23 <br>
 * <b>Author:</b> qingyong <br>
 * <b>Description:</b> 请求 <br>
 */
public interface IHttpClient {

    /**
     * byte请求
     */
    Request byteRequest(HttpRequest httpRequest, HttpListener<byte[]> listener, Object tag);

    /**
     * String请求
     */
    Request stringRequest(HttpRequest httpRequest, HttpListener<String> listener, Object tag);

    /**
     * gzip请求
     */
    Request gZipRequest(HttpRequest httpRequest, HttpListener<String> listener, Object tag);

    /**
     * JsonObject请求
     * @return
     */
    Request jsonObjectRequest(String requestBody, HttpRequest httpRequest, HttpListener<JSONObject> listener, Object tag);

    /**
     * JsonArray请求
     */
    Request jsonArrayRequest(String requestBody, HttpRequest httpRequest, HttpListener<JSONArray> listener, Object tag);

    /**
     * Gson请求，可以映射Model
     */
    <T> Request gsonRequest(Class<T> tClass, TypeToken<T> typeToken, HttpRequest httpRequest, HttpListener<T> listener, Object tag);
}

```
然后再写一个具体的实现类即可。
```
package com.yong.volleyok;

/**
 * <b>Project:</b> com.yong.volleyok.http <br>
 * <b>Create Date:</b> 2016/4/23 <br>
 * <b>Author:</b> qingyong <br>
 * <b>Description:</b> 请求的具体实现类 <br>
 */
public class HttpClient implements IHttpClient {

    private static HttpClient INSTANCE;
    private static final int[] sLock = new int[0];

    private final RequestQueue mRequestQueue;
    private final Context mContext;

    private HttpClient(Context context) {
        mContext = context;
        mRequestQueue = Volley.newRequestQueue(context,
                new OkHttp3Stack(new OkHttpClient()));
    }

    /**
     * 这里使用Application的Context
     *
     * @param context
     * @return
     */
    public static HttpClient getInstance(Context context) {
        if (null == INSTANCE) {
            synchronized (sLock) {
                if (null == INSTANCE) {
                    INSTANCE = new HttpClient(context);
                }
            }
        }
        return INSTANCE;
    }

    /**
     * 添加请求
     *
     * @param request
     */
    public void addRequest(Request request, Object tag) {
        if (tag != null) {
            request.setTag(tag);
        }
        mRequestQueue.add(request);
    }

    /**
     * 取消请求
     *
     * @param tag
     */
    public void cancelRequest(Object tag) {
        mRequestQueue.cancelAll(tag);
    }

    public Request ByteRequest(HttpRequest httpRequest, HttpListener<byte[]> listener) {
        return byteRequest(httpRequest, listener, null);
    }

    @Override
    public Request byteRequest(HttpRequest httpRequest, HttpListener<byte[]> listener, Object tag) {
        ByteRequest request = new ByteRequest(httpRequest, listener);
        addRequest(request, tag);
        return request;
    }

    public Request stringRequest(HttpRequest httpRequest, HttpListener<String> listener) {
        return stringRequest(httpRequest, listener, null);
    }

    @Override
    public Request stringRequest(HttpRequest httpRequest, HttpListener<String> listener, Object tag) {
        StringRequest request = new StringRequest(httpRequest, listener);
        addRequest(request, tag);
        return request;
    }

    public Request gZipRequest(HttpRequest httpRequest, HttpListener<String> listener) {
        return gZipRequest(httpRequest, listener, null);
    }

    @Override
    public Request gZipRequest(HttpRequest httpRequest, HttpListener<String> listener, Object tag) {
        GZipRequest request = new GZipRequest(httpRequest, listener);
        addRequest(request, tag);
        return request;
    }

    public Request jsonObjectRequest(HttpRequest httpRequest, HttpListener<JSONObject> listener) {
        return jsonObjectRequest(null, httpRequest, listener);
    }

    public Request jsonObjectRequest(String requestBody, HttpRequest httpRequest, HttpListener<JSONObject> listener) {
        return jsonObjectRequest(requestBody, httpRequest, listener, null);
    }

    @Override
    public Request jsonObjectRequest(String requestBody, HttpRequest httpRequest, HttpListener<JSONObject> listener, Object tag) {
        JsonObjectRequest request = new JsonObjectRequest(requestBody, httpRequest, listener);
        addRequest(request, tag);
        return request;
    }

    public Request jsonArrayRequest(HttpRequest httpRequest, HttpListener<JSONArray> listener) {
        return jsonArrayRequest(httpRequest, listener, null);
    }

    public Request jsonArrayRequest(HttpRequest httpRequest, HttpListener<JSONArray> listener, Object tag) {
        return jsonArrayRequest(null, httpRequest, listener, tag);
    }

    @Override
    public Request jsonArrayRequest(String requestBody, HttpRequest httpRequest, HttpListener<JSONArray> listener, Object tag) {
        JsonArrayRequest request = new JsonArrayRequest(requestBody, httpRequest, listener);
        addRequest(request, tag);
        return request;
    }

    public <T> Request gsonRequest(Class<T> tClass, HttpRequest httpRequest, HttpListener<T> listener) {
        return gsonRequest(tClass, httpRequest, listener, null);
    }

    public <T> Request gsonRequest(Class<T> tClass, HttpRequest httpRequest, HttpListener<T> listener, Object tag) {
        return gsonRequest(tClass, null, httpRequest, listener, tag);
    }

    @Override
    public <T> Request gsonRequest(Class<T> tClass, TypeToken<T> typeToken,
                                   HttpRequest httpRequest, HttpListener<T> listener, Object tag) {
        GsonRequest<T> request = new GsonRequest<T>(tClass, typeToken, httpRequest, listener);
        addRequest(request, tag);
        return request;
    }
}

```

六、使用
当然使用也非常简单，看HttpClient就知道有哪些方法。
```
        mResult = (TextView) findViewById(R.id.result);
        mHttpClient = HttpUtil.getHttpClient();
        HttpRequest request = new HttpRequest.Builder("http://www.mocky.io/v2/571b3c270f00001a0faddfcc")
                .setMethod(Request.Method.GET)
                .build();
        mHttpClient.stringRequest(request, new HttpListener<String>() {
            @Override
            public void onSuccess(String result) {
                Log.e("TAG", result);
                mResult.setText(result);
            }

            @Override
            public void onError(VolleyError error) {
                mResult.setText(error.getMessage());
            }
        });
```

库和Demo地址:
[https://github.com/qingyongai/VolleyOkExtension](https://github.com/qingyongai/VolleyOkExtension)