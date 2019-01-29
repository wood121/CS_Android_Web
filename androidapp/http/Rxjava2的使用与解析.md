>**app/build.gradle中的配置**
compile 'io.reactivex.rxjava2:rxjava:2.0.7'
compile 'io.reactivex.rxjava2:rxandroid:2.0.1'

### 一、概述
##### 1.概述
&emsp;&emsp;Rx是ReactiveX的缩写，而ReactiveX是Reactive Extensions的缩写。Rxjava在github上主页的介绍："RxJava is a Java VM implementation of [Reactive Extensions](http://reactivex.io/): a library for composing asynchronous and event-based programs by using observable sequences."其根本就是一个实现异步操作的库。
##### 2.Rxjava的观察者模式
&emsp;&emsp;RxJava 的异步实现，是通过一种扩展的观察者模式来实现的。有四个基本概念：Observable (被观察者)、 Observer (观察者)、 subscribe (订阅)、事件。Observable 和 Observer 通过 subscribe() 方法实现订阅关系，从而 Observable 可以在需要的时候发出事件来通知 Observer。大致如图：
![Rxjava的观察者模式](https://upload-images.jianshu.io/upload_images/4330197-e2acdb722edd7124.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
&emsp;&emsp;其被观察者对象有Flowable, Observable, Single, Completable, Maybe。订阅逻辑是“被观察者.subscribe（观察者）”，这个点需要适应一下。

### 二、基本使用
##### 1、五种被观察者
###### 1）Flowable
&emsp;&emsp;基本使用
```
//被观察者
Flowable<String> flowable = Flowable.create(new FlowableOnSubscribe<String>() {
    @Override
    public void subscribe(FlowableEmitter<String> e) throws Exception {
        //订阅观察者时的操作
        e.onNext("test1");
        e.onNext("test2");
        e.onComplete();
    }
}, BackpressureStrategy.BUFFER);

//创建观察者
FlowableSubscriber<String> subscriber = new FlowableSubscriber<String>() {
    @Override
    public void onSubscribe(Subscription s) {
        //订阅时候的操作
        s.request(Long.MAX_VALUE);//请求多少事件，这里表示不限制
    }
    @Override
    public void onNext(String s) {
        //onNext事件处理，这里是顺序执行的，
        Log.i("tag", s);
    }
    @Override
    public void onError(Throwable t) {
        //onError事件处理
    }
    @Override
    public void onComplete() {
        //onComplete事件处理
    }
};

flowable.subscribe(subscriber);

/**
一个正常的事件队列情况应该如下:
onNext -> onNext ... -> onComplete
onNext -> onNext ... -> onError
*/
```
&emsp;&emsp;我们点开源码，可以看到subscribe有许多重载方法
```
subscribe(onNext)       //如果我们只关心onNext()获取的数据，调用这个方法即可
subscribe(onNext,onError)
subscribe(onNext,onError,onComplete)
subscribe(onNext,onError,onComplete,onSubscribe)

//我们实现一个完整参数的，其中的Consumer和Action都属于Actions
flowable.subscribe(
        new Consumer<String>() {//相当于onNext
            @Override
            public void accept(String s) throws Exception {
            }
        }, new Consumer<Throwable>() {//相当于onError
            @Override
            public void accept(Throwable throwable) throws Exception {
            }
        }, new Action() {//相当于onComplete，注意这里是Action
            @Override
            public void run() throws Exception {
            }
        }, new Consumer<Subscription>() {//相当于onSubscribe
            @Override
            public void accept(Subscription subscription) throws Exception {
            }
        });
```
###### 2) Observable
&emsp;&emsp;基本使用
```
//被观察者
Observable<String> observable = Observable.create(new ObservableOnSubscribe<String>() {
    @Override
    public void subscribe(ObservableEmitter<String> e) throws Exception {
        //订阅观察者时的操作
        e.onNext("test1");
        e.onNext("test2");
        e.onComplete();
    }
});

//观察者
Observer<String> observer = new Observer<String>() {
    @Override
    public void onSubscribe(Disposable d) {
        //订阅时候的操作，无需request
    }
    @Override
    public void onNext(String s) {
        //onNext事件处理
        Log.i("observer", s);
    }
    @Override
    public void onError(Throwable e) {
        //onError事件处理
    }
    @Override
    public void onComplete() {
        //onComplete事件处理
    }
};

observable.subscribe(observer);

//理，我们点开subscribe也可以看到许多重载方法，也可以与Flowable中一样订阅Actions。
```
&emsp;&emsp;看了Flowable和Observable的基本用法，我们会发现前者支持背压（即BackpressureStrategy），所谓的背压就是生产者（被观察者）的生产速度大于消费者（观察者）消费速度从而导致的问题。举一个简单点的例子，如果被观察者快速发送消息，但是观察者处理消息的很缓慢，如果没有特定的流（Flow）控制，就会导致大量消息积压占用系统资源，最终导致十分缓慢。官方给的建议：
>&emsp;&emsp;使用Observable - 不超过1000个元素、随着时间流逝基本不会出现OOM - GUI事件或者1000Hz频率以下的元素 - 平台不支持Java Steam(Java8新特性) - Observable开销比Flowable低
&emsp;&emsp;使用Flowable - 超过10k+的元素(可以知道上限) - 读取硬盘操作（可以指定读取多少行） - 通过JDBC读取数据库 - 网络（流）IO操作
###### 3) Single
&emsp;&emsp;如果你使用一个单一连续事件流，即只有一个onNext事件，接着就触发onComplete或者onError，这样你可以使用Single。**注意：从现在开始，我们的demo使用的Rxjava写起来特别舒服的链式调用！**
```
//被观察者
 Single.create(new SingleOnSubscribe<String>() {
    @Override
    public void subscribe(SingleEmitter<String> e) throws Exception {
        e.onSuccess("test");
        e.onSuccess("test2");//错误写法，重复调用也不会处理
    }
}).subscribe(new SingleObserver<String>() {//订阅观察者SingleObserver
    @Override
    public void onSubscribe(Disposable d) {

    }

    @Override
    public void onSuccess(String s) {
        //相当于onNext和onComplete
    }

    @Override
    public void onError(Throwable e) {

    }
});

//同理可查看subscribe的重载方法了解一下
```
###### 4) Completable
&emsp;&emsp;如果你的观察者连onNext事件都不关心，你可以使用Completable，他只有onComplete和onError两个事件：
```
Completable.create(new CompletableOnSubscribe() {//被观察者

    @Override
    public void subscribe(CompletableEmitter e) throws Exception {
        e.onComplete();//单一onComplete或者onError
    }

}).subscribe(new CompletableObserver() {//观察者
    @Override
    public void onSubscribe(Disposable d) {

    }

    @Override
    public void onComplete() {

    }

    @Override
    public void onError(Throwable e) {

    }
});
```
###### 5) Maybe
&emsp;&emsp;如果你有一个需求是可能发送一个数据或者不会发送任何数据，这时候你就需要Maybe，它类似于Single和Completable的混合体。
```
Maybe.create(new MaybeOnSubscribe<String>() {
    @Override
    public void subscribe(MaybeEmitter<String> e) throws Exception {
        e.onSuccess("test");//发送一个数据的情况，或者onError，不需要再调用onComplete(调用了也不会触发onComplete回调方法)
        //e.onComplete();//不需要发送数据的情况，或者onError
    }
}).subscribe(new MaybeObserver<String>() {//订阅观察者
    @Override
    public void onSubscribe(Disposable d) {

    }

    @Override
    public void onSuccess(String s) {
        //发送一个数据时，相当于onNext和onComplete，但不会触发另一个方法onComplete
        Log.i("tag", s);
    }

    @Override
    public void onComplete() {
        //无数据发送时候的onComplete事件
        Log.i("tag", "onComplete");
    }

    @Override
    public void onError(Throwable e) {

    }
});
```
###### 2、操作符
###### 1)创建操作符
&emsp;&emsp;上面的操作我们都用到了基本操作符create，Rxjava为我们提供了许多可以简化操作的操作符。
>just：Completable不能使用该操作服（因为没有onNext事件）；对于Flowable和Observable最多能接收10个参数，也就是发送10个数据；而Single和Maybe只能接收1个参数（只能发送一次onNext事件）。
fromArray：fromArray可以直接传入一个数组，例如fromArray(new int[]{1, 2, 3})，但是不要直接传递一个list进去，这样它会直接把list当做一个数据元素发送。
fromIterable：可以遍历可迭代数据集合。
empty：不会发送任何数据，而是直接发送onComplete事件。
error：直接发送onError事件给观察者。
never：什么都不会发送的操作符，也不会触发观察者任何的回调。
timer：可以指定一段时间发送数据（固定值0L） 。
intaerval：不断地发送数据，可以采用interval操作符。
intervalRange：指定发送范围。
range / rangeLong：
defer：

###### 2)过滤操作符
> filter：设定任意的规则来过滤数据
take：如果需要使用类似request(long)的方法来限制发射数据的数量。
takeLast：
firstElement / lastElement：
first / last：
firstOrError / lastOrError：
elementAt / elementAtOrError：
ofType：
skip / skipLast：
ignoreElements：
distinct / distinctUntilChanged：
timeout：
throttleFirst：
throttleLast / sample：
throttleWithTimeout / debounce：

###### 3)合并、聚合操作符
>startWith / startWithArray：如果你需要在被观察发送元素之前追加数据或者追加新的被观察者，这时候可以使用startWith操作符
concat / concatArray：
merge / mergeArray：
concatDelayError / mergeDelayError：
zip：
combineLatest：
combineLatestDelayError：
reduce：
count：
collect：

###### 4)条件操作符
>all： 要判断所有元素是否满足某个条件，可以使用all操作符，它接受一个Predicate，其中的test方法用于判断某个元素是否满足条件，最终输出是否所有元素都满足条件。
ambArray：
contains：
any：
isEmpty、defaultIfEmpty、switchIfEmpty
sequenceEqual：
takeUntil、takeWhile：
skipUntil、skipWhile：

###### 5)变换操作符
>Map：可以把每一个元素转换成新的元素发射，接收一个Function<T,R>作为转换逻辑的操作。
flatMap：把每一个元素转换成新的被观察者，每个被观察者发射的元素将会合并成新的被观察者，这些元素顺序输出。
lift：针对事件项和事件序列的。注意：RxJava 都不建议开发者自定义 Operator 来直接使用 lift()，而是建议尽量使用已有的 lift() 包装方法（如 map() flatMap() 等）进行组合来实现需求。
compose：对 Observable 整体的变换，需要传入一个Transformer。

###### 6)其它较为常用的操作符
>doOnSubscribe：在onNext()之前调用的，而且可以指定线程。


##### 3.Scheduler
```
//指定被观察者所在线程，我们一般指定在工作线程
.subscribeOn(Schedulers.io())
//指定观察者所在线程，我们一般指定在UI线程
.observeOn(AndroidSchedulers.mainThread())
```

### 三、源码分析
###### 1.一些基本的继承关系
```
public abstract class Observable<T> implements ObservableSource<T>

public interface ObservableSource<T> {
    void subscribe(Observer<? super T> observer);
}

Observalbe关注两个子类ObservalbeCreate与AbstractObservableWithUpstream（其子类又🈶️例如ObservableMap,ObservableFlatMap这些等）
```
```
public interface ObservableOnSubscribe<T> {
    void subscribe(ObservableEmitter<T> e) throws Exception;
}

//这个是在ObservableCreate类中的方法
static final class CreateEmitter<T>
    extends AtomicReference<Disposable>
    implements ObservableEmitter<T>, Disposable {
}
public interface ObservableEmitter<T> extends Emitter<T> {
    void setDisposable(Disposable d);
    void setCancellable(Cancellable c);
    boolean isDisposed();
    ObservableEmitter<T> serialize();
}
public interface Emitter<T> {
    void onNext(@NonNull T value);
    void onError(@NonNull Throwable error);
    void onComplete();
}
```
```
public interface Observer<T> {
    void onSubscribe(Disposable d);
    void onNext(T t);
    void onError(Throwable e);
    void onComplete();
}

public interface Consumer<T> {
    void accept(@NonNull T t) throws Exception;
}
```
###### 2.不涉及操作符和线程切换的调用流程
&emsp;&emsp;create()方法内部实现：
```
@CheckReturnValue
    @SchedulerSupport(SchedulerSupport.NONE)
    public static <T> Observable<T> create(ObservableOnSubscribe<T> source) {
        ObjectHelper.requireNonNull(source, "source is null");
        return RxJavaPlugins.onAssembly(new ObservableCreate<T>(source));
    }

//就是给什么，返回什么
@SuppressWarnings({ "rawtypes", "unchecked" })
    @NonNull
    public static <T> Observable<T> onAssembly(@NonNull Observable<T> source) {
        Function<? super Observable, ? extends Observable> f = onObservableAssembly;
        if (f != null) {
            return apply(f, source);
        }
        return source;
    }

//ObservableCreate可以算是一种适配器的体现，create()需要返回的是Observable,而我现在有的是（方法传入的是）ObservableOnSubscribe对象，ObservableCreate将ObservableOnSubscribe适配成Observable。 
public final class ObservableCreate<T> extends Observable<T> {
    final ObservableOnSubscribe<T> source;

    public ObservableCreate(ObservableOnSubscribe<T> source) {
        this.source = source;
    }
}
```
&emsp;&emsp;subscribe()内部实现
```
@SchedulerSupport(SchedulerSupport.NONE)
    @Override
    public final void subscribe(Observer<? super T> observer) {
        ObjectHelper.requireNonNull(observer, "observer is null");
        try {
            observer = RxJavaPlugins.onSubscribe(this, observer);

            ObjectHelper.requireNonNull(observer, "Plugin returned null Observer");

            subscribeActual(observer);
        } catch (NullPointerException e) { // NOPMD
            throw e;
        } catch (Throwable e) {
            Exceptions.throwIfFatal(e);
            // can't call onError because no way to know if a Disposable has been set or not
            // can't call onSubscribe because the call might have set a Subscription already
            RxJavaPlugins.onError(e);

            NullPointerException npe = new NullPointerException("Actually not, but can't throw other exceptions due to RS");
            npe.initCause(e);
            throw npe;
        }
    }

//Observable中实现真正订阅任务的方法，具体在Observable的子类中实现，这里我们用到的子类是ObservableCreate
protected abstract void subscribeActual(Observer<? super T> observer);
//可以看到里面创建了一个CreateEmitter，即是我们之前提到过的Emitter的子类。 observer.onSubscribe(parent);source.subscribe(parent);这两句将整个数据流链路串接起来了。
@Override
protected void subscribeActual(Observer<? super T> observer) {
        CreateEmitter<T> parent = new CreateEmitter<T>(observer);
        observer.onSubscribe(parent);

        try {
            source.subscribe(parent);
        } catch (Throwable ex) {
            Exceptions.throwIfFatal(ex);
            parent.onError(ex);
        }
}
```
&emsp;&emsp;CreateEmitter的内部实现
```
//这里可以看到我们在e.onNext() 的时候，实际是调用了observer.onNext。
static final class CreateEmitter<T>
    extends AtomicReference<Disposable>
    implements ObservableEmitter<T>, Disposable {


        private static final long serialVersionUID = -3434801548987643227L;

        final Observer<? super T> observer;

        CreateEmitter(Observer<? super T> observer) {
            this.observer = observer;
        }

        @Override
        public void onNext(T t) {
            if (t == null) {
                onError(new NullPointerException("onNext called with null. Null values are generally not allowed in 2.x operators and sources."));
                return;
            }
            if (!isDisposed()) {
                observer.onNext(t);
            }
        }

        @Override
        public void onError(Throwable t) {
            if (t == null) {
                t = new NullPointerException("onError called with null. Null values are generally not allowed in 2.x operators and sources.");
            }
            if (!isDisposed()) {
                try {
                    observer.onError(t);
                } finally {
                    dispose();
                }
            } else {
                RxJavaPlugins.onError(t);
            }
        }

        @Override
        public void onComplete() {
            if (!isDisposed()) {
                try {
                    observer.onComplete();
                } finally {
                    dispose();
                }
            }
        }

        @Override
        public void setDisposable(Disposable d) {
            DisposableHelper.set(this, d);
        }

        @Override
        public void setCancellable(Cancellable c) {
            setDisposable(new CancellableDisposable(c));
        }

        @Override
        public ObservableEmitter<T> serialize() {
            return new SerializedEmitter<T>(this);
        }

        @Override
        public void dispose() {
            DisposableHelper.dispose(this);
        }

        @Override
        public boolean isDisposed() {
            return DisposableHelper.isDisposed(get());
        }
    }
```
###### 3.与操作和线程切换有关的源码分析
###### 1)Map操作：基本流程之前分析的一致，只不过ObservableCreate对象换成了ObservableMap；之后subScribe过程实际调用的是ObservableMap中的subscribeActual。
```
@CheckReturnValue
@SchedulerSupport(SchedulerSupport.NONE)
    public final <R> Observable<R> map(Function<? super T, ? extends R> mapper) {
        ObjectHelper.requireNonNull(mapper, "mapper is null");
        return RxJavaPlugins.onAssembly(new ObservableMap<T, R>(this, mapper));
    }

public final class ObservableMap<T, U> extends AbstractObservableWithUpstream<T, U> {
    final Function<? super T, ? extends U> function;

    public ObservableMap(ObservableSource<T> source, Function<? super T, ? extends U> function) {
        super(source);
        this.function = function;
    }

    @Override
    public void subscribeActual(Observer<? super U> t) {
        source.subscribe(new MapObserver<T, U>(t, function));
    }

    static final class MapObserver<T, U> extends BasicFuseableObserver<T, U> {
        final Function<? super T, ? extends U> mapper;

        MapObserver(Observer<? super U> actual, Function<? super T, ? extends U> mapper) {
            super(actual);
            this.mapper = mapper;
        }

        @Override
        public void onNext(T t) {
            if (done) {
                return;
            }

            if (sourceMode != NONE) {
                actual.onNext(null);
                return;
            }

            U v;

            try {
                v = ObjectHelper.requireNonNull(mapper.apply(t), "The mapper function returned a null value.");
            } catch (Throwable ex) {
                fail(ex);
                return;
            }
            actual.onNext(v);
        }

        @Override
        public int requestFusion(int mode) {
            return transitiveBoundaryFusion(mode);
        }

        @Nullable
        @Override
        public U poll() throws Exception {
            T t = qs.poll();
            return t != null ? ObjectHelper.<U>requireNonNull(mapper.apply(t), "The mapper function returned a null value.") : null;
        }
    }
}
```

### 四、应用场景
###### 1）与Retrofit结合
修改ApiService
```
@POST("translate?doctype=json&jsonversion=&type=&keyfrom=&model=&mid=&imei=&vendor=&screen=&ssid=&network=&abtest=")
@FormUrlEncoded
Observable<Reception2> postCall(@Field("i") String targetSentence);
```
添加Rxjava
```
.addCallAdapterFactory(RxJava2CallAdapterFactory.create())
```
发起请求
```
httpService.postCall("i love you")
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Observer<Reception2>() {
                    @Override
                    public void onSubscribe(Disposable d) {

                    }

                    @Override
                    public void onNext(Reception2 reception2) {
                        Log.e("url", "reception:" + reception2.toString());
                    }

                    @Override
                    public void onError(Throwable e) {

                    }

                    @Override
                    public void onComplete() {

                    }
                });
```

###### 2) 其它
&emsp;&emsp;关于Rxbus,RxBinding等其它应用抽取另外篇幅进行分析。阅读链接[]()


###### 相关参考
[Rxjava在github托管的源码](https://github.com/ReactiveX/RxJava)
[Rxjava 2.x使用详解（一）](https://maxwell-nc.github.io/android/rxjava2-1.html)（5种被观察者与5种操作符的盘点）
[给 Android 开发者的 RxJava 详解](http://gank.io/post/560e15be2dca930e00da1083)
[避免打断链式结构：使用.compose( )操作符](https://www.jianshu.com/p/e9e03194199e)
[RxJava2 源码解析（一）](https://blog.csdn.net/zxt0601/article/details/61614799)
[RxJava2 源码解析（二）](https://blog.csdn.net/zxt0601/article/details/61637439)
