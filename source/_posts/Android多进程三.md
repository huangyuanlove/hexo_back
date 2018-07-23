---
title: Android多进程三
date: 2018-07-23 17:38:39
tags: [Android]
---
Android中中IPC方式有很多，比如使用Bundle，使用文件共享，使用Messenger，使用AIDL，使用ContentProvider，使用Socket等。前两种方式比较简单，自己玩。
下面主要是抄的《Android开发艺术探索》2.4.4章节，看过书的就不用看了。
<!-- more -->
上一篇主要抄了Messenger来进行进程间通信的方法，可以发现Messenger是以串行的方式处理客户端发来的消息，如果大量的消息同时发送到服务端，服务端仍然只能一个个处理，如果有大量的并发请求，那么用Messenger就不太合适了。同时，Messenger的作用主要是为了传递消息，很多时候我们可能需要跨进程调用服务端的方法，这种情形用Messenger就无法做到了，但是我们可以使用AIDL来实现跨进程的方法调用。AIDL也是Messenger的底层实现，因此Messenger本质上也是AIDL，只不过系统为我们做了封装从而方便上层的调用而已。在上一节中，我们介绍了Binder的概念，大家对Binder也有了一定的了解，在Binder的基础上我们可以更加容易地理解AIDL。这里先介绍使用AIDL来进行进程间通信的流程，分为服务端和客户端两个方面。
##### 服务端
服务端首先要创建一个Service用来监听客户端的连接请求，然后创建一个AIDL文件，将暴露给客户端的接口在这个AIDL文件中声明，最后在Service中实现这个AIDL接口即可。
##### 客户端
客户端所要做事情就稍微简单一些，首先需要绑定服务端的Service，绑定成功后，将服务端返回的Binder对象转成AIDL接口所属的类型，接着就可以调用AIDL中的方法了。

##### 具体实现方式
###### AIDL接口的创建
收看看AIDL接口的创建，如下所示创建了一个后缀为AIDL的文件，在里面声明了一个接口和两个方法。创建AIDL文件的方式可以看这个[Android多进程(一)](http://blog.huangyuanlove.com/2018/06/21/Android%E5%A4%9A%E8%BF%9B%E7%A8%8B-%E4%B8%80/#more)

``` java
// IBookManager.aidl
package com.huangyuanlove.testandroid;

// Declare any non-default types here with import statements
import com.huangyuanlove.testandroid.Book;
interface IBookManager {
    /**
     * Demonstrates some basic types that you can use as parameters
     * and return values in AIDL.
     */
    void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat,
            double aDouble, String aString);
            List<Book> getBookList();
            void addBook(in Book book);
}
```
在AIDL文件中，并不是所有的数据类型都是可以使用的，只支持如下几种类型：
* 基本数据类型（int、long、char、boolean、double等）；
* String和CharSequence；
* List：只支持ArrayList，里面每个元素都必须能够被AIDL支持；
* Map：只支持HashMap，里面的每个元素都必须被AIDL支持，包括key和value；
* Parcelable：所有实现了Parcelable接口的对象；
* AIDL：所有的AIDL接口本身也可以在AIDL文件中使用。

以上6种数据类型就是AIDL所支持的所有类型，其中自定义的Parcelable对象和AIDL对象必须要显式import进来，不管它们是否和当前的AIDL文件位于同一个包内。比如IBookManager.aidl这个文件，里面用到了Book这个类，这个类实现了Parcelable接口并且和IBookManager.aidl位于同一个包中，但是遵守AIDL的规范，我们仍然需要显式地import进来：com.huangyuanlove.testandroid.Book。
另外一个需要注意的地方是，如果AIDL文件中用到了自定义的Parcelable对象，那么必须新建一个和它同名的AIDL文件，并在其中声明它为Parcelable类型。在上面的IBookManager.aidl中，我们用到了Book这个类，所以，我们必须要创建Book.aidl，然后在里面添加如下内容：
``` java
package com.huangyuanlove.testandroid;
parcelable Book;
```

###### 远程服务端Service的实现
上面讲述了如何定义AIDL接口，接下来实现这个接口。先创建一个service，代码如下：
``` java
package com.huangyuanlove.testandroid;

import android.app.Service;
import android.content.Intent;
import android.os.Binder;
import android.os.IBinder;
import android.os.RemoteException;
import android.support.annotation.Nullable;

import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;

public class BookManagerService extends Service {

    private CopyOnWriteArrayList<Book> mBookList = new CopyOnWriteArrayList<>();

    private Binder mBinder = new IBookManager.Stub() {
        @Override
        public void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat, double aDouble, String aString) throws RemoteException {
        }

        @Override
        public List<Book> getBookList() throws RemoteException {
            return mBookList;
        }

        @Override
        public void addBook(Book book) throws RemoteException {
            mBookList.add(book);
        }
    };

    @Override
    public void onCreate() {
        super.onCreate();
        mBookList.add(new Book(1,"Android"));
        mBookList.add(new Book(2,"IOS"));
    }

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return mBinder;
    }
}
```
上面是一个服务端Service的典型实现，首先在onCreate中初始化添加了两本图书的信息，然后创建了一个Binder对象并在onBind中返回它，这个对象继承`IBookManager.Stub`并实现了它内部的AIDL方法，注意这里采用了CopyOnWriteArrayList，这个CopyOnWriteArrayList支持并发读/写。在前面
我们提到，AIDL方法是在服务端的Binder线程池中执行的，因此当多个客户端同时连接的时候，会存在多个线程同时访问的情形，所以我们要在AIDL方法中处理线程同步，而我们这里直接使用CopyOnWriteArrayList来进行自动的线程同步。AIDL中所支持的是抽象的List，而List只是一个接口，因此虽然服务端返回的是CopyOnWriteArrayList，但是在Binder中会按照List的规范去访问数据并最终形成一个新的ArrayList传递给客户端。所以，我们在服务端采用CopyOnWriteArrayList是完全可以的。和此类似的还有ConcurrentHashMap，然后我们需要在XML中注册这个Service:
``` xml
<service android:name=".BookManagerService"
            android:process=":remote"/>
```
###### 客户端的实现
客户端的实现就比较简单了，首先要绑定远程服务，绑定成功后将服务端返回的Binder对象转换成AIDL接口，然后就可以通过这个接口去调用服务端的远程方法了，代码如下所示：
``` java
public class MainActivity extends AppCompatActivity {

    private ServiceConnection mConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            IBookManager bookManager= IBookManager.Stub.asInterface(service);
            try {
                List<Book> list = bookManager.getBookList();
                Log.d("xuan","bookManager.getBookList()-->" + list.size());
                bookManager.addBook(new Book(3,"java"));
                List<Book> newList = bookManager.getBookList();
                Log.d("xuan","bookManager.getBookList()-->" + newList.size());
            }catch (RemoteException e){
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
        setContentView(R.layout.activity_main);
        Intent intent = new Intent(this,BookManagerService.class);
        bindService(intent,mConnection, Context.BIND_AUTO_CREATE);
    }

    @Override
    protected void onDestroy() {
        unbindService(mConnection);
        super.onDestroy();

    }
}
```
绑定成功以后，会通过bookManager去调用getBookList方法，然后打印出所获取的图书信息。需要注意的是，服务端的方法有可能需要很久才能执行完毕，这个时候下面的代码就会导致ANR，这一点是需要注意的，后面会再介绍这种情况，接着再调用一下另外一个接口addBook，我们在客户端给服务端添加一本书，然后再获取一次。
现在我们考虑一种情况，假设有一种需求：用户不想时不时地去查询图书列表了，太累了，于是，他去问图书馆，“当有新书时能不能把书的信息告诉我呢？”。这就是一种典型的观察者模式。
首先，我们需要提供一个AIDL接口，每个用户都需要实现这个接口并且向图书馆申请新书的提醒功能，当然用户也可以随时取消这种提醒。之所以选择AIDL
接口而不是普通接口，是因为AIDL中无法使用普通接口。这里我们创建一个IOnNewBookArrivedListener.aidl文件，我们所期望的情况是：当服务端有新书到来时，就会通知每一个已经申请提醒功能的用户。从程序上来说就是调用所有IOnNewBookArrivedListener对象中的onNewBookArrived方法，并把新书的对象通过参数传递给客户端，内容如下所示：
``` java
// IOnNewBookArrivedListener.aidl
package com.huangyuanlove.testandroid;

// Declare any non-default types here with import statements
import com.huangyuanlove.testandroid.Book;
interface IOnNewBookArrivedListener {
    /**
     * Demonstrates some basic types that you can use as parameters
     * and return values in AIDL.
     */
    void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat,
            double aDouble, String aString);
    void onNewBookArrived(in Book newBook);
}
```
AIDL中除了基本数据类型，其他类型的参数必须标上方向：in、out或者inout，in表示输入型参数，out表示输出型参数，inout表示输入输出型参数，至于它们具体的区别，官网是这么说的：
>All non-primitive parameters require a directional tag indicating which way the data goes . Either in , out , or inout . Primitives are in by default , and connot be otherwise .

>所有的非基本参数都需要一个定向tag来指出数据的流向，不管是 in , out , 还是 inout 。基本参数的定向tag默认是并且只能是 in 。

我们要根据实际需要去指定参数类型，不能一概使用out或者inout，因为这在底层实现是有开销的。最后，AIDL接口中只支持方法，不支持声明静态常量，这一点区别于传统的接口。
除了要新增加一个AIDL接口，还需要在原有的接口中添加两个新方法，代码如下：
``` java
package com.huangyuanlove.testandroid;

import com.huangyuanlove.testandroid.Book;
import com.huangyuanlove.testandroid.IOnNewBookArrivedListener;
interface IBookManager {
    void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat,
            double aDouble, String aString);
    List<Book> getBookList();
    void addBook(in Book book);
    void registerListener(IOnNewBookArrivedListener listener);
    void unRegisterListener(IOnNewBookArrivedListener listener);
}

```
接着，服务端中的Service的实现也需要修改一下，主要是Service中的IBookManager.Stub的实现，需要实现新增加的两个方法(IDE没有提示的话可以make一下)。同时，在BookManagerService中还开启了一个线程，每隔5s就向书库中增加一本新书并通知所有感兴趣的用户，整个代码如下所示：
``` java

public class BookManagerService extends Service {
    private AtomicBoolean mIsServiceDestroyed = new AtomicBoolean(false);
    private CopyOnWriteArrayList<Book> mBookList = new CopyOnWriteArrayList<>();
    private CopyOnWriteArrayList<IOnNewBookArrivedListener> mListenerList
            = new CopyOnWriteArrayList<>();

    private Binder mBinder = new IBookManager.Stub() {
        @Override
        public void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat, double aDouble, String aString) throws RemoteException {

        }

        @Override
        public List<Book> getBookList() throws RemoteException {
            return mBookList;
        }

        @Override
        public void addBook(Book book) throws RemoteException {
            mBookList.add(book);
        }

        @Override
        public void registerListener(IOnNewBookArrivedListener listener) throws RemoteException {
            if (!mListenerList.contains(listener)) {
                mListenerList.add(listener);
            } else {
                Log.d("xuan", "already exists.");
            }
            Log.d("xuan", "registerListener,size:" + mListenerList.size());
        }

        @Override
        public void unRegisterListener(IOnNewBookArrivedListener listener) throws RemoteException {
            if (mListenerList.contains(listener)) {
                mListenerList.remove(listener);
                Log.d("xuan", "unregister listener succeed.");
            } else {
                Log.d("xuan", "not found,can not unregister.");
            }
            Log.d("xuan", "unregisterListener,current size:" + mListenerList.size());
        }
    };


    @Override
    public void onCreate() {
        super.onCreate();

        mBookList.add(new Book(1, "Android"));
        mBookList.add(new Book(2, "IOS"));
        new Thread(new ServiceWorker()).start();
    }

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return mBinder;
    }

    @Override
    public void onDestroy() {
        mIsServiceDestroyed.set(true);
        super.onDestroy();
    }


    private void onNewBookArrived(Book book) throws RemoteException {
        mBookList.add(book);
        Log.d("xuan", "onNewBookArrived,notify listeners:" + mListenerList.
                size());
        for (int i = 0; i < mListenerList.size(); i++) {
            IOnNewBookArrivedListener listener = mListenerList.get(i);
            Log.d("xuan", "onNewBookArrived,notify listener:" + listener);
            listener.onNewBookArrived(book);
        }
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
                Book newBook = new Book(bookId, "new book#" + bookId);
                try {
                    onNewBookArrived(newBook);
                } catch (RemoteException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}

```
最后还需要修改一下客户端的代码，主要有两方面：首先客户端要注册IOnNewBookArrivedListener到远程服务端，这样当有新书时服务端才能通知当前客户端，同时我们要在Activity退出时解除这个注册；另一方面，当有新书时，服务端会回调客户端的IOnNewBookArrivedListener对象中的onNewBookArrived方法，但是这个方法是在客户端的Binder线程池中执行的，因此，为了便于进行UI操作，我们需要有一个Handler可以将其切换到客户端的主线程中去执行，代码如下：
``` java
public class MainActivity extends AppCompatActivity {

    private static final int MESSAGE_NEW_BOOK_ARRIVED = 1;
    private IBookManager mRemoteBookManager;

    private Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MESSAGE_NEW_BOOK_ARRIVED:
                    Log.d("MainActivity","receive new book :" + msg.obj);
                    break;
                default:
                    super.handleMessage(msg);
            }
        }
    };

    private ServiceConnection mConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            IBookManager bookManager= IBookManager.Stub.asInterface(service);
            try {
                mRemoteBookManager = bookManager;
                List<Book> list = bookManager.getBookList();
                Log.d("MainActivity","bookManager.getBookList()-->" + list.size() +">> " + list.toString());
                bookManager.addBook(new Book(3,"java"));
                List<Book> newList = bookManager.getBookList();
                Log.d("MainActivity","bookManager.getBookList()-->" + newList.size()+">> " + newList.toString());
                bookManager.registerListener(mOnNewBookArrivedListener);
            }catch (RemoteException e){
                e.printStackTrace();
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            mRemoteBookManager = null;
            Log.e("MainActivity","binder died.");
        }
    };


    private IOnNewBookArrivedListener mOnNewBookArrivedListener = new IOnNewBookArrivedListener.Stub() {
        @Override
        public void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat, double aDouble, String aString) throws RemoteException {

        }

        @Override
        public void onNewBookArrived(Book newBook) throws RemoteException {
            mHandler.obtainMessage(MESSAGE_NEW_BOOK_ARRIVED,newBook)
                    .sendToTarget();
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Intent intent = new Intent(this,BookManagerService.class);
        bindService(intent,mConnection, Context.BIND_AUTO_CREATE);
    }

    @Override
    protected void onDestroy() {
        if (mRemoteBookManager != null
                && mRemoteBookManager.asBinder().isBinderAlive()) {
            try {
                Log.d("MainActivity","unregister listener:" + mOnNewBookArrivedListener);
                mRemoteBookManager.unRegisterListener(mOnNewBookArrivedListener);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }
        unbindService(mConnection);
        super.onDestroy();
    }
}
```
从上面的代码可以看出，当BookManagerActivity关闭时，我们会在onDestroy中去解除已经注册到服务端的listener，这就相当于我们不想再接收图书馆的新书提醒了，所以我
们可以随时取消这个提醒服务。按back键退出BookManagerActivity,下面是打印出的log
``` log
07-23 14:56:55.493 16905-16918/com.huangyuanlove.testandroid:remote D/BookManagerService: not found,can not unregister.
07-23 14:56:57.185 16905-16918/com.huangyuanlove.testandroid:remote D/BookManagerService: unregisterListener,current size:1
```
从上面的log可以看出，程序没有像我们所预期的那样执行。在解注册的过程中，服务端竟然无法找到我们之前注册的那个listener，其实，这是必然的，这种解注册的处理方式在日常开发过程中时常使用到，但是放到多进程中却无法奏效，因为Binder会把客户端传递过来的对象重新转化并生成一个新的对象。虽然我们在注册和解注册过程中使用的是同一个客户端对象，但是通过Binder传递到服务端后，却会产生两个全新的对象。别忘了对象是不能跨进程直接传输的，对象的跨进程传输本质上都是反序列化的过程，这就是为什么AIDL中的自定义对象都必须要实现Parcelable接口的原因。可以使用`RemoteCallbackList`。
