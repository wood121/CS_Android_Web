### 一、Context概述
>&emsp;&emsp;Context是一个抽象类，抽象了一个App应用所有功能的集合，里面定义了各种抽象方法，包括获取系统资源、获取系统服务、发送广播、启动Activity,Service等。

![Context类中定义的相关方法](https://upload-images.jianshu.io/upload_images/4330197-e72200df35a64dc1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
### 二、Context相关继承关系
![Context的继承关系图](https://upload-images.jianshu.io/upload_images/4330197-5eb4399a6ac8f287.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
&emsp;&emsp;Context类是一个抽象类。ContextImpl类是它的具体实现，但是我们并不能直接访问ContextImpl；ContextWrapper是Context的一个包装类，其里面所有的方法实现都是调用其内部mBase（mBase就是ContextImpl对象）变量的方法。ContextWrapper还有一个ContextThemeWrapper子类，该类中扩展了主题相关的方法。
&emsp;&emsp;由继承关系图可以看出，Application和Service是继承自ContextWrapper，而Activity是继承自ContextThemeWrapper（因为Activity在启动的时候系统都会加载一个主题，Service与Application没有界面所以直接继承ContextWrapper）。
&emsp;&emsp;虽然Activity，Application，Service都有一个共同的祖先Context，但是他们自己本身持有的Context对象是不同的。在一个应用中&emsp;**总Context实例个数 = Service个数 + Activity个数 + 1（Application对应的Context实例）**
### 三、Application相关源码
&emsp;&emsp;我们从应用的主入口开始看
```
public static void main(String[] args) {
        ...

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
这里主要做了两件事，以下几句构建了主线程的消息循环。
```
Looper.prepareMainLooper();
if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }
Looper.loop();
```
下面这两句，绑定应用进程到ActivityManagerService；
```
ActivityThread thread = new ActivityThread();
thread.attach(false);
```
&emsp;&emsp;下面我们再来看看attach方法中做了什么：
```
private void attach(boolean system) {
        sCurrentActivityThread = this;
        mSystemThread = system;
        if (!system) {
           ...省略
            RuntimeInit.setApplicationObject(mAppThread.asBinder());
            final IActivityManager mgr = ActivityManagerNative.getDefault();
            try {
                mgr.attachApplication(mAppThread);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
           ...省略
        } else {
            //因为ActivityThread中调用的时候attach(false)，所以这部分可以省略不看。
           ...省略
        }
       ...省略
 }
```
可以看到ActivityManagerNative通过getDefault()方法返回ActivityManagerService实例，ActivityManagerService通过attachApplication将ApplicationThread对象绑定到ActivityManagerService，而ApplicationThread作为Binder实现ActivityManagerService对应用进程的通信和控制。

### 四、Activity相关源码
#####1.Activity的启动过程
&emsp;&emsp;我们从Activity的startActivity方法开始，它有几种重载方式，但最终都会调用startActivityForResult方法。其中调用了Instrumentation的execStartActivity方法。里面再有一句''ActivityManagerNative.getDefault().startActivity''，（AMS继承自ActivityManagerNative，后者继承自Binder且实现了IActivityManager这个Binder接口，所以AMS是IActivityManager的具体实现。由于ActivityManagerNative.getDefault()是一个IActivityManager类型的Binder对象，所以它就是AMS。
&emsp;&emsp;AMS中的startActivity又调用ActivityStackSuperviosr的startActivityMayWait方法，接下来的一段互相调用流程见下图
![传递顺序.jpeg](https://upload-images.jianshu.io/upload_images/4330197-16da7bf1c9125136.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
&emsp;&emsp;最后在ActivityStackSupervisor的realStartActivityLocked方法中调用app.thread.scheduleLaunchActivity()方法，其中app.thread的类型为IApplicationThread，其实现是ActivityThread的内部类ApplicationThread，即是调用了ApplicationThread的scheduleLaunchActivity()，这里发送一个消息给H处理（即是ActivityThread中的内部Handler类）。
&emsp;&emsp;最终消息传给了Handler的LAUCH_ACTIVITY中，HandleLaunchActivity()方法对消息进行处理。其中又调用了performLaunchActivity方最终完成了Activity 对象的创建和启动过程。

### 五、Service相关源码
...

##### 相关参考：
《Android开发艺术探索》
[android context 详解](https://www.jianshu.com/p/f670d5725951)
[Android中Context源码分析（一）](https://blog.csdn.net/dongxianfei/article/details/52303847)
[Android中ContextImpl源码分析（二）](https://blog.csdn.net/dongxianfei/article/details/54632423)
[Android线程管理（二）-ActivityThread](https://www.cnblogs.com/younghao/p/5126408.html)
[Android应用程序的Activity启动过程简要介绍和学习计划](https://blog.csdn.net/luoshengyang/article/details/6685853)
