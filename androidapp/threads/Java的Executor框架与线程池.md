### 一、Executor框架
###### 1.概述
&emsp;&emsp;在JavaSE5之后，直接使用Thread类并不是被提倡的并发开发方式，而Executor成为主要角色。有以下几方面的原因：
>&emsp;&emsp;复用效率。Thread每一次构建和销毁都有一定成本，而线程池执行器中，线程是可以复用的，这在一定程度上降低了线程的维护成本，提高了开发和运行效率。
&emsp;&emsp;资源限制和管理。对于线程的启用需要有一个限制和管理，而线程池执行器将这些方面融入，降低了并发多线程的管理成本。
###### 2.继承关系
![Executor继承关系图](https://upload-images.jianshu.io/upload_images/4330197-a36c7ecdc07adc9f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
>&emsp;&emsp;Executor 接口中之定义了一个方法 execute（Runnable command），该方法接收一个 Runable 实例，它用来执行一个任务，任务即一个实现了 Runnable 接口的类。
&emsp;&emsp;ExecutorService 接口继承自 Executor 接口，它提供了更丰富的实现多线程的方法，比如，ExecutorService 提供了关闭自己的方法，以及可为跟踪一个或多个异步任务执行状况而生成 Future 的方法。
&emsp;&emsp;ThreadPoolExecutor是线程池的真正实现，它的构造方法提供了一系列参数来配置线程池。
&emsp;&emsp;Executors 提供了一系列工厂方法用于创先线程池，返回的线程池都实现了 ExecutorService 接口。

### 二、ThreadPoolExecutor
###### 1.构造方法
```
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) 
```

>corePoolSize：线程池的核心线程数，默认情况下，核心线程会在线程池中一直存活，即使它们处于闲置状态。
maximumPoolSize：线程池所能容纳的最大线程数。
keepAliveTime：非核心线程闲置时的超时时间，超过就会被回收。
unit：指定keepAliveTime参数的时间单位。
workQueue：线程池中的任务队列。
threadFactory：线程工厂，为线程池提供创建新线程的功能。
handler：线程池对拒绝任务的处理策略。AbortPolicy 直接丢弃新任务；CallerRunsPolicy 用调用者的线程执行新的任务；DiscardPolicy 静默丢弃任务不通知调用者；DiscardOldestPolicy 丢弃最旧的任务。

###### 2.任务执行规则
>1）如果线程池中的线程数量未达到核心线程的数量，那么会直接启动一个核心线程来执行任务。
2）如果线程池中的线程数量已经达到或者超过核心线程的数量，那么任务会被插入到任务队列中排队等待执行。
3）如果在步骤2中无法将任务插入到任务队列中，这往往是由于任务队列已满，这个时候如果线程数量未达到线程池规定的最大值，那么会立刻启动一个非核心线程来执行任务。
4）如果步骤3中线程数量已经达到线程池的最大值，那么就拒绝执行此任务，ThreadPoolExecutor会调用RejectedExecutionHandler的rejectedExecution方法来通知调用者。
###### eg1：AsynTask中的线程池配置
```
    //获取当前的cpu核心数
    private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();

    //我们想在核心池中至少有2个线程，最多4个线程，更喜欢有1个小于CPU计数的CPU，以避免CPU背景饱和
    //线程池核心容量
    private static final int CORE_POOL_SIZE = Math.max(2, Math.min(CPU_COUNT - 1, 4));
    //线程池最大容量
    private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
    //过剩的空闲线程的存活时间
    private static final int KEEP_ALIVE_SECONDS = 30;
    //ThreadFactory 线程工厂，通过工厂方法newThread来获取新线程
    private static final ThreadFactory sThreadFactory = new ThreadFactory() {
        //原子整数，可以在超高并发下正常工作
        private final AtomicInteger mCount = new AtomicInteger(1);

        public Thread newThread(Runnable r) {
            return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
        }
    };
    //静态阻塞式队列，用来存放待执行的任务，初始容量：128个
    private static final BlockingQueue<Runnable> sPoolWorkQueue =
            new LinkedBlockingQueue<Runnable>(128);

    /**
     * 可以用来并行执行任务的一个Executor
     * 静态并发线程池，可以用来并行执行任务，尽管从3.0开始，AsyncTask默认是串行执行任务
     * 但是我们仍然能构造出并行的AsyncTask
     */
    public static final Executor THREAD_POOL_EXECUTOR;

    static {
        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
                CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
                sPoolWorkQueue, sThreadFactory);
        threadPoolExecutor.allowCoreThreadTimeOut(true);
        THREAD_POOL_EXECUTOR = threadPoolExecutor;
    }
```
### 三、Executors
&emsp;&emsp;Executors 提供了一系列工厂方法用于创先线程池，返回的线程池都实现了 ExecutorService 接口。
```
public static ExecutorService newFixedThreadPool(int nThreads)
创建固定数目线程的线程池。

public static ExecutorService newCachedThreadPool()
创建一个可缓存的线程池，调用execute将重用以前构造的线程（如果线程可用）。如果现有线程没有可用的，则创建一个新线 程并添加到池中。终止并从缓存中移除那些已有 60 秒钟未被使用的线程。

public static ExecutorService newSingleThreadExecutor()
创建一个单线程化的Executor。

public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize)
创建一个支持定时及周期性的任务执行的线程池，多数情况下可用来替代Timer类。
```
### 四、线程池的自定义与封装

```
public class ThreadPoolManager {

    /**
     * *********************静态内部类的单例模式*********************
     */
    private ThreadPoolManager mThreadPoolManager;

    public static ThreadPoolManager getInstance() {
        return SingletoHolder.sInstance;
    }

    private static class SingletoHolder {
        private static final ThreadPoolManager sInstance = new ThreadPoolManager();
    }

    /**
     * *********************线程池的创建、执行任务、shutDown操作*********************
     */
    private static ThreadPoolExecutor mExecutor = null;
    //根据cpu的数量动态的配置核心线程数和最大线程数
    private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();
    //核心线程数 = CPU核心数 + 1
    private static final int CORE_POOL_SIZE = CPU_COUNT + 1;
    //线程池最大线程数 = CPU核心数 * 2 + 1
    private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
    //非核心线程闲置时超时1s
    private static final int KEEP_ALIVE = 1;

    /**
     * 初始化ThreadPoolExecutor
     * 双重检查加锁,只有在第一次实例化的时候才启用同步机制,提高了性能
     */
    private void initThreadPoolExecutor() {
        if (mExecutor == null || mExecutor.isShutdown() || mExecutor.isTerminated()) {
            synchronized (ThreadPoolManager.class) {
                if (mExecutor == null || mExecutor.isShutdown() || mExecutor.isTerminated()) {
                    TimeUnit unit = TimeUnit.MILLISECONDS;
                    BlockingQueue<Runnable> workQueue = new LinkedBlockingDeque<>();
                    ThreadFactory threadFactory = Executors.defaultThreadFactory();
                    RejectedExecutionHandler handler = new ThreadPoolExecutor.DiscardPolicy();

                    mExecutor = new ThreadPoolExecutor(CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE, unit, workQueue,
                            threadFactory, handler);
                }
            }
        }
    }

    /**
     执行任务和提交任务的区别?
     1.有无返回值
     execute->没有返回值
     submit-->有返回值
     2.Future的具体作用?
     1.有方法可以接收一个任务执行完成之后的结果,其实就是get方法,get方法是一个阻塞方法
     2.get方法的签名抛出了异常===>可以处理任务执行过程中可能遇到的异常
     */
    /**
     * 执行任务
     */
    public void execute(Runnable task) {
        initThreadPoolExecutor();
        mExecutor.execute(task);
    }

    /**
     * 提交任务
     */
    public Future<?> submit(Runnable task) {
        initThreadPoolExecutor();
        return mExecutor.submit(task);
    }

    /**
     * 移除任务
     */
    public void remove(Runnable task) {
        initThreadPoolExecutor();
        mExecutor.remove(task);
    }

    /**
     * 立马关闭线程池
     */
    public static void shutDown() {
        if (mExecutor != null) {
            Log.e("wood121", "shutDown::::::" + mExecutor);
            mExecutor.shutdown();
            Log.e("wood121", "shutDown::::::" + mExecutor);
        }
    }

    public static void shutDownNow() {
        if (mExecutor != null) {
            Log.e("wood121", "shutDownNow::::::" + mExecutor);
            mExecutor.shutdownNow();
            Log.e("wood121", "shutDownNow::::::" + mExecutor);
        }
    }
}
```


###### 相关参考
《Android开发艺术探索》
[并发新特性—Executor 框架与线程池](http://wiki.jikexueyuan.com/project/java-concurrency/executor.html)
[Android线程池（三）常用封装](https://blog.csdn.net/iromkoear/article/details/65714291)
