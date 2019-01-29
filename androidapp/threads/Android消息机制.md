&emsp;&emsp;本文是结合《Android艺术探索》中第10章与Android源码阅读理解的一些总结。关于IDE关联源码，可以查看此篇[Mac下Android Studio关联源码](https://www.jianshu.com/p/28972fc145a8)。
### 一、Android消息机制指的是什么、为什么会有Handler？
>&emsp;&emsp;Android的消息机制主要是指Handler的运行机制以及Handler所附带的MessageQueue和Looper的工作过程。
&emsp;&emsp;为什么会有Handler？这是因为Android规定访问UI只能在主线程中进行，如果在子线程中访问UI，程序将会抛出异常；同时Android又建议不要在主线程中进行耗时操作，否则会导致程序无法响应即ANR。但是，我们在开发过程中会遇到一种情况，例如在子线程进行网络请求，获取数据后刷新UI，但是子线程中无法访问UI，因此系统提供了Handler。
&emsp;&emsp;为什么Android会不允许在线程中访问ui？这是因为Android的UI控件不是线程安全的。如果为了线程安全对UI控件的访问加上锁机制一方面会让UI访问变得复杂，另一方面会降低UI访问的效率。

### 二、Handler在主线程中接收消息及其实现原理
##### 1.使用方法
&emsp;&emsp;主线程中创建Handler对象
```
Handler handler=new Handler(){
    @Override
    public void handleMessage(Message msg) {
        super.handleMessage(msg);
       //执行逻辑
}};
```
&emsp;&emsp;子线程中使用handler对象发送消息
```
Message message=new Message();
message.what=1;
message.obj="haha";
handler.sendMessage(message);
```
##### 2.创建Handler对象内部源码
&emsp;&emsp;翻开源码，我们可以看到Handler内部有7个构造方法，最终都是调用了public Handler(Callback callback, boolean async) {...}或者public Handler(Looper looper, Callback callback, boolean async) {...}。
```
public Handler(Callback callback, boolean async) {
        ...
        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        ...
    }
```
可以看到先检查了该线程内是否存在Looper，如果没有，就会抛出异常，所以我们在子线程中这种方式使用Handler之前需要先调用Looper.prepare()准备好该线程的Looper对象。其次拿到了Looper对象中mQueue（即MessageQueue对象），在Looper的构造方法中我们会看到MessageQueue对象的创建，在后面我们会提到。

##### 3.发送消息内部源码
&emsp;&emsp;我们可以通过handler的post方法投递一个Runnable或者直接使用sendMessage方法发送一个消息。post其实也是通过sendMessage方法实现。
&emsp;&emsp;我们点开sendMessage的源码，会发现最终都会走到enqueueMessage方法，其内部实现如下：
```
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```
记住msg.target = this，这句话将handler对象赋予了msg的target，我们在Looper.loop()方法会看到有一句“msg.target.dispatchMessage(msg);”，其中的target就是此时传递过去的。
&emsp;&emsp;我们再来看看queue.enqueueMessage(msg, uptimeMillis)方法中的实现，
```
boolean enqueueMessage(Message msg, long when) {
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        synchronized (this) {
            if (mQuitting) {
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w(TAG, e.getMessage(), e);
                msg.recycle();
                return false;
            }

            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // Inserted within the middle of the queue.  Usually we don't have to wake
                // up the event queue unless there is a barrier at the head of the queue
                // and the message is the earliest asynchronous message in the queue.
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }
```
其内部的主要操作其实就是单链表的插入操作。
##### 4.主线程的消息循环
&emsp;&emsp;前面我们已经看到了Handler对象的创建，消息的发送过程，接下来我们了解下消息是如何传递到handler的handleMessage方法中去的。这里我们需要了解主线程的消息循环。
&emsp;&emsp;首先我们找到Android的主线程ActivityThread，主线程入口main方法。其中调用Looper.prepareMainLooper()准备好了主线程的looper对象，后面Looper.loop()开启了主线程的轮巡。相关阅读链接，[Android中为什么主线程不会因为Looper.loop()里的死循环卡死？](https://www.zhihu.com/question/34652589) 
```
public static void main(String[] args) {
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");
        SamplingProfilerIntegration.start();

        // CloseGuard defaults to true and can be quite spammy.  We
        // disable it here, but selectively enable it later (via
        // StrictMode) on debug builds, but using DropBox, not logs.
        CloseGuard.setEnabled(false);

        Environment.initForCurrentUser();

        // Set the reporter for event logging in libcore
        EventLogger.setReporter(new EventLoggingReporter());

        // Make sure TrustedCertificateStore looks in the right place for CA certificates
        final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
        TrustedCertificateStore.setDefaultUserDirectory(configDir);

        Process.setArgV0("<pre-initialized>");

        Looper.prepareMainLooper();

        ActivityThread thread = new ActivityThread();
        thread.attach(false);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        if (false) {
            Looper.myLooper().setMessageLogging(new
                    LogPrinter(Log.DEBUG, "ActivityThread"));
        }

        // End of event ActivityThreadMain.
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        Looper.loop();

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```
&emsp;&emsp;我们看看Looper.prepareMainLooper()的内部实现，调用prepare(false)创建Looper对象并存入sThreadLocal，其后使用锁机制保证对象的唯一并通过myLooper()将prepare(false)中存入的Looper对象取出赋予sMainLooper。另外我们可以看到Looper的构造方法中创建了一个MessageQueue对象。
```
public static void prepareMainLooper() {
        prepare(false);
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper();
        }
    }
private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }
public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }
```
&emsp;&emsp;我们看看Looper.loop()的内部实现，其内部开启了一个死循环，queue.next()不断地去取消息队列中的消息（与queue.enqueueMessage相对将消息取出并从单链表中移除），如果有消息传递过来则调用msg.target.dispatchMessage(msg)，这里即是调用了handler.dispatchMessage(msg)，dispatchMessage中会将消息传递给handleMessage(msg)，从而完成了消息的传递。
```
public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            // This must be in a local variable, in case a UI event sets the logger
            final Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

            final long traceTag = me.mTraceTag;
            if (traceTag != 0 && Trace.isTagEnabled(traceTag)) {
                Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
            }
            try {
                msg.target.dispatchMessage(msg);
            } finally {
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }

            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }

            // Make sure that during the course of dispatching the
            // identity of the thread wasn't corrupted.
            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }

            msg.recycleUnchecked();
        }
    }
```
##### 5.Handler消息机制流程图
&emsp;&emsp;我们来梳理下整个流程：首先创建Handler对象，而后将消息发送至消息队列；于此同时主线程的Looper.loop()会通过queue.next()不断去访问消息队列，一旦消息队列中有消息传递过来就会将消息取出来传递给handler的handleMessage()方法，从而完成整个流程。
![Handler消息机制流程图](https://upload-images.jianshu.io/upload_images/4330197-c6f2f4defe8f730c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 三、Hanlder在子线程中接收消息
##### 1.应用场景
&emsp;&emsp;通常我们都是在 Activity 中，让子线程执行耗时任务，执行完之后给主线程发送消息让主线程更新UI。其实还有很多应用场景需要让主线程给子线程发送消息，该消息作为任务的载体，比如在IntentService 中，主线程就给子线程发送了消息，让子线程干活。 
##### 2.应用Demo
&emsp;&emsp;1）一般情况
```
public class MainActivity extends AppCompatActivity {
    private Handler subHandler;//是在子线程中创建的Handler对象
    private Looper myLooper;//子线程中的Looper对象

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
      
        new Thread(new Runnable() {
            @Override
            public void run() {
                /**
                 *  1、创建了Looper对象，然后Looper对象中创建了MessageQueue
                 *  2、并将当前的Looper对象跟当前的线程（子线程）绑定ThreadLocal
                 */
                Looper.prepare();

                /**
                 * 1、创建Handler对象，然后从当前线程中获取Looper对象，然后获取到MessageQueue对象
                 */
                subHandler = new Handler(){
                    @Override
                    public void handleMessage(Message msg) {
                        super.handleMessage(msg);
                        Toast.makeText(MainActivity.this,msg.obj.toString(),Toast.LENGTH_LONG).show();
                    }
                };

                //获取当前线程中的Looper对象
                myLooper = Looper.myLooper();

                /**
                 * 1、从当前线程中找到之前创建的Looper对象，然后找到MessageQueue
                 * 2、开启死循环，遍历消息池中的消息
                 * 3、当获取到msg的时候，调用这个msg的handler的dispatchMsg方法，让msg执行起来
                 */
                Looper.loop();
                Log.e("zxj","loop()方法执行完了");
            }
        }).start();
    }

    public void onSendClick(View v){
        //从消息池中获取一个旧的msg，如果没有，重新创建消息
        Message message = subHandler.obtainMessage(1, "我是主线程发送来是消息");
        message.sendToTarget();
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        if(myLooper != null)
            myLooper.quit();
    }
}
```
&emsp;&emsp;2）使用HanlderThread
```
public class MainActivity extends AppCompatActivity {
    SimpleDateFormat df = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");//设置日期格式
    TextView updateTimeTv;
    Handler mUIHandler=new Handler(){
        @Override
        public void handleMessage(Message msg) {
            //更新ui
            updateTimeTv.setText((String)msg.obj);
            //1秒后再次请求服务器，并更新数据
            timeMessagehandler.sendEmptyMessageAtTime(0,1000);
        }
    };


    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        updateTimeTv = (TextView)findViewById(R.id.tv);
        initThreadHandler();
    }

    HandlerThread handlerThread;
    Handler timeMessagehandler;
    private void initThreadHandler() {
        //构建HandlerThread
        handlerThread=new HandlerThread("updateTimeThread");
        //启动handler
        handlerThread.start();
        //构建子线程的handler对象
        timeMessagehandler=new Handler(handlerThread.getLooper()){
            @Override
            public void handleMessage(Message msg) {
                //请求服务端时间
                try {
                    //模拟耗时请求
                    Thread.sleep(3000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                //网络请求成功，通过主线程更新ui
                Message message=Message.obtain();
                message.obj= df.format(new Date());
                mUIHandler.sendMessage(message);
                System.out.println("handlerTimeMessage"+Thread.currentThread().getName());
            }
        };

        //向子线程发送消息，通知它发送网络请求
        timeMessagehandler.sendEmptyMessage(0);
    }

}   
```
### 四、其它用法
&emsp;&emsp;子线程更新UI
```
// 方法一：子线程使用Handler将数据发送至主线程进行更新，即“第二块”中提到的方式
//方法二：
mainHandler.post(new Runnable() {
    @Override
    public void run() {
        //你的处理逻辑
        titleView.setText("postRunnable——Result");
    }
});
//方法三：在activity中，直接使用runOnUiThread
runOnUiThread(new Runnable() {    //有context的环境
    @Override
    public void run() {
        titleView.setText("runOnUiThread——Result");
    }
});
Activity activity = (Activity) imageView.getContext();    
activity.runOnUiThread(new Runnable() {  //没有context的环境
    @Override
    public void run() {
        imageView.setImageBitmap(bitmap);
    }
});
//方法四：如果有子view或自定义view，处理逻辑后可以直接用view.post()方法.
viewPostBtn.post(new Runnable() {
    @Override
    public void run() {
        titleView.setText("viewPost——Result");
    }
});
```
&emsp;&emsp;Handler中的定时器
```
Runnable runnable=new Runnable() {
    @Override
    public void run() {
        handler.sendEmptyMessage(1);
        handler.postDelayed(runnable, 1000);
    }
};
```
### 五、使用Handler造成的内存泄漏原因及解决方案
##### 1.造成内存泄漏的原因
&emsp;&emsp;第二部分我们展示的Handler使用方式是会造成内存泄漏的，我们来进行分析：当使用内部类（包括匿名类）来创建Handler的时候，Handler对象会隐式地持有一个外部类对象（通常是一个Activity）的引用（不然你怎么可能通过Handler来操作Activity中的View？）。而Handler通常会伴随着一个耗时的后台线程（例如从网络拉取图片）一起出现，这个后台线程在任务执行完毕（例如图片下载完毕）之后，通过消息机制通知Handler，然后Handler把图片更新到界面。然而，如果用户在网络请求过程中关闭了Activity，正常情况下，Activity不再被使用，它就有可能在GC检查时被回收掉，但由于这时线程尚未执行完，而该线程持有Handler的引用（不然它怎么发消息给Handler？），这个Handler又持有Activity的引用，就导致该Activity无法被回收（即内存泄露），直到网络请求结束（例如图片下载完毕）。另外，如果你执行了Handler的postDelayed()方法，该方法会将你的Handler装入一个Message，并把这条Message推到MessageQueue中，那么在你设定的delay到达之前，会有一条MessageQueue -> Message -> Handler -> Activity的链，导致你的Activity被持有引用而无法被回收。
##### 2.解决方案
&emsp;&emsp;将Handler声明为静态类（在Java 中，非静态的内部类和匿名内部类都会隐式地持有其外部类的引用，静态的内部类不会持有外部类的引用。静态类不持有外部类的对象，所以你的Activity可以随意被回收。由于Handler不再持有外部类对象的引用，导致程序不允许你在Handler中操作Activity中的对象了。所以你需要在Handler中增加一个对Activity的弱引用）。
```
    static class MyHandler extends Handler {
        WeakReference<Activity> mWeakReference;
        public MyHandler(Activity activity) 
        {
            mWeakReference=new WeakReference<Activity>(activity);
        }
        @Override
        public void handleMessage(Message msg)
        {
            final Activity activity=mWeakReference.get();
            if(activity!=null)
            {
                if (msg.what == 1)
                {
                    noteBookAdapter.notifyDataSetChanged();
                }
            }
        }
    }
```
##### 相关参考
[Handler常见用法小结](https://blog.csdn.net/tobestrong_csdn/article/details/52441152)
[让主线程给子线程发送消息](https://www.jianshu.com/p/78e7734e9e04)
[handlerThread使用场景分析及源码解析](https://blog.csdn.net/nightcurtis/article/details/77676349)
[Android使用Handler造成内存泄露的分析及解决方法](https://www.cnblogs.com/xujian2014/p/5025650.html)
