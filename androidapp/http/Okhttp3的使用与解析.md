### 一、概述
&emsp;&emsp;官方文档[OkHttp](https://square.github.io/okhttp/) 中对Okhtttp有一段这样的介绍，了解一下：
*An HTTP+HTTP/2 client for Android and Java applications。that’s efficient by default:
HTTP/2 support allows all requests to the same host to share a socket.
Connection pooling reduces request latency (if HTTP/2 isn’t available).
Transparent GZIP shrinks download sizes.
Response caching avoids the network completely for repeat requests.*

### 二、基本使用
>一些简单使用：
&emsp;&emsp;一般get请求、 一般post请求；
&emsp;&emsp;基于http的文件上传、文件下载、加载图片；
&emsp;&emsp;支持请求回调，直接返回对象、对象集合
&emsp;&emsp;支持session的保持

OkHttpClient okHttpClient = new OkHttpClient(); 对象先创建。
###### 1.Get请求
&emsp;&emsp;一般的Get请求、文件下载；
```
    /**
     * 一般异步Get请求
     */
    private void getAsy() {
        Request request = new Request.Builder().url("http://www.baidu.com").method("GET", null).build();
        Call call = okHttpClient.newCall(request);
        call.enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {

            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {
                final String string = response.body().string();
                /**
                 * 注意：这个执行的线程不是ui线程，如果要操作控件，需要使用handler
                 */
                runOnUiThread(new Runnable() {
                    @Override
                    public void run() {
                        btnGet.setText("btn传回的数据：" + string);
                    }
                });
            }
        });
    }

    /**
     * 异步GET请求下载文件
     */
    private void getFile() {
        //2.创建Request对象，设置一个url地址（百度地址）,设置请求方式。
        Request request = new Request.Builder().url("https://www.baidu.com/img/bd_logo1.png").get().build();

        okHttpClient.newCall(request).enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {

            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {
                //以流的形式读取
                InputStream is = response.body().byteStream();
                //设置下载图片存储路径和名称
                int len = 0;
                File file = new File(Environment.getExternalStorageDirectory(), "baidu.png");
                FileOutputStream fos = new FileOutputStream(file);
                byte[] buf = new byte[128];
                while ((len = is.read(buf)) != -1) {
                    fos.write(buf, 0, len);
                    Log.e("wood121", "onResponse: " + len);
                }
                fos.flush();
                fos.close();
                is.close();

                //或者直接设置到imageView中
                Bitmap bitmap = BitmapFactory.decodeStream(is);
                runOnUiThread(new Runnable() {
                    @Override
                    public void run() {
                        //子线程刷新UI
                    }
                });
            }
        });

    }
```
###### 2.Post请求
&emsp;&emsp;POST请求提交键值对、提交字符串、上传文件、上传Multipart文件。
```
    /**
     * post请求提交键值对
     */
    private void postAsy() {
        FormBody formBody = new FormBody.Builder().add("name", "zhangqilu").add("age", "12").build();
        Request request = new Request.Builder().url("url").post(formBody).build();
        Call call = okHttpClient.newCall(request);
        call.enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {

            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {
                Log.e("wood121", response.body().toString());
            }
        });
    }

   /**
     * 异步POST请求提交字符串
     */
    private void postString() {
        MediaType mediaType = MediaType.parse("application/json; charset=utf-8");//"类型,字节码"
        //字符串
        String value = "{username:admin;password:admin}";
        RequestBody requestBody = RequestBody.create(mediaType, value);
        Request request = new Request.Builder().url("url").build();

        okHttpClient.newCall(request).enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {

            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {

            }
        });
    }

    /**
     * 异步POST请求上传文件
     */
    private void postFile() {
        MediaType mediaType = MediaType.parse("application/octet-stream");
        //上传的图片
        File file = new File(Environment.getExternalStorageDirectory(), "zhuangqilu.png");
        //2.通过RequestBody.create 创建requestBody对象,application/octet-stream 表示文件是任意二进制数据流
        RequestBody requestBody = RequestBody.create(mediaType, file);
        //3.创建Request对象，设置URL地址，将RequestBody作为post方法的参数传入
        Request request = new Request.Builder().url("url").post(requestBody).build();

        okHttpClient.newCall(request).enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {

            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {

            }
        });
    }

   /**
     * 异步POST请求上传Multipart文件
     */
    private void postMultipart() {
        //上传的图片
        File file = new File(Environment.getExternalStorageDirectory(), "zhuangqilu.png");

        MultipartBody multipartBody = new MultipartBody.Builder()
                .setType(MultipartBody.FORM)
                .addFormDataPart("username", "hahha")
                .addFormDataPart("age", "hehh")
                .addFormDataPart("imge", "zhangliu.png", RequestBody.create(MediaType.parse("imge/png"), file))
                .build();

        Request request = new Request.Builder().url("url").post(multipartBody).build();
        okHttpClient.newCall(request).enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {

            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {

            }
        });
    }
```

### 三、源码分析
&emsp;&emsp;首先放上两张图，对Okhttp的分层与调用流程有个大概的了解：
![图一：ok3jiagou.png](https://upload-images.jianshu.io/upload_images/4330197-593ded46688cb22e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![图二：okhttp_full_process.png](https://upload-images.jianshu.io/upload_images/4330197-53977da81310f2fa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 1.OkhttpClient、Call、Request、Response
###### 1）OkhttpClient 与 Call
&emsp;&emsp;OkhttpClient对象的创建过程，通过Builder模式。
```
public OkHttpClient() {
        this(new OkHttpClient.Builder());
}

public Builder() {
            this.dispatcher = new Dispatcher();
            this.protocols = OkHttpClient.DEFAULT_PROTOCOLS;
            this.connectionSpecs = OkHttpClient.DEFAULT_CONNECTION_SPECS;
            this.eventListenerFactory = EventListener.factory(EventListener.NONE);
            this.proxySelector = ProxySelector.getDefault();
            this.cookieJar = CookieJar.NO_COOKIES;
            this.socketFactory = SocketFactory.getDefault();
            this.hostnameVerifier = OkHostnameVerifier.INSTANCE;
            this.certificatePinner = CertificatePinner.DEFAULT;
            this.proxyAuthenticator = Authenticator.NONE;
            this.authenticator = Authenticator.NONE;
            this.connectionPool = new ConnectionPool();
            this.dns = Dns.SYSTEM;
            this.followSslRedirects = true;
            this.followRedirects = true;
            this.retryOnConnectionFailure = true;
            this.connectTimeout = 10000;
            this.readTimeout = 10000;
            this.writeTimeout = 10000;
            this.pingInterval = 0;
 }
```
&emsp;&emsp;Call接口与相关类
```
//Call相关的继承关系
public interface Call extends Cloneable {
    Request request();

    Response execute() throws IOException;
    void enqueue(Callback var1);
    void cancel();

    boolean isExecuted();
    boolean isCanceled();

    Call clone();
    public interface Factory {
        Call newCall(Request var1);
    }
}

//RealCall是作为Call的具体实现，里面重点execute()和enqueue(Callback responseCallback)两个方法，这会在后续调用流程中再分析。
final class RealCall implements Call

//okHttpClient实现了Call中的Factory接口，实现了newCall方法
public class OkHttpClient implements Factory
public Call newCall(Request request) {
    return new RealCall(this, request, false);
}
```
###### 2）Request与Response
&emsp;&emsp;Request的相关源码
```
public final class Request {
    //url字符串和端口号信息，默认端口号：http为80，https为443.其他自定义信息
    final HttpUrl url;  
    //"get","post","head","delete","put"....
    final String method;
    //包含了请求的头部信息，name和value对。
    final Headers headers;
    //请求的数据内容
    @Nullable
    final RequestBody body;
    //请求的附加字段。对资源文件的一种摘要。保存在头部信息中：ETag: "5694c7ef-24dc"。客户端可以在二次请求的时候，在requst的头部添加缓存的tag信息（如If-None-Match:"5694c7ef-24dc"），服务端用改信息来判断数据是否发生变化。
    final Object tag;
    ...
}

//RequestBody中最重要的create方法
public static RequestBody create(@Nullable MediaType contentType, String content) {...}
public static RequestBody create(@Nullable final MediaType contentType, final ByteString content) {...}
public static RequestBody create(@Nullable MediaType contentType, byte[] content) {...}
public static RequestBody create(@Nullable final MediaType contentType, final byte[] content, final int offset, final int byteCount) {...}
public static RequestBody create(@Nullable final MediaType contentType, final File file) {...}
//RequestBody自身可上传string与文件，有两个继承类：FormBody和MultipartBody分别对应提交键值对和上传Multipart文件。
public final class FormBody extends RequestBody{...}
public final class MultipartBody extends RequestBody {...}
```
&emsp;&emsp;Response的相关源码
```
public final class Response implements Closeable {
    //网络请求的信息
    final Request request;
    //网络协议，okhttp支持"http/1.0","http/1.1","h2"和"spdy/3.1" 
    final Protocol protocol;
    //返回状态码 
    final int code;
    //状态信息，与状态码相对应
    final String message;
    //TLS(传输层安全协议)的握手信息（包含协议版本，密码套件(https://en.wikipedia.org/wiki/Cipher_suite)，证书列表
    @Nullable
    final Handshake handshake;
    //相应的头信息，格式与请求的头信息相同。
    final Headers headers;
    //数据内容在ResponseBody中
    @Nullable
    final ResponseBody body;
    //网络返回的原声数据(如果未使用网络，则为null)
    @Nullable
    final Response networkResponse;
    //从cache中读取的网络原生数据
    @Nullable
    final Response cacheResponse;
    //网络重定向后的，存储的上一次网络请求返回的数据。
    @Nullable
    final Response priorResponse;
    //发起请求的时间轴
    final long sentRequestAtMillis;
    //收到返回数据时的时间轴
    final long receivedResponseAtMillis;
    //缓存控制指令，由服务端返回数据的中的Header信息指定，或者客户端发器请求的Header信息指定。key："Cache-Control"
      //详见<a href="http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.9">RFC 2616,14.9</a>
    private volatile CacheControl cacheControl;
...
}
```
&emsp;&emsp;我们使用的Retrofit2，它提供了接口converter将自定义的数据对象（各类自定义的request和response）和OKHttp3中网络请求的数据类型（ReqeustBody和ResponseBody）进行转换。在[Retrofit2的使用与解析](https://www.jianshu.com/p/0bbac3a5f196)中，我有进行相关的Converter自定义，实现构建OkHttp3中的RequestBody和ResponseBody。一般我们使用Converter来转换不规范的返回数据，或者对数据进行加解密传输。

##### .请求的分发与线程池技术
###### 1）Call的创建过程
&emsp;&emsp;Call的创建过程，实际是创建了RealCall对象：
```
 public Call newCall(Request request) {
        return new RealCall(this, request, false);
}

RealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
        Factory eventListenerFactory = client.eventListenerFactory();
        this.client = client;
        this.originalRequest = originalRequest;
        //这个forWebSocket，一般情况下没有用到，WebSocket相关内容可以另外了解
        this.forWebSocket = forWebSocket;
        this.retryAndFollowUpInterceptor = new RetryAndFollowUpInterceptor(client, forWebSocket);
        this.eventListener = eventListenerFactory.create(this);
}
```
###### 2）Dispatcher类
&emsp;&emsp;这个Dispatcher对象是在OkhttpClient的builder构造中进行创建的，上面讲OkhttpClient有提到。我们现在看看它内部的重要参数与方法：
```
//异步等待的任务
private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque();
//异步正在执行的任务
private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque();
// 同步正在执行的任务
private final Deque<RealCall> runningSyncCalls = new ArrayDeque();

//同步操作，可以看到这里只是将Call存入了runningSyncCalls
synchronized void executed(RealCall call) {
        this.runningSyncCalls.add(call);
}

//异步操作，如果当前正在执行的call数量大于maxRequests,64,
//或者该call的Host上的call超过maxRequestsPerHost，5，则加入readyAsyncCalls排队等待。
//否则加入runningAsyncCalls，并执行。
synchronized void enqueue(AsyncCall call) {
        if(this.runningAsyncCalls.size() < this.maxRequests && this.runningCallsForHost(call) < this.maxRequestsPerHost) {
            this.runningAsyncCalls.add(call);
            this.executorService().execute(call);
        } else {
            this.readyAsyncCalls.add(call);
       }
}
//Okhttp中创建的线程池， new SynchronousQueue()使用的同步队列，先来先服务。
public synchronized ExecutorService executorService() {
        if(this.executorService == null) {
            this.executorService = new ThreadPoolExecutor(0, 2147483647, 60L, TimeUnit.SECONDS, new SynchronousQueue(), Util.threadFactory("OkHttp Dispatcher", false));
        }
        return this.executorService;
}
```

###### 3）同步请求
```
//RealCall中发起同步请求的方法
public Response execute() throws IOException {
        synchronized(this) {
            if(this.executed) {
                throw new IllegalStateException("Already Executed");
            }

            this.executed = true;
        }

        this.captureCallStackTrace();

        Response var2;
        try {
            this.client.dispatcher().executed(this);
            Response result = this.getResponseWithInterceptorChain();
            if(result == null) {
                throw new IOException("Canceled");
            }

            var2 = result;
        } finally {
            this.client.dispatcher().finished(this);
        }

        return var2;
}
```
this.client.dispatcher().executed(this); 这一步实际是将Call存起来而已；真正得到请求结果的是Response result = this.getResponseWithInterceptorChain();方法。最后的finished方法是任务完成后进行回调处理的，这个方法在异步请求过程中再进行分析。

###### 4）异步请求
```
//RealCall中发起的异步请求
public void enqueue(Callback responseCallback) {
        synchronized(this) {
            if(this.executed) {
                throw new IllegalStateException("Already Executed");
            }

            this.executed = true;
        }

        this.captureCallStackTrace();
        this.client.dispatcher().enqueue(new RealCall.AsyncCall(responseCallback));
}

//即调用了dispatcher中异步方法，这个之前有提到，
synchronized void enqueue(AsyncCall call) {
        if(this.runningAsyncCalls.size() < this.maxRequests && this.runningCallsForHost(call) < this.maxRequestsPerHost) {
            this.runningAsyncCalls.add(call);
            this.executorService().execute(call);
        } else {
            this.readyAsyncCalls.add(call);
        }
}
```
&emsp;&emsp;通过上面的发起请求流程，接下来我们看看在dispatcher中真正的执行这个任务的过程
```
public abstract class NamedRunnable implements Runnable {
    protected final String name;
    public NamedRunnable(String format, Object... args) {
        this.name = Util.format(format, args);
    }

    public final void run() {
        String oldName = Thread.currentThread().getName();
        Thread.currentThread().setName(this.name);
        try {
            this.execute();
        } finally {
            Thread.currentThread().setName(oldName);
        }
    }
    protected abstract void execute();
}

final class AsyncCall extends NamedRunnable {
        private final Callback responseCallback;

        AsyncCall(Callback responseCallback) {
            super("OkHttp %s", new Object[]{RealCall.this.redactedUrl()});
            this.responseCallback = responseCallback;
        }

        String host() {
            return RealCall.this.originalRequest.url().host();
        }

        Request request() {
            return RealCall.this.originalRequest;
        }

        RealCall get() {
            return RealCall.this;
        }

        protected void execute() {
            boolean signalledCallback = false;

            try {
                Response e = RealCall.this.getResponseWithInterceptorChain();
                if(RealCall.this.retryAndFollowUpInterceptor.isCanceled()) {
                    signalledCallback = true;
                    this.responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
                } else {
                    signalledCallback = true;
                    this.responseCallback.onResponse(RealCall.this, e);
                }
            } catch (IOException var6) {
                if(signalledCallback) {
                    Platform.get().log(4, "Callback failure for " + RealCall.this.toLoggableString(), var6);
                } else {
                    this.responseCallback.onFailure(RealCall.this, var6);
                }
            } finally {
                RealCall.this.client.dispatcher().finished(this);
            }}}
```
&emsp;&emsp;AsyncCall是继承自NamedRunnable类的，NamedRunnable中的run调用了this.execute()即调用了子类的execute()方法，也就是AsyncCall的execute()方法。接之前走到dispatcher的enqueue方法中this.executorService().execute(call)，execute(call)可以知道最终是走到AsyncCall的execute()方法中，拿到请求结果也是Response e = RealCall.this.getResponseWithInterceptorChain();这个方法。
&emsp;&emsp;接下来看到finally中调用的RealCall.this.client.dispatcher().finished(this);
```
void finished(AsyncCall call) {
     this.finished(this.runningAsyncCalls, call, true);
}

private <T> void finished(Deque<T> calls, T call, boolean promoteCalls) {
        int runningCallsCount;
        Runnable idleCallback;
        synchronized(this) {
            if(!calls.remove(call)) {
                throw new AssertionError("Call wasn\'t in-flight!");
            }

            if(promoteCalls) {
                this.promoteCalls();
            }
           //这里统计总的call 数量，runningCallsCount()计算了所有正在运行的同步与异步任务。
            runningCallsCount = this.runningCallsCount();
            idleCallback = this.idleCallback;
        }
       //如果没有运行的任务，即回调
        if(runningCallsCount == 0 && idleCallback != null) {
            idleCallback.run();
        }
}

private void promoteCalls() {
        if(this.runningAsyncCalls.size() < this.maxRequests) {
            if(!this.readyAsyncCalls.isEmpty()) {
                Iterator i = this.readyAsyncCalls.iterator();

                do {
                    if(!i.hasNext()) {
                        return;
                    }

                    AsyncCall call = (AsyncCall)i.next();
                    if(this.runningCallsForHost(call) < this.maxRequestsPerHost) {
                        i.remove();
                        this.runningAsyncCalls.add(call);
                        this.executorService().execute(call);
                    }
                } while(this.runningAsyncCalls.size() < this.maxRequests);

            }
        }
 }
```
可以看到我们在完成RealCall.AsyncCall的任务时，会调用finished方法，这个方法又会调用promoteCalls方法，在promoteCalls中完成readyAsyncCalls存储的任务向runningAsyncCalls转换，并同时执行Call任务。注意，在同步任务中，finish中传入的false，就不会调用
promoteCalls方法，直接走后面的判断，然后确认是否进行回调处理。

##### 3.拦截器
&emsp;&emsp;在上面的分析过程中，我们看到不管是同步还是异步请求，最终都是调用的getResponseWithInterceptorChain()获取到请求结果，接下来我们看看里面的内部实现：
```
    Response getResponseWithInterceptorChain() throws IOException {
        //创建一个拦截器列表
        ArrayList interceptors = new ArrayList();
         //优先处理自定义拦截器
        interceptors.addAll(this.client.interceptors());
        //失败重连拦截器
        interceptors.add(this.retryAndFollowUpInterceptor);
        //接口桥接拦截器(同时处理cookie逻辑)
        interceptors.add(new BridgeInterceptor(this.client.cookieJar()));
        //缓存拦截器
        interceptors.add(new CacheInterceptor(this.client.internalCache()));
        //分配连接拦截器
        interceptors.add(new ConnectInterceptor(this.client));
        //web的socket连接的网络配置拦截器
        if(!this.forWebSocket) {
            interceptors.addAll(this.client.networkInterceptors());
        }
        //最后是连接服务器发起真正的网络请求的拦截器
        interceptors.add(new CallServerInterceptor(this.forWebSocket));
        RealInterceptorChain chain = new RealInterceptorChain(interceptors, (StreamAllocation)null, (HttpCodec)null, (RealConnection)null, 0, this.originalRequest);
        return chain.proceed(this.originalRequest);
    }
```
一个网络请求，按一定的顺序，经由多个拦截器进行处理，该拦截器可以决定自己处理并且返回我的结果，也可以选择向下继续传递，让后面的拦截器处理返回它的结果。这个设计模式叫做 **责任链模式**。一个拦截器可以拦截请求和响应，获取或修改其中的信息，这在编程中是非常有用的。不仅在源码中拦截器使用的很广泛，开发者也可以根据自己的需求自定义拦截器，并将其加入到 OkHttpClient 对象中。具体后面应用场景会设计拦截器的自定义。
&emsp;&emsp;我们再细看RealInterceptorChain的构造和chain.proceed()方法的实现：
```
//RealConnection实现创建Socket对象，连接到目标网络的功能；
//StreamAllocation进行输入输出流操作
public RealInterceptorChain(List<Interceptor> interceptors, StreamAllocation streamAllocation, HttpCodec httpCodec, RealConnection connection, int index, Request request) {
        this.interceptors = interceptors;
        this.connection = connection;
        this.streamAllocation = streamAllocation;
        this.httpCodec = httpCodec;
        this.index = index;
        this.request = request;
 }

public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec, RealConnection connection) throws IOException {
        if(this.index >= this.interceptors.size()) {
            throw new AssertionError();
        } else {
            ++this.calls;
            if(this.httpCodec != null && !this.connection.supportsUrl(request.url())) {
                throw new IllegalStateException("network interceptor " + this.interceptors.get(this.index - 1) + " must retain the same host and port");
            } else if(this.httpCodec != null && this.calls > 1) {
                throw new IllegalStateException("network interceptor " + this.interceptors.get(this.index - 1) + " must call proceed() exactly once");
            } else {
                RealInterceptorChain next = new RealInterceptorChain(this.interceptors, streamAllocation, httpCodec, connection, this.index + 1, request);
                Interceptor interceptor = (Interceptor)this.interceptors.get(this.index);
                Response response = interceptor.intercept(next);
                if(httpCodec != null && this.index + 1 < this.interceptors.size() && next.calls != 1) {
                    throw new IllegalStateException("network interceptor " + interceptor + " must call proceed() exactly once");
                } else if(response == null) {
                    throw new NullPointerException("interceptor " + interceptor + " returned null");
                } else {
                    return response;
                }
            }
        }
 }
```
这里我们可以看到interceptor.intercept(next)，这个next是基于本次的RealInterceptorChain对象将this.index+1传入创建链的下一级，一次次传递最终达到最后的interceptor。顺序是由interceptors的添加顺序决定的。我们这里可以看看Interceptro接口与目前已实现的Interceptors：
```
public interface Interceptor {
    Response intercept(Interceptor.Chain var1) throws IOException;
    public interface Chain {
        Request request();
        Response proceed(Request var1) throws IOException;
        @Nullable
        Connection connection();
    }
}
```
![图三：Interceptors](https://upload-images.jianshu.io/upload_images/4330197-5830ddaa29a5a76f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

&emsp;&emsp;为了不让本篇过长，拦截器的源码解析与自定义拆分另一篇：**[Okhttp3的拦截器解析与自定义](https://www.jianshu.com/p/ce0ec70c6892)**。

##### 4.连接池
...... 

### 四、应用场景
##### 1.封装应用
 [hongyangAndroid](https://github.com/hongyangAndroid)/**[okhttputils](https://github.com/hongyangAndroid/okhttputils)**，已停止维护。
[jeasonlzy](https://github.com/jeasonlzy)/**[okhttp-OkGo](https://github.com/jeasonlzy/okhttp-OkGo)**，基于 Http 协议，封装了 OkHttp 的网络请求框架，支持 RxJava，RxJava2，支持自定义缓存，支持批量断点下载管理和批量上传管理功能。
##### 2.与https
```
OkHttpClient okHttpClient = new OkHttpClient.Builder()
		// 创建一个证书工厂
        .sslSocketFactory(SSLSocketFactory, X509TrustManager) 
        .build();
```
##### 3.与WebSocket
```
// 方法可以选择实现
OkHttpClient okHttpClient = new OkHttpClient.Builder()
        .build();
Request request = new Request.Builder()
        .url("https://wwww.xxx.com")
        .build();
okHttpClient.newWebSocket(request, new WebSocketListener() {
    @Override
    public void onMessage(WebSocket webSocket, String text) {
        super.onMessage(webSocket, text);
      // 当收到文本消息
    }

    @Override
    public void onOpen(WebSocket webSocket, Response response) {
        super.onOpen(webSocket, response);
       // 连接成功
    }

    @Override
    public void onMessage(WebSocket webSocket, ByteString bytes) {
        super.onMessage(webSocket, bytes);
       // 收到字节消息，可转换为文本
    }

    @Override
    public void onClosed(WebSocket webSocket, int code, String reason) {
        super.onClosed(webSocket, code, reason);
       // 连接被关闭
    }

    @Override
    public void onFailure(WebSocket webSocket, Throwable t, Response response) {
        super.onFailure(webSocket, t, response);
       // 连接失败
    }
});
```

###### 相关参考
[okhttp的github主页](https://github.com/square/okhttp)
[okhttp的官方文档](https://square.github.io/okhttp/)
[Android OkHttp完全解析 是时候来了解OkHttp了](https://blog.csdn.net/lmj623565791/article/details/47911083)
[Android OkHttp3简介和使用详解](https://blog.csdn.net/zhangqiluGrubby/article/details/71480546)
[关于Okhttp（一）-基本使用](http://lowett.com/2017/02/09/okhttp-1/)
[OKHttp3源码解析](https://www.jianshu.com/p/dd412b8ba43f)
[拆轮子系列：拆 OkHttp](https://blog.piasy.com/2016/07/11/Understand-OkHttp/)
[OKHTTP3源码和设计模式-1](http://www.liuguangli.win/archives/769)
[OKHTTP3源码2-连接池管理](https://zhuanlan.zhihu.com/p/35537037)
