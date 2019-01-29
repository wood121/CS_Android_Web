### 一、概述
>**app/build.gradle中的配置**
compile 'com.squareup.retrofit2:retrofit:2.3.0'
compile 'com.squareup.retrofit2:converter-gson:2.3.0'
compile 'com.squareup.retrofit2:adapter-rxjava2:2.3.0'

>&emsp;&emsp;我们来看下源码中注释文档对Retrofit的概括：“ Retrofit adapts a Java interface to HTTP calls by using annotations on the declared methods to define how requests are made. Create instances using Builder() and pass your interface to  create() to generate an implementation.”

### 二、基本使用
##### 1.get请求
###### 1) 没有数据传输
```
/**
**************第一步：独立创建的请求接口**************
*/
public interface MyHttpService {
    /**
     * 注解中传入网络请求的 部分URL
     * @return
     */
    @GET("ajax.php?a=fy&f=auto&t=auto&w=hello%20world")
    Call<Reception> getCall();
}

/**
**************第二步：创建后台数据反馈bean类**************
*/
public class Reception2 {
    private String type;
    private int errorCode;
    private int elapsedTime;
    private List<List<TranslateResultBean>> translateResult;

    @Override
    public String toString() {
        return "Reception2{" +
                "type='" + type + '\'' +
                ", errorCode=" + errorCode +
                ", elapsedTime=" + elapsedTime +
                ", translateResult=" + translateResult.toString() +
                '}';
    }

    public String getType() {
        return type;
    }

    public void setType(String type) {
        this.type = type;
    }

    public int getErrorCode() {
        return errorCode;
    }

    public void setErrorCode(int errorCode) {
        this.errorCode = errorCode;
    }

    public int getElapsedTime() {
        return elapsedTime;
    }

    public void setElapsedTime(int elapsedTime) {
        this.elapsedTime = elapsedTime;
    }

    public List<List<TranslateResultBean>> getTranslateResult() {
        return translateResult;
    }

    public void setTranslateResult(List<List<TranslateResultBean>> translateResult) {
        this.translateResult = translateResult;
    }

    public static class TranslateResultBean {
        /**
         * src : merry me
         * tgt : 我快乐
         */

        public String src;
        public String tgt;

        public String getSrc() {
            return src;
        }

        public void setSrc(String src) {
            this.src = src;
        }

        public String getTgt() {
            return tgt;
        }

        public void setTgt(String tgt) {
            this.tgt = tgt;
        }

        @Override
        public String toString() {
            return "TranslateResultBean{" +
                    "src='" + src + '\'' +
                    ", tgt='" + tgt + '\'' +
                    '}';
        }
    }
}

/**
************第三步：在activity中创建对象、发起请求  **************
*/
//创建Retrofit对象
Retrofit retrofit = new Retrofit.Builder()
        .baseUrl("http://fy.iciba.com/")
        .addConverterFactory(GsonConverterFactory.create())
        .build();

//创建 网络请求接口 的实例
MyHttpService myHttpService = retrofit.create(MyHttpService.class);
Call<Reception> call = myHttpService.getCall();

//发起请求、得到请求回调
call.enqueue(new Callback<Reception>() {
//其中Reception是服务器返回的bean对象。
    @Override
    public void onResponse(Call<Reception> call, Response<Reception> response) {
        Reception reception = response.body();
        Log.e("url", "response:" + response.toString());
        Log.e("url", "reception:" + reception.toString());
    }
    @Override
    public void onFailure(Call<Reception> call, Throwable t) {
    }
});
```
###### 2）动态的url访问 @path
>例如：
//用于访问zhy的信息
http://192.168.1.102:8080/springmvc_users/user/zhy
//用于访问lmj的信息
http://192.168.1.102:8080/springmvc_users/user/lmj
```
public interface IUserBiz{
    @GET("{username}")
    Call<User> getUser(@Path("username") String username);
}

Call<User> call = userBiz.getUser("zhy");
```
###### 3）查询参数 @ Query
>http://baseurl/users?sortby=username
http://baseurl/users?sortby=id
```
@GET("user")
Call<Reception> getUserBySort(@Query("sortBy") String sort);

Call<List<User>> call = userBiz.getUsersBySort("username");
```

##### 2.post请求
###### 1）请求体的方式向服务器传入json字符串@Body
```
public interface IUserBiz{
 @POST("add")
 Call<List<User>> addUser(@Body User user);
}

Call<List<User>> call = userBiz.addUser(new User(1001, "jj", "123,", "jj123", "jj@qq.com"));
```
###### 2）表单的方式传递键值对@FormUrlEncoded
```
public interface IUserBiz{
    @POST("login")
    @FormUrlEncoded
    Call<User> login(@Field("username") String username, @Field("password") String password);
}

Call<User> call = userBiz.login("zhy", "123");
```
###### 3）单文件上传 @MultipartBody
```
public interface IUserBiz{
    @Multipart
    @POST("register")
    Call<User> registerUser(@Part MultipartBody.Part photo, @Part("username") RequestBody username, @Part("password") RequestBody password);
}

File file = new File(Environment.getExternalStorageDirectory(), "icon.png");
RequestBody photoRequestBody = RequestBody.create(MediaType.parse("image/png"), file);
MultipartBody.Part photo = MultipartBody.Part.createFormData("photos", "icon.png", photoRequestBody);
Call<User> call = userBiz.registerUser(photo, RequestBody.create(null, "abc"), RequestBody.create(null, "123"));
```
###### 4）多文件上传@PartMap
```
 public interface IUserBiz {
     @Multipart
     @POST("register")
      Call<User> registerUser(@PartMap Map<String, RequestBody> params,  @Part("password") RequestBody password);
}

File file = new File(Environment.getExternalStorageDirectory(), "messenger_01.png");
RequestBody photo = RequestBody.create(MediaType.parse("image/png", file);
Map<String,RequestBody> photos = new HashMap<>();
photos.put("photos\"; filename=\"icon.png", photo);
photos.put("username",  RequestBody.create(null, "abc"));
Call<User> call = userBiz.registerUser(photos, RequestBody.create(null, "123"));
```
###### 6)下载文件
>推荐使用okhttp封装好的[okhttpUtils](https://github.com/hongyangAndroid/okhttp-utils)

### 三、源码分析
##### 1.Retrofit对象的创建
```
public Builder() {
      this(Platform.get());
}

public Retrofit build() {
      if (baseUrl == null) {
        throw new IllegalStateException("Base URL required.");
      }

      okhttp3.Call.Factory callFactory = this.callFactory;
      if (callFactory == null) {
        callFactory = new OkHttpClient();
      }

      Executor callbackExecutor = this.callbackExecutor;
      if (callbackExecutor == null) {
        callbackExecutor = platform.defaultCallbackExecutor();
      }

      // Make a defensive copy of the adapters and add the default Call adapter.
      List<CallAdapter.Factory> adapterFactories = new ArrayList<>(this.adapterFactories);
      adapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));

      // Make a defensive copy of the converters.
      List<Converter.Factory> converterFactories = new ArrayList<>(this.converterFactories);

      return new Retrofit(callFactory, baseUrl, converterFactories, adapterFactories,
          callbackExecutor, validateEagerly);
}
```
可以看到Retrofit通过builder模式进行对象创建，在build()方法中配置相关的参数，其中 **baseUrl** 就是我们需要访问的地址。
```
//Call.Factory
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
OkhttpClient是Call.Factory的子类实现。

//callbackExecutor和CallAdapter.Factory
static class Android extends Platform {
    @Override public Executor defaultCallbackExecutor() {
      return new MainThreadExecutor();
    }
    //可以看到callbackExecutor就是利用handler将回调切换到主线程的线程池。
    static class MainThreadExecutor implements Executor {
      private final Handler handler = new Handler(Looper.getMainLooper());
      @Override public void execute(Runnable r) {
        handler.post(r);
      }

    @Override CallAdapter.Factory defaultCallAdapterFactory(@Nullable Executor callbackExecutor) {
      if (callbackExecutor == null) throw new AssertionError();
      return new ExecutorCallAdapterFactory(callbackExecutor);
    }
    }
}
final class ExecutorCallAdapterFactory extends CallAdapter.Factory{... }

//converterFactories就是能将数据转换的Converter对象
```
简单的梳理一下，请求地址baseUrl、callFactory是实现了Call.Factory接口的OkhttpClient、adapterFactories主要是Call进行转换（Android平台下CallAdapter.Factory的实现类ExecutorCallAdapterFactory）、callbackExecutor将回调传递到UI线程（也就是MainThreadExecutor对象，里面使用了主线程的Handler）、Converter.Factory需要我们传入（支持的类型：见[官网文档](http://square.github.io/retrofit/)）。

##### 2.为接口实现实例


### 四、配置OkhttpClient
### 五、自定义Converter.Factory



###### 相关参考
[retrofit官方文档](http://square.github.io/retrofit/)
[Retrofit2 完全解析 探索与okhttp之间的关系](https://blog.csdn.net/lmj623565791/article/details/51304204)
 [Retrofit用法详解（关于APIService中的注解）](http://duanyytop.github.io/2016/08/06/Retrofit%E7%94%A8%E6%B3%95%E8%AF%A6%E8%A7%A3/)
 [拆轮子系列：拆 Retrofit](https://blog.piasy.com/2016/06/25/Understand-Retrofit/)
[Retrofit2 源码解析](https://juejin.im/entry/59a17205518825244a435c66)

