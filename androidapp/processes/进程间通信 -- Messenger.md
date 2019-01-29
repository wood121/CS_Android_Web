&emsp;&emsp;相关阅读连接：[Android IPC机制及AIDL](https://www.jianshu.com/p/92b4ba022f82)

### 一、概述 
>&emsp;&emsp; Messenger是一种轻量级的IPC方案，它的底层实现是AIDL。
&emsp;&emsp; 服务端进程：创建一个Service来处理客户端的连接请求；同时创建一个Handler并通过它来创建一个Messager对象；在onBind方法中返回这个Messager对象底层的Binder对象。
&emsp;&emsp; 客户端进程：绑定服务端Service，绑定成功后用服务器返回的Binder创建Messenger，这个Messager就可以向服务端发送消息，发送的消息类型为Messaged；
### 二、客户端向服务端发送消息
```
public class MessengerActivity extends AppCompatActivity {
    private static final String TAG = MessengerActivity.class.getSimpleName() + " :";
    private Messenger mMService;
    private Messenger mGetReplyMessenger = new Messenger(new MessgerHandler());

    private static class MessgerHandler extends Handler {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case 102:
                    Log.e("wood121", TAG + "receiver from server:" + msg.getData().get("reply"));
                    break;
                default:
                    super.handleMessage(msg);
            }
        }
    }

    private ServiceConnection mConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            Log.e("wood121", TAG + "onServiceConnected");
            //这是客户端得到Binder对象、创建Messenger、而后通过Messenger对象给服务端发消息
            mMService = new Messenger(service);
            Message msg = Message.obtain(null, 101);
            Bundle bundle = new Bundle();
            bundle.putString("msg", "hello,this is client");
            msg.setData(bundle);
            msg.replyTo = mGetReplyMessenger;
            try {
                mMService.send(msg);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {

        }
    };


    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_messenger);

        Intent intent = new Intent(this, MessengerService.class);
        bindService(intent, mConnection, Context.BIND_AUTO_CREATE);
    }

    @Override
    protected void onDestroy() {
        unbindService(mConnection);
        super.onDestroy();
    }
}
```
### 三、服务端向客户端发送消息
```
public class MessengerService extends Service {
    private static final String TAG = MessengerService.class.getSimpleName() + " :";
    private final Messenger mMessenger = new Messenger(new MessengerHandler());

    public MessengerService() {
    }

    @Override
    public IBinder onBind(Intent intent) {
        return mMessenger.getBinder();
    }

    private static class MessengerHandler extends Handler {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case 101:
                    //接收的来自客户端的问候
                    Log.e("wood121", TAG + "receive msg from client:" + msg.getData().get("msg"));

                    //服务端给客户端回复一条消息
                    Messenger client = msg.replyTo;
                    Message msgService = Message.obtain(null, 102);
                    Bundle bundlerService = new Bundle();
                    bundlerService.putString("reply", "嗯，你的消息我已经收到了，稍后会回复你");
                    msgService.setData(bundlerService);
                    try {
                        client.send(msgService);
                    } catch (RemoteException e) {
                        e.printStackTrace();
                    }
                    break;
            }
            super.handleMessage(msg);
        }
    }
}

//注意manifest文件中的注册
<service
    android:name=".service.MessengerService"
    android:process=":remote"
    android:enabled="true"
    android:exported="true"/>
```
&emsp;&emsp;**注意**：在查看Logcat过程中，客户端与服务端处于两个不同的进程，需要对应选中，如下图所示：
![查看客户端与服务端的日志](https://upload-images.jianshu.io/upload_images/4330197-69254e44472697a4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 四、工作原理
![Messenger消息传输原理图](https://upload-images.jianshu.io/upload_images/4330197-4ebb6d1332e1a9a7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

