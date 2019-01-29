### 一、Binder概述
>&emsp;&emsp;Binder是一个实现了IBinder接口的一个类；
&emsp;&emsp;从IPC角度来说，Binder是Android中的一种跨进程通信方式，还可以理解为一种虚拟的物理设备，它的设备驱动是/dev/binder；
&emsp;&emsp;从Framework角度来说，Binder是ServiceManager连接各种Manager(ActivityManager,WindowManager)和相应ManagerService的桥梁；
&emsp;&emsp;从application来说，Binder是客户端和服务端进行通信的媒介，当bindService的时候，服务端会返回一个包含了服务端业务调用的Binder对象，通过这个对象客户端可以获取服务端提供的服务或数据。
### 二、通过AIDL来分析Binder的工作机制
&emsp;&emsp;创建AIDL文件后自动生成的xxx.java文件
```
public interface IBookManager extends android.os.IInterface {
    /**
     * Local-side IPC implementation stub class.
     */
    public static abstract class Stub extends android.os.Binder implements com.example.wood121.viewdemos.aidl.IBookManager {
        //DESCRIPTOR，作为Binder的唯一标识
        private static final java.lang.String DESCRIPTOR = "com.example.wood121.viewdemos.aidl.IBookManager";

        /**
         * Construct the stub at attach it to the interface.
         */
        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }

        /**
         * Cast an IBinder object into an com.example.wood121.viewdemos.aidl.IBookManager interface,
         * generating a proxy if needed.
         */
        //用于服务端的BInder对象转换为客户端所需的AIDL类型的对象
        public static com.example.wood121.viewdemos.aidl.IBookManager asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin != null) && (iin instanceof com.example.wood121.viewdemos.aidl.IBookManager))) {
                return ((com.example.wood121.viewdemos.aidl.IBookManager) iin);
            }
            return new com.example.wood121.viewdemos.aidl.IBookManager.Stub.Proxy(obj);
        }
      
        //返回当前的Binder对象
        @Override
        public android.os.IBinder asBinder() {
            return this;
        }
        
        /*这个方法运行在服务端的Binder连接池中，当客户端发起跨进程请求时，远程请求会通过系统底层封装后交由此方法来处理*/
        @Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
            switch (code) {
                case INTERFACE_TRANSACTION: {
                    reply.writeString(DESCRIPTOR);
                    return true;
                }
                case TRANSACTION_getBookList: {
                    data.enforceInterface(DESCRIPTOR);
                    java.util.List<com.example.wood121.viewdemos.aidl.Book> _result = this.getBookList();
                    reply.writeNoException();
                    reply.writeTypedList(_result);
                    return true;
                }
                case TRANSACTION_addBook: {
                    data.enforceInterface(DESCRIPTOR);
                    com.example.wood121.viewdemos.aidl.Book _arg0;
                    if ((0 != data.readInt())) {
                        _arg0 = com.example.wood121.viewdemos.aidl.Book.CREATOR.createFromParcel(data);
                    } else {
                        _arg0 = null;
                    }
                    this.addBook(_arg0);
                    reply.writeNoException();
                    return true;
                }
                case TRANSACTION_registerListener: {
                    data.enforceInterface(DESCRIPTOR);
                    com.example.wood121.viewdemos.aidl.IOnNewBookArrivedListener _arg0;
                    _arg0 = com.example.wood121.viewdemos.aidl.IOnNewBookArrivedListener.Stub.asInterface(data.readStrongBinder());
                    this.registerListener(_arg0);
                    reply.writeNoException();
                    return true;
                }
                case TRANSACTION_unregisterListener: {
                    data.enforceInterface(DESCRIPTOR);
                    com.example.wood121.viewdemos.aidl.IOnNewBookArrivedListener _arg0;
                    _arg0 = com.example.wood121.viewdemos.aidl.IOnNewBookArrivedListener.Stub.asInterface(data.readStrongBinder());
                    this.unregisterListener(_arg0);
                    reply.writeNoException();
                    return true;
                }
            }
            return super.onTransact(code, data, reply, flags);
        }

        private static class Proxy implements com.example.wood121.viewdemos.aidl.IBookManager {
            private android.os.IBinder mRemote;

            Proxy(android.os.IBinder remote) {
                mRemote = remote;
            }

            @Override
            public android.os.IBinder asBinder() {
                return mRemote;
            }

            public java.lang.String getInterfaceDescriptor() {
                return DESCRIPTOR;
            }

           //这个方法运行在客户端，当客户端远程调用此方法时，...
            @Override
            public java.util.List<com.example.wood121.viewdemos.aidl.Book> getBookList() throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                java.util.List<com.example.wood121.viewdemos.aidl.Book> _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    mRemote.transact(Stub.TRANSACTION_getBookList, _data, _reply, 0);
                    _reply.readException();
                    _result = _reply.createTypedArrayList(com.example.wood121.viewdemos.aidl.Book.CREATOR);
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }

            @Override
            public void addBook(com.example.wood121.viewdemos.aidl.Book book) throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    if ((book != null)) {
                        _data.writeInt(1);
                        book.writeToParcel(_data, 0);
                    } else {
                        _data.writeInt(0);
                    }
                    mRemote.transact(Stub.TRANSACTION_addBook, _data, _reply, 0);
                    _reply.readException();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }

            @Override
            public void registerListener(com.example.wood121.viewdemos.aidl.IOnNewBookArrivedListener listener) throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    _data.writeStrongBinder((((listener != null)) ? (listener.asBinder()) : (null)));
                    mRemote.transact(Stub.TRANSACTION_registerListener, _data, _reply, 0);
                    _reply.readException();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }

            @Override
            public void unregisterListener(com.example.wood121.viewdemos.aidl.IOnNewBookArrivedListener listener) throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    _data.writeStrongBinder((((listener != null)) ? (listener.asBinder()) : (null)));
                    mRemote.transact(Stub.TRANSACTION_unregisterListener, _data, _reply, 0);
                    _reply.readException();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }
        }

        static final int TRANSACTION_getBookList = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
        static final int TRANSACTION_addBook = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
        static final int TRANSACTION_registerListener = (android.os.IBinder.FIRST_CALL_TRANSACTION + 2);
        static final int TRANSACTION_unregisterListener = (android.os.IBinder.FIRST_CALL_TRANSACTION + 3);
    }

    public java.util.List<com.example.wood121.viewdemos.aidl.Book> getBookList() throws android.os.RemoteException;

    public void addBook(com.example.wood121.viewdemos.aidl.Book book) throws android.os.RemoteException;

    public void registerListener(com.example.wood121.viewdemos.aidl.IOnNewBookArrivedListener listener) throws android.os.RemoteException;

    public void unregisterListener(com.example.wood121.viewdemos.aidl.IOnNewBookArrivedListener listener) throws android.os.RemoteException;
}
```
![Binder工作机制.jpg](https://upload-images.jianshu.io/upload_images/4330197-d7d8d335ab1ab2f2.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
### 三、手写一个Binder
##### 1.声明一个AIDL性质的接口
```
public interface IBookManagerSelf extends IInterface {

    static final java.lang.String DESCRIPTOR = "com.example.wood121.viewdemos.aidl.IBookManagerSelf";

    static final int TRANSACTION_getBookList = IBinder.FIRST_CALL_TRANSACTION + 0;
    static final int TRANSACTION_addBook = IBinder.FIRST_CALL_TRANSACTION + 1;
    static final int TRANSACTION_registerListener = (IBinder.FIRST_CALL_TRANSACTION + 2);
    static final int TRANSACTION_unregisterListener = (IBinder.FIRST_CALL_TRANSACTION + 3);

    public List<Book> getBookList() throws android.os.RemoteException;

    public void addBook(Book book) throws android.os.RemoteException;

    public void registerListener(IOnNewBookArrivedListener listener) throws android.os.RemoteException;

    public void unregisterListener(IOnNewBookArrivedListener listener) throws android.os.RemoteException;

}
```
##### 2.实现Stub类和Stub类的Proxy代理
```
public class BookManagerImpl extends Binder implements IBookManagerSelf {

    public BookManagerImpl() {
        this.attachInterface(this, DESCRIPTOR);
    }

    /**
     * Cast an IBinder object into an com.example.wood121.viewdemos.aidl.IBookManagerSelf interface,
     * generating a proxy if needed.
     */
    public static IBookManagerSelf asInterface(android.os.IBinder obj) {
        if ((obj == null)) {
            return null;
        }
        android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
        if (((iin != null) && (iin instanceof IBookManagerSelf))) {
            return ((IBookManagerSelf) iin);
        }
        return new BookManagerImpl.Proxy(obj);
    }

    @Override
    public android.os.IBinder asBinder() {
        return this;
    }

    @Override
    public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
        switch (code) {
            case INTERFACE_TRANSACTION: {
                reply.writeString(DESCRIPTOR);
                return true;
            }
            case TRANSACTION_getBookList: {
                data.enforceInterface(DESCRIPTOR);
                java.util.List<com.example.wood121.viewdemos.aidl.Book> _result = this.getBookList();
                reply.writeNoException();
                reply.writeTypedList(_result);
                return true;
            }
            case TRANSACTION_addBook: {
                data.enforceInterface(DESCRIPTOR);
                com.example.wood121.viewdemos.aidl.Book _arg0;
                if ((0 != data.readInt())) {
                    _arg0 = com.example.wood121.viewdemos.aidl.Book.CREATOR.createFromParcel(data);
                } else {
                    _arg0 = null;
                }
                this.addBook(_arg0);
                reply.writeNoException();
                return true;
            }
            case TRANSACTION_registerListener: {
                data.enforceInterface(DESCRIPTOR);
                com.example.wood121.viewdemos.aidl.IOnNewBookArrivedListener _arg0;
                _arg0 = com.example.wood121.viewdemos.aidl.IOnNewBookArrivedListener.Stub.asInterface(data.readStrongBinder());
                this.registerListener(_arg0);
                reply.writeNoException();
                return true;
            }
            case TRANSACTION_unregisterListener: {
                data.enforceInterface(DESCRIPTOR);
                com.example.wood121.viewdemos.aidl.IOnNewBookArrivedListener _arg0;
                _arg0 = com.example.wood121.viewdemos.aidl.IOnNewBookArrivedListener.Stub.asInterface(data.readStrongBinder());
                this.unregisterListener(_arg0);
                reply.writeNoException();
                return true;
            }
        }
        return super.onTransact(code, data, reply, flags);
    }

    @Override
    public List<Book> getBookList() throws RemoteException {
        return null;
    }

    @Override
    public void addBook(Book book) throws RemoteException {

    }

    @Override
    public void registerListener(IOnNewBookArrivedListener listener) throws RemoteException {

    }

    @Override
    public void unregisterListener(IOnNewBookArrivedListener listener) throws RemoteException {

    }


    private static class Proxy implements IBookManagerSelf {
        private android.os.IBinder mRemote;

        Proxy(android.os.IBinder remote) {
            mRemote = remote;
        }

        @Override
        public android.os.IBinder asBinder() {
            return mRemote;
        }

        public java.lang.String getInterfaceDescriptor() {
            return DESCRIPTOR;
        }

        @Override
        public java.util.List<com.example.wood121.viewdemos.aidl.Book> getBookList() throws android.os.RemoteException {
            android.os.Parcel _data = android.os.Parcel.obtain();
            android.os.Parcel _reply = android.os.Parcel.obtain();
            java.util.List<com.example.wood121.viewdemos.aidl.Book> _result;
            try {
                _data.writeInterfaceToken(DESCRIPTOR);
                mRemote.transact(BookManagerImpl.TRANSACTION_getBookList, _data, _reply, 0);
                _reply.readException();
                _result = _reply.createTypedArrayList(com.example.wood121.viewdemos.aidl.Book.CREATOR);
            } finally {
                _reply.recycle();
                _data.recycle();
            }
            return _result;
        }

        @Override
        public void addBook(com.example.wood121.viewdemos.aidl.Book book) throws android.os.RemoteException {
            android.os.Parcel _data = android.os.Parcel.obtain();
            android.os.Parcel _reply = android.os.Parcel.obtain();
            try {
                _data.writeInterfaceToken(DESCRIPTOR);
                if ((book != null)) {
                    _data.writeInt(1);
                    book.writeToParcel(_data, 0);
                } else {
                    _data.writeInt(0);
                }
                mRemote.transact(BookManagerImpl.TRANSACTION_addBook, _data, _reply, 0);
                _reply.readException();
            } finally {
                _reply.recycle();
                _data.recycle();
            }
        }

        @Override
        public void registerListener(com.example.wood121.viewdemos.aidl.IOnNewBookArrivedListener listener) throws android.os.RemoteException {
            android.os.Parcel _data = android.os.Parcel.obtain();
            android.os.Parcel _reply = android.os.Parcel.obtain();
            try {
                _data.writeInterfaceToken(DESCRIPTOR);
                _data.writeStrongBinder((((listener != null)) ? (listener.asBinder()) : (null)));
                mRemote.transact(BookManagerImpl.TRANSACTION_registerListener, _data, _reply, 0);
                _reply.readException();
            } finally {
                _reply.recycle();
                _data.recycle();
            }
        }

        @Override
        public void unregisterListener(com.example.wood121.viewdemos.aidl.IOnNewBookArrivedListener listener) throws android.os.RemoteException {
            android.os.Parcel _data = android.os.Parcel.obtain();
            android.os.Parcel _reply = android.os.Parcel.obtain();
            try {
                _data.writeInterfaceToken(DESCRIPTOR);
                _data.writeStrongBinder((((listener != null)) ? (listener.asBinder()) : (null)));
                mRemote.transact(BookManagerImpl.TRANSACTION_unregisterListener, _data, _reply, 0);
                _reply.readException();
            } finally {
                _reply.recycle();
                _data.recycle();
            }
        }
    }

}
```
#### 四、在源码中的应用分析
&emsp;&emsp;通过手写Binder我们了解了一种不通过AIDL方式来使用的Binder机制，在Activity的启动过程中，我们会涉及到以下几个接口与类：
```
public interface IActivityManager extends IInterface 
public abstract class ActivityManagerNative extends Binder implements IActivityManager
class ActivityManagerProxy implements IActivityManager

public interface IApplicationThread extends IInterface
public abstract class ApplicationThreadNative extends Binder implements IApplicationThread
private class ApplicationThread extends ApplicationThreadNative
```
###### 1、我们分析一下IActivityManager, ActivityManagerNative, ActivityManagerProxy以及相关的ActivityManagerService, ActivityManager
。。。给连接[Android源码学习之六——ActivityManager框架解析](https://blog.csdn.net/caowenbin/article/details/6036726)


