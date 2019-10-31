# Okhttp简单使用
## 简介
	支持HTTP/2，允许所有同一个主机地址的请求共享同一个socket连接
	连接池减少请求延时
	透明的GZIP压缩减少响应数据的大小
	缓存响应内容，避免一些完全重复的请求
**新版Okhttp使用了Kotlin重写了，旧版的我都还没看完，新版都出来了；大哥我努力学了，学习赶不上啊。**
## 使用
	implementation("com.squareup.okhttp3:okhttp:4.2.2")

[okhttp GitHub地址](https://github.com/square/okhttp)

### 同步/异步请求
异步get请求

```java
//        异步get请求
        String add = "http://www.baidu.com";
//        第一步、创建client
        OkHttpClient client = new OkHttpClient();
//        还有其他相关设置
//        OkHttpClient client1 = new OkHttpClient.Builder()
//                .readTimeout(10, TimeUnit.SECONDS)   读超时设置
//                .connectTimeout(20,TimeUnit.SECONDS)  连接超时设置
//                ...
//                .build();

//        第二步、构造Request对象
        final Request request = new Request.Builder()
                .url(add)
                .get()//默认get
                .build();
//        第三步、构建Call对象；
        Call call = client.newCall(request);
//        通过Call的enqueue()方法提交异步请求；
        call.enqueue(new Callback() {
            @Override
            public void onFailure(@NotNull Call call, @NotNull IOException e) {
                
            }

            @Override
            public void onResponse(@NotNull Call call, @NotNull Response response) throws IOException {

            }
        });

```

同步get请求

前面几步跟异步的请求的是一样的，就是在最后一步是使用Call execute()的方法。
但是这个方法会堵塞，因此不能放在Android 主线程，不然会出现ANR；

```java
new Thread(new Runnable() {
    @Override
    public void run() {
        try {
            Response response = call.execute();
            Log.d(TAG, "run: " + response.body().string());
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}).start();
```
post方式提交json字符串数据【这个就简单做一下，我也只做到提交json数据】

```java
MediaType mediaType = MediaType.parse("text/x-markdown; charset=utf-8");
JSONObject jsonObject = new JSONObject();
jsonObject.put("test",111);
String requestBody = jsonObject.toString();
Request request = new Request.Builder()
        .url("https://api.github.com/markdown/raw")
        .post(RequestBody.create(mediaType, requestBody))
        .build();
OkHttpClient okHttpClient = new OkHttpClient();
okHttpClient.newCall(request).enqueue(new Callback() {
    @Override
    public void onFailure(Call call, IOException e) {
        Log.d(TAG, "onFailure: " + e.getMessage());
    }

    @Override
    public void onResponse(Call call, Response response) throws IOException {
        Log.d(TAG, response.protocol() + " " +response.code() + " " + response.message());
        Headers headers = response.headers();
        for (int i = 0; i < headers.size(); i++) {
            Log.d(TAG, headers.name(i) + ":" + headers.value(i));
        }
        Log.d(TAG, "onResponse: " + response.body().string());
    }
});
```

## okhttp 拦截器概念

> OkHttp的拦截器链可谓是其整个框架的精髓，用户可传入的 interceptor 分为两类：
①一类是全局的 interceptor，该类 interceptor 在整个拦截器链中最早被调用，通过 OkHttpClient.Builder#addInterceptor(Interceptor) 传入；
②另外一类是非网页请求的 interceptor ，这类拦截器只会在非网页请求中被调用，并且是在组装完请求之后，真正发起网络请求前被调用，所有的 interceptor 被保存在 List<Interceptor> interceptors 集合中，按照添加顺序来逐个调用，具体可参考 RealCall#getResponseWithInterceptorChain() 方法。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191031155425327.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTM5MDcxNDg=,size_16,color_FFFFFF,t_70)
新版的okhttp已经使用kotlin重写了。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191031155656183.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTM5MDcxNDg=,size_16,color_FFFFFF,t_70)

那么就这个样了，加油~

ヾ(◍°∇°◍)ﾉﾞ
