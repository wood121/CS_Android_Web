&emsp;&emsp;本篇是作为**[Okhttp3的使用与解析](https://www.jianshu.com/p/d6f35ccf73d0)** 拦截器模块的补充。

### 一、拦截器在Okhttp源码中的应用
##### 1.RetryAndFollowUpInterceptor
```
    public Response intercept(Chain chain) throws IOException {
        Request request = chain.request();
        //创建StreamAllocation对象
        this.streamAllocation = new StreamAllocation(this.client.connectionPool(), this.createAddress(request.url()), this.callStackTrace);
        int followUpCount = 0;
        Response priorResponse = null;

        while(!this.canceled) {
            Response response = null;
            boolean releaseConnection = true;

            try {
                //通过RealInterceptorChain对象调用下一个拦截器，并从中获取响应Response
                response = ((RealInterceptorChain)chain).proceed(request, this.streamAllocation, (HttpCodec)null, (RealConnection)null);
                releaseConnection = false;
            } catch (RouteException var13) {
                if(!this.recover(var13.getLastConnectException(), false, request)) {
                    throw var13.getLastConnectException();
                }

                releaseConnection = false;
                continue;
            } catch (IOException var14) {
                boolean requestSendStarted = !(var14 instanceof ConnectionShutdownException);
                if(!this.recover(var14, requestSendStarted, request)) {
                    throw var14;
                }

                releaseConnection = false;
                continue;
            } finally {
                if(releaseConnection) {
                    this.streamAllocation.streamFailed((IOException)null);
                    this.streamAllocation.release();
                }

            }
            
            // 如果之前发生过重定向，且priorResponse不为空，则创建新的Response，并将ResponseBody设置为空状态。
            if(priorResponse != null) {
                response = response.newBuilder().priorResponse(priorResponse.newBuilder().body((ResponseBody)null).build()).build();
            }
            
            //followUpRequest涉及到 HTTP 中认证质询、重定向和重试等协议的实现
            Request followUp = this.followUpRequest(response);
            //若 followUp 重试请求为空，则当前请求结束，并返回当前的响应
            if(followUp == null) {
                if(!this.forWebSocket) {
                    this.streamAllocation.release();
                }
                return response;
            }
             //关闭响应结果
            Util.closeQuietly(response.body());

          //对重定向过程中一
            ++followUpCount;
            if(followUpCount > 20) {
                this.streamAllocation.release();
                throw new ProtocolException("Too many follow-up requests: " + followUpCount);
            }

            if(followUp.body() instanceof UnrepeatableRequestBody) {
                this.streamAllocation.release();
                throw new HttpRetryException("Cannot retry streamed HTTP body", response.code());
            }

            if(!this.sameConnection(response, followUp.url())) {
                this.streamAllocation.release();
                this.streamAllocation = new StreamAllocation(this.client.connectionPool(), this.createAddress(followUp.url()), this.callStackTrace);
            } else if(this.streamAllocation.codec() != null) {
                throw new IllegalStateException("Closing the body of " + response + " didn\'t close its backing stream. Bad interceptor?");
            }

            request = followUp;
            priorResponse = response;
        }

        this.streamAllocation.release();
        throw new IOException("Canceled");
    }
```
&emsp;&emsp;followUpRequest(response)，做验证、重定向、失败重连。
```
    private Request followUpRequest(Response userResponse) throws IOException {
        if(userResponse == null) {
            throw new IllegalStateException();
        } else {
            RealConnection connection = this.streamAllocation.connection();
            Route route = connection != null?connection.route():null;
            int responseCode = userResponse.code();
            String method = userResponse.request().method();
            switch(responseCode) {
            case 307:
            case 308:
                if(!method.equals("GET") && !method.equals("HEAD")) {
                    return null;
                }
            case 300:
            case 301:
            case 302:
            case 303:
                if(!this.client.followRedirects()) {
                    return null;
                } else {
                    String location = userResponse.header("Location");
                    if(location == null) {
                        return null;
                    } else {
                        HttpUrl url = userResponse.request().url().resolve(location);
                        if(url == null) {
                            return null;
                        } else {
                            boolean sameScheme = url.scheme().equals(userResponse.request().url().scheme());
                            if(!sameScheme && !this.client.followSslRedirects()) {
                                return null;
                            }

                            Builder requestBuilder = userResponse.request().newBuilder();
                            if(HttpMethod.permitsRequestBody(method)) {
                                boolean maintainBody = HttpMethod.redirectsWithBody(method);
                                if(HttpMethod.redirectsToGet(method)) {
                                    requestBuilder.method("GET", (RequestBody)null);
                                } else {
                                    RequestBody requestBody = maintainBody?userResponse.request().body():null;
                                    requestBuilder.method(method, requestBody);
                                }

                                if(!maintainBody) {
                                    requestBuilder.removeHeader("Transfer-Encoding");
                                    requestBuilder.removeHeader("Content-Length");
                                    requestBuilder.removeHeader("Content-Type");
                                }
                            }

                            if(!this.sameConnection(userResponse, url)) {
                                requestBuilder.removeHeader("Authorization");
                            }

                            return requestBuilder.url(url).build();
                        }
                    }
                }
            case 401:
                return this.client.authenticator().authenticate(route, userResponse);
            case 407:
                Proxy selectedProxy = route != null?route.proxy():this.client.proxy();
                if(selectedProxy.type() != Type.HTTP) {
                    throw new ProtocolException("Received HTTP_PROXY_AUTH (407) code while not using proxy");
                }

                return this.client.proxyAuthenticator().authenticate(route, userResponse);
            case 408:
                if(userResponse.request().body() instanceof UnrepeatableRequestBody) {
                    return null;
                }

                return userResponse.request();
            default:
                return null;
            }
        }
    }
```
这里我向说明的是case 401，我们可以利用到的authenticate比较重要，了解一下。

##### 2. BridgeInterceptor
>主要功能：
请求从应用层数据类型类型转化为网络调用层的数据类型。
将网络层返回的数据类型 转化为 应用层数据类型：
 &emsp;&emsp; 1. 保存最新的cookie(默认没有cookie，需要应用程序自己创建，详见[Cookie的API](https://square.github.io/okhttp/3.x/okhttp/okhttp3/CookieJar.html)和[Cookie的持久化](https://segmentfault.com/a/1190000004345545)；
 &emsp;&emsp; 2. 如果request中使用了"gzip"压缩，则进行Gzip解压。解压完毕后移除Header中的"Content-Encoding"和"Content-Length"（因为Header中的长度对应的是压缩前数据的长度，解压后长度变了，所以Header中长度信息实效了）；
 &emsp;&emsp; 3. 返回response。
```
    public Response intercept(Chain chain) throws IOException {
        Request userRequest = chain.request();
        Builder requestBuilder = userRequest.newBuilder();
        RequestBody body = userRequest.body();
        if(body != null) {
            MediaType transparentGzip = body.contentType();
            if(transparentGzip != null) {
                requestBuilder.header("Content-Type", transparentGzip.toString());
            }

            long cookies = body.contentLength();
            if(cookies != -1L) {
                requestBuilder.header("Content-Length", Long.toString(cookies));
                requestBuilder.removeHeader("Transfer-Encoding");
            } else {
                requestBuilder.header("Transfer-Encoding", "chunked");
                requestBuilder.removeHeader("Content-Length");
            }
        }

        if(userRequest.header("Host") == null) {
            requestBuilder.header("Host", Util.hostHeader(userRequest.url(), false));
        }

        if(userRequest.header("Connection") == null) {
            requestBuilder.header("Connection", "Keep-Alive");
        }

        boolean transparentGzip1 = false;
        if(userRequest.header("Accept-Encoding") == null && userRequest.header("Range") == null) {
            transparentGzip1 = true;
            requestBuilder.header("Accept-Encoding", "gzip");
        }

        List cookies1 = this.cookieJar.loadForRequest(userRequest.url());
        if(!cookies1.isEmpty()) {
            requestBuilder.header("Cookie", this.cookieHeader(cookies1));
        }

        if(userRequest.header("User-Agent") == null) {
            requestBuilder.header("User-Agent", Version.userAgent());
        }

        Response networkResponse = chain.proceed(requestBuilder.build());
        HttpHeaders.receiveHeaders(this.cookieJar, userRequest.url(), networkResponse.headers());
        okhttp3.Response.Builder responseBuilder = networkResponse.newBuilder().request(userRequest);
        if(transparentGzip1 && "gzip".equalsIgnoreCase(networkResponse.header("Content-Encoding")) && HttpHeaders.hasBody(networkResponse)) {
            GzipSource responseBody = new GzipSource(networkResponse.body().source());
            Headers strippedHeaders = networkResponse.headers().newBuilder().removeAll("Content-Encoding").removeAll("Content-Length").build();
            responseBuilder.headers(strippedHeaders);
            responseBuilder.body(new RealResponseBody(strippedHeaders, Okio.buffer(responseBody)));
        }

        return responseBuilder.build();
    }
```

##### 3. CacheInterceptor



##### 4. ConnectInterceptor


##### 5. CallServerInterceptor

### 二、拦截器的自定义
&emsp;&emsp;我们看下interceptor接口，其中我们的chain的子类实现是RealInterceptorChain，我们自定义的拦截器最重要的是intercept()方法。
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

public final class RealInterceptorChain implements Chain{
    ...
    //构造方法
    public RealInterceptorChain(List<Interceptor> interceptors, StreamAllocation streamAllocation, HttpCodec httpCodec, RealConnection connection, int index, Request request) {
        this.interceptors = interceptors;
        this.connection = connection;
        this.streamAllocation = streamAllocation;
        this.httpCodec = httpCodec;
        this.index = index;
        this.request = request;
    }
}
```
&emsp;&emsp;我们再来自定义一个日志拦截器
```
public class Wood121LoggerInceptor implements Interceptor {

    private static final Charset UTF8 = Charset.forName("UTF-8");

    @Override
    public Response intercept(Chain chain) throws IOException {
        //reqest、response都可以获取到
        Request request = chain.request();
        long startTime = System.currentTimeMillis();
        Response response = chain.proceed(request);
        long endTime = System.currentTimeMillis();
        long durationTime = endTime - startTime;

        BufferedSource source = response.body().source();
        source.request(Long.MAX_VALUE);
        Buffer buffer = source.buffer();
        String log = "\n================="
                .concat("\nnetwork code ==== " + response.code())
                .concat("\nnetwork url ===== " + request.url())
                .concat("\nduration ======== " + durationTime)
                .concat("\nrequest duration ============ " + (response.receivedResponseAtMillis() - response.sentRequestAtMillis()))
                .concat("\nrequest header == " + request.headers())
                .concat("\nrequest ========= " + bodyToString(request.body()))
                .concat("\nbody ============ " + buffer.clone().readString(UTF8));
        Log.e("wood121", "Wood121LoggerInceptor: " + log);

        return response;
    }

    /**
     * 请求体转String
     *
     * @param request 请求体
     * @return String 类型的请求体
     */
    private static String bodyToString(final RequestBody request) {
        try {
            final Buffer buffer = new Buffer();
            request.writeTo(buffer);
            return buffer.readUtf8();
        } catch (final Exception e) {
            return "did not work";
        }
    }
}
```

##### 相关参考
[Okhttp3源码解析](https://www.jianshu.com/p/dd412b8ba43f)
[OKHttp3 源码解析之拦截器（二）](http://lijiankun24.com/OkHttp3-%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%E4%B9%8B%E6%8B%A6%E6%88%AA%E5%99%A8/)
