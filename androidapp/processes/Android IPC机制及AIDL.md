### 一、IPC概述
##### 1.基本概念
>&emsp;&emsp;IPC是Inter-Process Communication 的缩写，是指两个进程间进行数据交换的过程。
##### 2.应用场景
>&emsp;&emsp;Android多进程分为两种：一种情况是一个应用因为某些原因自身需要采用多进程模式来实现；另一种情况是当前应用需要向其它应用获取数据，由于是两个应用，所以必须采用跨进程的方式来获取所需的数据。
>&emsp;&emsp;Android中使用多进程只有一种方式，那就是给四大组件在AndroidMenifest中指定android:process = ":xxx"属性。注意进程命名如果是":xxx"则该进程属于当前应用的私有进程，其它应用组件不可以和它跑在同一个进程中；若非如此，则该进程属于全局进程，其它应用通过ShareUID方式可以和它跑在同一进程中。
>&emsp;&emsp;使用多进程会造成几个问题：1、静态成员和单例模式完全失效；2、线程同步机制完全失效；3、SharePreferences的可靠性下降；4、Application会多次创建。
##### 3.IPC基础
###### 1）序列化
&emsp;&emsp;Serilizable接口是Java中的序列化接口，使用起来简单但是开销很大，序列化与反序列化过程需要大量I/O操作。主要用在将对象序列化到存储设备中或者将对象序列化后通过网络传输。
&emsp;&emsp;Parcelable接口是Android中的序列化方式，主要用在内存序列化上。
###### 2）Binder
&emsp;&emsp;[Binder的使用、上层原理与源码中的应用方式](https://www.jianshu.com/p/180e2f0e9654)

##### 4.Android中的IPC方式
###### 1）Bundle
&emsp;&emsp;当我们在一个进程中启动了另一个进程的Activity,Service和Receiver，我们可以在Bundle中附加我们需要传输给远程进程的信息并通过Intent发出去。
###### 2）文件共享
&emsp;&emsp;两个进程通过读／写同一个文件来交换数据。
###### 3）Messenger
&emsp;&emsp;[进程间通信 -- Messenger](https://www.jianshu.com/p/cf6778214303)
###### 4）AIDL
&emsp;&emsp;接下来的篇幅将会对AIDL应用进行解读。
###### 5）ContentProvider
###### 6）Socket

### 二、AIDL概述
##### 1.基本概念与应用场景
>&emsp;&emsp;AIDL全称是Android Interface Difine Language(android自定义接口语言)  。
&emsp;&emsp;其主要应用场景就是有一对多且有RPC需求的情况。我理解这是从服务端角度来考虑的；如果从客户端角度来考虑使用服务，主要是写应用程序内部的服务（即本地服务），或者调用其他应用程序提供的服务（即远程服务）。
&emsp;&emsp;客户端调用远程服务的步骤：在该工程中新建一个和远程服务类所在工程一样结果的包、将远程服务工程中的接口文件 (.aidl文件)拷贝到该新建的包下、绑定远程服务。服务端见后面的Demo。
##### 2.支持的数据类型
>基本数据类型、
String和CharSequence、
List（只支持ArrayList且里面每个元素都必须能够被AIDL支持）、
Map（只支持HashMap且里面每个元素都必须能够被AIDL支持）、
Parcelable及其子类对象、
AIDL接口本身也可以在AIDL文件中使用。
##### 3.demo
###### 1)创建AIDL
创建需要操作的实体类
```
public class Book implements Parcelable {

    public int bookId;
    public String bookName;

    public Book(int i, String mBookName) {
        this.bookId = i;
        this.bookName = mBookName;
    }

    public Book(Parcel in) {
        bookId = in.readInt();
        bookName = in.readString();
    }

    public static final Creator<Book> CREATOR = new Creator<Book>() {
        @Override
        public Book createFromParcel(Parcel in) {
            return new Book(in);
        }

        @Override
        public Book[] newArray(int size) {
            return new Book[size];
        }
    };


    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeInt(bookId);
        dest.writeString(bookName);
    }

    @Override
    public String toString() {
        return "Book{" +
                "bookId=" + bookId +
                ", bookName='" + bookName + '\'' +
                '}';
    }
}
```
创建实体类的映射aidl文件及相关接口aidl
```
// IMyAidlInterface.aidl
package com.example.wood121.viewdemos.aidl;

// Declare any non-default types here with import statements
//需要注意：此处使用parcelable去声明我们创建的实体类；
parcelable Book;
```
```
// IOnNewBookArrivedListener.aidl
package com.example.wood121.viewdemos.aidl;

// Declare any non-default types here with import statements

import com.example.wood121.viewdemos.aidl.Book;

interface IOnNewBookArrivedListener {

    void onNewBookArrived(in Book newBook);
}
```
```
// IBookManager.aidl
package com.example.wood121.viewdemos.aidl;

// Declare any non-default types here with import statements
/*需要注意几点：在其它aidl文件中使用映射的实体类需要import导入（虽然在同一个包中）；
AIDL文件中除了基本数据类型，其它类型的参数必须标注方向"in, out或者inout"。*/
import com.example.wood121.viewdemos.aidl.Book;
import com.example.wood121.viewdemos.aidl.IOnNewBookArrivedListener;

interface IBookManager {
    List<Book> getBookList();
    void addBook(in Book book);
    void registerListener(IOnNewBookArrivedListener listener);
    void unregisterListener(IOnNewBookArrivedListener listener);
}
```
在AS上点击将代码同步一下，可得到跨进程 Client 和 Server 的通信媒介，具体途径如图：
![自动生成AIDL对应.java文件](https://upload-images.jianshu.io/upload_images/4330197-50f0708af79250e6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

###### 2）服务端代码
```
public class BookManagerService extends Service {

    private static final String TAG = "BMS";

    private AtomicBoolean mIsServiceDestroyed = new AtomicBoolean(false);

    //需要进行处理的对象
    private CopyOnWriteArrayList<Book> mBookList = new CopyOnWriteArrayList<>();
    //    private CopyOnWriteArrayList<IOnNewBookArrivedListener> mListenerList = new CopyOnWriteArrayList<>();
    //为了能在多进程情况下Activity退出时解除注册，RemoteCallbackList是系统专门提供的用于删除跨进程listener的接口
    private RemoteCallbackList<IOnNewBookArrivedListener> mListenerList = new RemoteCallbackList<>();

    private Binder mBinder = new IBookManager.Stub() {

        /**
         * 书籍的书籍
         * @return
         * @throws RemoteException
         */
        @Override
        public List<Book> getBookList() throws RemoteException {
            //假设服务端的是耗时操作，则需要客户端在请求的时候使用子线程访问
            SystemClock.sleep(3000);
            return mBookList;
        }

        /**
         * 添加书籍
         * @param book
         * @throws RemoteException
         */
        @Override
        public void addBook(Book book) throws RemoteException {
            mBookList.add(book);
        }

        /**
         * 添加监听，新书到了
         * @param listener
         * @throws RemoteException
         */
        @Override
        public void registerListener(IOnNewBookArrivedListener listener) throws RemoteException {
//            if (!mListenerList.contains(listener)) {
//                mListenerList.add(listener);
//            } else {
//                Log.e(TAG, "already exits");
//            }
//            Log.e(TAG, "registerListener,size:" + mListenerList.size());

            mListenerList.register(listener);
        }

        @Override
        public void unregisterListener(IOnNewBookArrivedListener listener) throws RemoteException {
//            if (mListenerList.contains(listener)) {
//                mListenerList.remove(listener);
//                Log.e(TAG, "unregister listener ok");
//            } else {
//                Log.e(TAG, "not found");
//            }
//            Log.e(TAG, "unRegisterListener,size:" + mListenerList.size());

            mListenerList.unregister(listener);
        }
    };

    public BookManagerService() {
    }

    @Override
    public void onCreate() {
        super.onCreate();
        mBookList.add(new Book(1, "Android"));
        mBookList.add(new Book(2, "iOS"));
        new Thread(new ServiceWorker()).start();
    }

    /**
     * 满足需求：用户不想时不时去查询图书列表，要起图书馆"有新书时能不能把书的信息告诉用户"
     *
     * @param book
     * @throws RemoteException
     */
    private void onNewBookArrived(Book book) throws RemoteException {
        mBookList.add(book);
//        Log.e(TAG, "onNewBookArrived,notify listeners:" + mListenerList.size());
//
//        for (int i = 0; i < mListenerList.size(); i++) {
//            IOnNewBookArrivedListener listener = mListenerList.get(i);
//            Log.e(TAG, "onNewBookArrived,notify listeners:" + listener);
//            listener.onNewBookArrived(book);
//        }

        int N = mListenerList.beginBroadcast();
        for (int j = 0; j < N; j++) {
            IOnNewBookArrivedListener broadcastItem = mListenerList.getBroadcastItem(j);
            if (broadcastItem != null) {
                broadcastItem.onNewBookArrived(book);
            }
        }
        mListenerList.finishBroadcast();
    }

    private class ServiceWorker implements Runnable {

        @Override
        public void run() {
            while (!mIsServiceDestroyed.get()) {
                try {
                    Thread.sleep(5000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

                int bookId = mBookList.size() + 1;
                Book book = new Book(bookId, "newBook#" + bookId);
                try {
                    onNewBookArrived(book);
                } catch (RemoteException e) {
                    e.printStackTrace();
                }
            }
        }
    }


    @Override
    public IBinder onBind(Intent intent) {
        return mBinder;
    }

    @Override
    public void onDestroy() {
        mIsServiceDestroyed.set(true);
        super.onDestroy();
    }
}
```
###### 3）客户端代码
```
public class BookManagerActivity extends AppCompatActivity {


    private static final String TAG = "BookManagerActivity";
    private static final int MESSAGE_NEW_BOOK_ARRIVED = 1;

    private IBookManager mRemoteBookManager;

    private Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MESSAGE_NEW_BOOK_ARRIVED:
                    Log.e(TAG, "receive new book :" + msg.obj);
                    break;
            }
        }
    };

    private ServiceConnection mConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            //客户端获取服务端binder对象
            IBookManager bookManager = IBookManager.Stub.asInterface(service);
            try {
                mRemoteBookManager = bookManager;

                //客户端获取服务端书籍数据
                List<Book> bookList = bookManager.getBookList();
                Log.e(TAG, "query book list,list type:" + bookList.getClass().getCanonicalName());
                Log.e(TAG, "query book list:" + bookList.toString());

                //客户端向服务端添加一本书
                Book newBook = new Book(3, "Android开发艺术探索");
                bookManager.addBook(newBook);
                Log.e(TAG, "add book:" + newBook);
                List<Book> newList = bookManager.getBookList();
                Log.e(TAG, "query book list:" + newList.toString());

                //注册有新书就能告知的监听
                bookManager.registerListener(mOnNewBookArrivedListener);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            //如果服务端进程意外停止了导致Binder意外死亡，可以在这里重连远程服务
            mRemoteBookManager = null;
            Log.e(TAG, "binder died");
        }
    };

    private IOnNewBookArrivedListener mOnNewBookArrivedListener = new IOnNewBookArrivedListener.Stub() {

        @Override
        public void onNewBookArrived(Book newBook) throws RemoteException {
            mHandler.obtainMessage(MESSAGE_NEW_BOOK_ARRIVED, newBook).sendToTarget();
        }
    };


    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_book_manager);

        //绑定远程服务
        Intent intent = new Intent(this, BookManagerService.class);
        bindService(intent, mConnection, Context.BIND_AUTO_CREATE);
    }

    @Override
    protected void onDestroy() {
        if (mRemoteBookManager != null && mRemoteBookManager.asBinder().isBinderAlive()) {
            Log.e(TAG, "unregister listener:" + mOnNewBookArrivedListener);
            try {
                mRemoteBookManager.unregisterListener(mOnNewBookArrivedListener);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }

        unbindService(mConnection);
        super.onDestroy();
    }
}
```
###### 4）AIDL 中使用权限验证功能
[AIDL权限验证](https://blog.csdn.net/mhtqq809201/article/details/50073657)

#### 相关参考
《Android开发艺术探索》
[安卓中本地服务Service和远程服务AIDL的使用](https://blog.csdn.net/gexiaoyizhimei/article/details/52435209)
[AIDL权限验证](https://blog.csdn.net/mhtqq809201/article/details/50073657)
