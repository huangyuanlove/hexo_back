---
title: Android多进程-二
date: 2018-06-22 16:03:38
tags: [Android]
---
Android中中IPC方式有很多，比如使用Bundle，使用文件共享，使用Messenger，使用AIDL，使用ContentProvider，使用Socket等。前两种方式比较简单，自己玩。
下面主要是抄的《Android开发艺术探索》2.4.3章节，看过书的就不用看了。
<!-- more -->
Messenger可以翻译为信使，顾名思义，通过它可以在不同进程中传递Message对象,在Message中放入我们需要传递的数据，就可以轻松地实现数据的进程间传递了。Messenger是一种轻量级的IPC方案，它的底层实现是AIDL，为什么这么说呢，我们大致看一下Messenger这个类的构造方法就明白了。下面是Messenger的两个构造方法，从构造方法的实现上我们可以明显看出AIDL的痕迹，不管是IMessenger还是Stub.asInterface,这种使用方法都表明它的底层是AIDL。
``` java
public Messenger (Handler target) {
    mTarget = target.getIMessenger();
}
public Messenger ( IBinder target) {
    mTarget = IMessenger.Stub.asInterface(target);
}
```
Messenger的使用方法很简单，它对AIDL做了封装，使得我们可以更简便地进行进程间通信。同时，由于它一次处理一个请求，因此在服务端我们不用考虑线程同步的问题，这是因为服务端中不存在并发执行的情形。实现一个Messenger有如下几个步骤，分为服务端和客户端。

1. 服务端进程
首先，我们需要在服务端创建--个Service来处理客户端的连接请求，同时创建一个Handler并通过它来创建一个Messenger对 象，然后在Service的onBind中返回这个Messenger对象底层的Binder即可。
2. 客户端进程
客户端进程中，首先要绑定服务端的Service,绑定成功后用服务端返回的IBinder对象创建一个Messenger,通过这个Messenger就可以向服务端发送消息了，发消息类型为Message对象。如果需要服务端能够回应客户端，就和服务端一-样，我们还需要创建一个Handler并创建一个 新的Messenger,并把这个Messenger对象通过Message的replyTo参数传递给服务端，服务端通过这个replyTo参数就可以回应客户端。

首先看服务端的代码，这是服务端的典型代码，可以看到MessengerHandler用来处理客户端发送的消息，并从消息中取出客户端发来的文本信息。而mMessenger是一个Messenger对象，它和MessengerHandler相关联，并在onBind方法中返回它里面的Binder对象，可以看出，这里Messenger的作用是将客户端发送的消息传递给MessengerHandler处理。

``` java
public class MessengerService extends Service {


    private static class MessengerHandler extends Handler{
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what){
                case 10001:
                    Log.e("xuan","收到客户端"+msg.getData().getString("msg"));
                    break;
                    default:
                        super.handleMessage(msg);
            }

        }
    }

    private final Messenger messenger = new Messenger(new MessengerHandler());
    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return messenger.getBinder();
    }
}

```
然后注册`service`,让其在单独的进程中运行
``` xml
<service
android: name=".messenger.MessengerService"
android: process=":remote">
```
在客户端,首先需要绑定远程进程的MessengerService,绑定成功后，根据服务端返回的binder对象创建Messenger对象并使用此对象向服务端发送消息。下面的代码在Bundle中向服务端发送了一句话，在，上面的服务端代码中会打印出这句话。

``` java
public class MessengerActivity extends AppCompatActivity {


    private Messenger mService;

    private ServiceConnection connection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            mService = new Messenger(service);
            Message msg = Message.obtain(null,10001);
            Bundle data = new Bundle();
            data.putString("msg","hi 来自客户端的问候");
            msg.setData(data);
            try {
                mService.send(msg);
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
        setContentView(R.layout.activity_messenger2);

        Intent intent = new Intent(this, MessengerService.class);
        bindService(intent,connection, Context.BIND_AUTO_CREATE);
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        unbindService(connection);
    }
}

```

通过上面的例子可以看出，在Messenger中进行数据传递必须将数据放入Message中，而Messenger和Message都实现了Parcelable接口，因此可以跨进程传输。简单来说,Message中所支持的数据类型就是Messenger所支持的传输类型。实际上，通过Messenger来传输Message, Message中 能使用的载体只有what、arg1、 arg2、 Bundle 以及replyTo。Message中的另一个字段object在同一-个进程中是很实用的，但是在进程间通信的时候，在Android2.2以前object字段不支持跨进程传输，即便是2.2以后，也仅仅是系统提供的实现了Parcelable接口的对象才能通过它来传输。这就意味着我们自定义的Parcelable对象是无法通过object字段来传输的，读者可以试一下。非系统的Parcelable对象的确无法通过object字段来传输，这也导致了object字段的实用性大大降低，所幸我们还有Bundle,Bundle中可以支持大量的数据类型。
上面的例子演示了如何在服务端接收客户端中发送的消息，但是有时候我们还需要能回应客户端，下面就介绍如何实现这种效果。还是采用上面的例子，但是稍微做一下修改，每当客户端发来一条消息，服务端就会自动回复一条“嗯，你的消息我已经收到，稍后会回复你。”，这很类似邮箱的自动回复功能。

首先看服务端的修改，服务端只需要修改MessengerHandler,当收到消息后，会立即回复一条消息给客户端。

``` java
public class MessengerService extends Service {
    private static class MessengerHandler extends Handler{
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what){
                case 10001:
                    Log.e("huangyuan","收到客户端"+msg.getData().getString("msg"));
                    Messenger clientMessenger = msg.replyTo;
                    Message message = Message.obtain(null,10001);
                    Bundle bundle = new Bundle();
                    bundle.putString("msg","服务端回应");
                    message.setData(bundle);
                    try {
                        clientMessenger.send(message);
                    } catch (RemoteException e) {

                        e.printStackTrace();
                    }
                    break;
                    default:
                        super.handleMessage(msg);
            }
        }
    }
    private final Messenger messenger = new Messenger(new MessengerHandler());
    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return messenger.getBinder();
    }
}

```

为了接收服务端的消息，客户端也需要准备一个接收消息的Messenger和Handler:
``` java
public class MessengerActivity extends AppCompatActivity {

    private Messenger mService;
    private Messenger getReplyMessenger = new Messenger(new MessengerHandler());
    private ServiceConnection connection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            mService = new Messenger(service);
            Message msg = Message.obtain(null,10001);
            Bundle data = new Bundle();
            data.putString("msg","hi 来自客户端的问候");
            msg.setData(data);
            msg.replyTo = getReplyMessenger;
            try {
                mService.send(msg);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
        }
    };

    private static class MessengerHandler  extends Handler{
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what){
                case 10001:
                    Log.e("huangyuan","收到服务端的回复-->"+ msg.getData().get("msg"));
                    break;
                    default:
                        super.handleMessage(msg);
            }
        }
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_messenger2);
        Intent intent = new Intent(this, MessengerService.class);
        bindService(intent,connection, Context.BIND_AUTO_CREATE);
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        unbindService(connection);
    }
}

```
关键的一点在于当客户端发送消息的时候，需要把接收服务端回复的Messenger通过Message的replyTo参数传递给服务端：
``` java
 msg.replyTo = getReplyMessenger;
```

----
以上