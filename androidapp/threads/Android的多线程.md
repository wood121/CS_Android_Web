### 一、Android的主线程与子线程
&emsp;&emsp;主线程（即UI线程）被阻塞（大概5秒钟）后会导致ANR(Application Not Responding)错误；子线程不能访问UI（因为UI控件不是线程安全的）。
&emsp;&emsp;为实现子线程执行耗时任务后刷新UI，Android为我们提供的有Handler与AsynTask。其中Handler主要接收子线程发送的数据，然后用此数据配合主线程更新UI（阅读连接：[Android消息机制](https://www.jianshu.com/p/cc977f452755)）；关于AsyncTask，见后面的分析。
### 二、AsynTask
###### 1.能干啥、有啥区别
&emsp;&emsp;实现的效果：在其他线程中执行一个耗时操作，并随时报告执行进度给UI线程，执行完成后将结果报告给UI线程”。

```
 * <p>AsyncTask enables proper and easy use of the UI thread. This class allows you
 * to perform background operations and publish results on the UI thread without
 * having to manipulate threads and/or handlers.</p>

 * <p>AsyncTask is designed to be a helper class around {@link Thread} and {@link Handler}
 * and does not constitute a generic threading framework. AsyncTasks should ideally be
 * used for short operations (a few seconds at the most.) If you need to keep threads
 * running for long periods of time, it is highly recommended you use the various APIs
 * provided by the <code>java.util.concurrent</code> package such as {@link Executor},
 * {@link ThreadPoolExecutor} and {@link FutureTask}.</p>
```
结合文档我们了解到AsynTask实际上是一个辅助类，其中封装了Thread和Handler；它只能执行短时间的异步任务（最多几秒钟），对于特别耗时的任务，文档中推荐了Executor、ThreadPoolExecutor和FutureTask。

###### 2.使用方法
&emsp;&emsp;涉及的参数与方法
```
//Params表示参数的类型、Progress表示后台任务执行进度的类型、Result表示后台任务返回结果的类型。不需要的参数可以传入Void
public abstract class AsyncTask<Params, Progress, Result>

void onPreExecute()：在主线程中执行。
Result doInBackground(Params... params)：在线程池中执行。parms表示异步任务的输入参数，在该方法中可以调用publishProgress来更新进度（publishProgress会调用onProgressUpdate方法），其执行结果会返回给onPostExecute()。
void onProgressUpdate(Progress... values)：在主线程中执行。
void onPostExecute(Result result)：在主线程中执行。
```
&emsp;&emsp;一个应用demo
```
public class AsynTaskDemoActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_asyn_task_demo);

//        new DownLoadFilesTask().execute(...);
    }

    private class DownLoadFilesTask extends AsyncTask<URL, Integer, Long> {

        @Override
        protected Long doInBackground(URL... params) {

            int count = params.length;
            int totalSize = 0;

            for (int i = 0; i < count; i++) {

//              int totalSize =  DownLoader.downloadFile(params[i]);
                publishProgress(i / count);
                if (isCancelled()) {
                    break;
                }
            }

            return null;
        }

        @Override
        protected void onProgressUpdate(Integer... values) {
//            setProgressPercent(values[0]);
        }

        @Override
        protected void onPostExecute(Long aLong) {
            ToastUtil.showToast(AsynTaskDemoActivity.this, "result:" + aLong);
        }
    }

}
```
&emsp;&emsp;注意事项
>Task的实例必须在UI thread中创建；execute方法必须在UI thread中调用；
不要手动的调用onPreExecute(), onPostExecute(Result)，doInBackground(Params...), onProgressUpdate(Progress...)这几个方法；
该task只能被执行一次，否则多次调用时将会出现异常。

###### 3.工作原理

### 三、IntentService
