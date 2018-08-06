---
title: Android多进程(一)
date: 2018-06-21 18:02:25
tags: [Android]
keywords: 多进程实现
---

Reference：《Android开发艺术探索》，作者：任玉刚
多进程基础以及一些名词
<!-- more -->
###### IPC

进程间通信或者跨进程通信，全称：Inter-Process Communication。
在操作系统中，线程是CPU调度的最小单元，同时线程是一种有限的系统资源。而进程一般指一个执行单元。一个进程可以包含做个线程。最简单的情况下，一个进程中可以只有一个线程，即主线程，在Android中，主线程也叫UI线程。
IPC不是Android中所独有的，任何一个操作系统都需要有相应的IPC机制，比如windows上可以通过剪贴板、管道和邮槽等来进行进程间通信；linux上可以通过命名管道、共享内容、信号量等来进行进程间通信。Android是基于Linux内核的移动操作系统，它的进程间通信方式没有完全继承自linux，在Android中可以通过Binder轻松实现进程间通信。除了Binder，Android还支持Socket。

###### 开启多进程

** 指定process属性 **

正常情况下，在Android中多进程是指一个应用中存在多个进程的情况，因此先忽略多个应用多进程的情况。首先在Android中使用多进程只有一种方式：在AndroidMenifest文件中指定`android:process`属性，除此之外还有一种非常规的方式，通过JNI在native层fork一个新的进程。也就是说我们无法给一个线程或者一个实体类指定其运行时所在的进程。

``` xml
<activity
	android:name=".ui.ActivityOne"
	android:process=":remote" />
<activity
	android:name=".ui.ActivityTwo"
	android:process="com.huangyuanlove.xuan" />
<activity
	android:name=".ui.ActivityThree" />
```

假如当前应用包名为`com.huangyuanlove.ipc`，当`ActivityOne`启动时，系统会为它创建一个单独的进程，进程名为`com.huangyuanlove.ipc:remote`，当`ActivityTwo`启动时，系统也会为它创建一个进程，进程名为：`com.huangyuanlove.xuan`，当然`ActivityThree`是运行在默认进程中，默认进程是包名。

** :name 和 全限定名的区别 **

* “:”的含义是在当前进程名的前面附加上包名(ActivityOne)，全限定名并不会附加包名。
* 以":"开头的进程属于当前应用的私有进程，其他应用的组件不可以和它跑在同一个进程中，其他不以":"开头的进程属于全局进程，其他应用通过ShareUID方式可以和它跑在同一个进程中。

​     Android会为每一个应用分配一个UID，具有相同UID的应用才能共享数据，需要注意的是，两个应用通过ShareUID跑在同一个进程中是有要求的，需要这两个应用有相同的ShareUID并且签名相同才可以。这种情况下，他们可以互相访问对方的私有数据，还可以共享内存数据，或者说它们看起来就像是一个应用的两个部分。

###### 带来的问题

* 静态成员和单例模式凉凉
* 线程同步锁机制凉凉
* SharedPreferences可靠性凉凉
* Application会创建多次

因为开启多进程之后，就不再是同一个内存区域，所以带来第一个问题，第二个问题也是同样，不在同一个内存区域，无论是对象锁还是全局锁可靠性基本就凉了。第三个问题和多进程写sp一样，第四个问题也是显而易见的，系统在创建新进程的时候会同时分配独立的虚拟机，所以这个过程就是启动一个应用的过程。

###### Parcelable 和 Serializable

这个自己玩

###### Binder

直观的讲，Binder是Android中的一个类，它继承了IBinder接口。从IPC角度来说，Binder是Android中的一种跨进程通信方式，从Android Framework角度来说，Binder是ServiceManager连接各种Manager(ActivityManager、WindowManager，等等)和相应ManagerService的桥梁；从Android应用层来说，Binder是客户端和服务端进行通信的媒介。

** 创建AIDL示例 **
创建一个Book类，实现Parcelable接口。

``` java
public class Book implements Parcelable {
    private int bookId;
    private String bookName;

    public Book(int bookId, String bookName) {
        this.bookId = bookId;
        this.bookName = bookName;
    }

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeInt(this.bookId);
        dest.writeString(this.bookName);
    }

    protected Book(Parcel in) {
        this.bookId = in.readInt();
        this.bookName = in.readString();
    }

    public static final Creator<Book> CREATOR = new Creator<Book>() {
        @Override
        public Book createFromParcel(Parcel source) {
            return new Book(source);
        }

        @Override
        public Book[] newArray(int size) {
            return new Book[size];
        }
    };
}

```
创建`IBookManager.aidl`文件，需要注意的是，在AndroidStudio中，右键创建aidl文件时候，IDE会自动创建一个和java平级的aidl文件夹，我们创建的aidl文件就在这里面。
![create_aidl_file](/image/Android/IPC/create_aidl.png )
** Book.aidl **
``` java
package com.example.huangyuan.testandroid;
parcelable Book;
```
`Book.aidl`是`Book.java`类在AIDL中的声明。

** IBookManager.aidl **
``` java
// IBookManager.aidl
package com.example.huangyuan.testandroid;
import com.example.huangyuan.testandroid.Book;
// Declare any non-default types here with import statements

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
其中 `basicTypes`是IDE自动生成的，我们自己添加 `getBookList()` 和 `addBook` 两个方法
尽管Book类和IBookManager的包名相同，但是在IBookManager中仍要导入Book类。下面看一下IDE生成的`IBookManager.java`类，该类在`app/build/generated/source/aidl/debug/packageName`包下。
``` java
/*
 * This file is auto-generated.  DO NOT MODIFY.
 * Original file: /Users/huangyuan/AndroidStudioProjects/TestAndroid/app/src/main/aidl/com/example/huangyuan/testandroid/IBookManager.aidl
 */
package com.example.huangyuan.testandroid;
// Declare any non-default types here with import statements

public interface IBookManager extends android.os.IInterface {
    /**
     * Local-side IPC implementation stub class.
     */
    public static abstract class Stub extends android.os.Binder implements com.example.huangyuan.testandroid.IBookManager {
        private static final java.lang.String DESCRIPTOR = "com.example.huangyuan.testandroid.IBookManager";

        /**
         * Construct the stub at attach it to the interface.
         */
        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }

        /**
         * Cast an IBinder object into an com.example.huangyuan.testandroid.IBookManager interface,
         * generating a proxy if needed.
         */
        public static com.example.huangyuan.testandroid.IBookManager asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin != null) && (iin instanceof com.example.huangyuan.testandroid.IBookManager))) {
                return ((com.example.huangyuan.testandroid.IBookManager) iin);
            }
            return new com.example.huangyuan.testandroid.IBookManager.Stub.Proxy(obj);
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
                case TRANSACTION_basicTypes: {
                    data.enforceInterface(DESCRIPTOR);
                    int _arg0;
                    _arg0 = data.readInt();
                    long _arg1;
                    _arg1 = data.readLong();
                    boolean _arg2;
                    _arg2 = (0 != data.readInt());
                    float _arg3;
                    _arg3 = data.readFloat();
                    double _arg4;
                    _arg4 = data.readDouble();
                    java.lang.String _arg5;
                    _arg5 = data.readString();
                    this.basicTypes(_arg0, _arg1, _arg2, _arg3, _arg4, _arg5);
                    reply.writeNoException();
                    return true;
                }
                case TRANSACTION_getBookList: {
                    data.enforceInterface(DESCRIPTOR);
                    java.util.List<com.example.huangyuan.testandroid.Book> _result = this.getBookList();
                    reply.writeNoException();
                    reply.writeTypedList(_result);
                    return true;
                }
                case TRANSACTION_addBook: {
                    data.enforceInterface(DESCRIPTOR);
                    com.example.huangyuan.testandroid.Book _arg0;
                    if ((0 != data.readInt())) {
                        _arg0 = com.example.huangyuan.testandroid.Book.CREATOR.createFromParcel(data);
                    } else {
                        _arg0 = null;
                    }
                    this.addBook(_arg0);
                    reply.writeNoException();
                    return true;
                }
            }
            return super.onTransact(code, data, reply, flags);
        }

        private static class Proxy implements com.example.huangyuan.testandroid.IBookManager {
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

            /**
             * Demonstrates some basic types that you can use as parameters
             * and return values in AIDL.
             */
            @Override
            public void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat, double aDouble, java.lang.String aString) throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    _data.writeInt(anInt);
                    _data.writeLong(aLong);
                    _data.writeInt(((aBoolean) ? (1) : (0)));
                    _data.writeFloat(aFloat);
                    _data.writeDouble(aDouble);
                    _data.writeString(aString);
                    mRemote.transact(Stub.TRANSACTION_basicTypes, _data, _reply, 0);
                    _reply.readException();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }

            @Override
            public java.util.List<com.example.huangyuan.testandroid.Book> getBookList() throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                java.util.List<com.example.huangyuan.testandroid.Book> _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    mRemote.transact(Stub.TRANSACTION_getBookList, _data, _reply, 0);
                    _reply.readException();
                    _result = _reply.createTypedArrayList(com.example.huangyuan.testandroid.Book.CREATOR);
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }

            @Override
            public void addBook(com.example.huangyuan.testandroid.Book book) throws android.os.RemoteException {
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
        }

        static final int TRANSACTION_basicTypes = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
        static final int TRANSACTION_getBookList = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
        static final int TRANSACTION_addBook = (android.os.IBinder.FIRST_CALL_TRANSACTION + 2);
    }

    /**
     * Demonstrates some basic types that you can use as parameters
     * and return values in AIDL.
     */
    public void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat, double aDouble, java.lang.String aString) throws android.os.RemoteException;

    public java.util.List<com.example.huangyuan.testandroid.Book> getBookList() throws android.os.RemoteException;

    public void addBook(com.example.huangyuan.testandroid.Book book) throws android.os.RemoteException;
}

```

结构比较简单，最外面是IBookManager接口，
其中声明了一个抽象内部类Stub，在该类中，声明了一个Proxy代理类。还声明了三个静态变量来标志aidl文件中的三个方法。这个Stub就是一个Binder类，当客户端和服务端都位于同一个进程时，方法调用不会走跨进程的transact过程，而当两者位于不同进程时，方法调用需要走transact过程，这个逻辑由内部Proxy类来完成。
最后，声明了aidl文件中的三个方法。
下面详细介绍：
** DESCRIPTOR **
Binder的唯一标示，一般用当前Binder的类名标示

** asInterface(android.os.IBinder obj) **
用于将服务端的BInder对象转换成客户端所需的AIDL借口类型的对象，这种转换过程是区分进程的，如果客户端和服务端位于同一进程，那么此方法返回的就是服务端的Stub对象本身，否则返回的是系统封装后的Stub.proxy对象。

** asBinder **
用于返回当前Binder对象

** onTransact **
这个方法运行在服务端中的Binder线程池中，当客户端发起跨进程请求时，远程请求会通过系统底层封装后交由此方法来处理。该方法的原型为public Boolean onTransact(int code,android.os. Parcel data,android.os.Parcel reply,int flags)。服务端通过code可以确定客户端所请求的目标方法是什么，接着从data中取出目标方法所需的参数(如果目标方法有参数的话)，然后执行目标方法。当目标方法执行完毕后，就向reply中写入返回值(如果目标方法有返回值的话)，onTransact方 法的执行过程就是这样的。需要注意的是，如果此方法返回false,那么客户端的请求会失败，因此我们可以利用这个特性来做权限验证，毕竟我们也不希望随便-一个进程都能远程调用我们的服务。

** Proxy#getBookList **
这个方法运行在客户端，当客户端远程调用此方法时，它的内部实现是这样的:首先创建该方法所需要的输入型Parcel对象_data、 输出型Parcel对象_ reply和返回值对象List;然后把该方法的参数信息写入_data中(如果有参数的话);接着调用transact方法来发起RPC (远程过程调用)请求，同时当前线程挂起;然后服务端的onTransact方法会被调用，直到RPC过程返回后，当前线程继续执行，并从_reply中 取出RPC过程的返回结果;最后返回_reply中的数据。

** Proxy#addBook **
这个方法运行在客户端，它的执行过程和getBookList是一样的，addBook没有返回值，所以它不需要从_replay中取出返回值。

接下来，我们介绍Binder的两个很重要的方法linkToDeath和unlinkToDeath。我们知道，Binder运行在服务端进程，如果服务端进程由于某种原因异常终止，这个时候我们到服务端的Binder连接断裂(称之为Binder死亡)，会导致我们的远程调用失败。更为关键的是，如果我们不知道Binder连接已经断裂，那么客户端的功能就会受到影响。为了解决这个问题，Binder中提供 了两个配对的方法linkToDeath和unlinkToDeath,通过linkToDeath我们可以给Binder设置- - 个死亡代理，当Binder死亡时， 我们就会收到通知，这个时候我们就可以重新发起连接请求从而恢复连接。那么到底如何给Binder设置死亡代理呢?也很简单。
首先，声明一个DeathRecipient对象。DeathRecipient是一个接口，其内部只有一个方法binderDied,我们需要实现这个方法，当Binder死亡的时 候，系统就会回调binderDied方法，然后我们就可以移出之前绑定的binder代理并重新绑定远程服务:
``` java
private IBinder .DeathRecipient mDeathRecipient = new IBinder.DeathRecipient() {
	@Override
	public void binderDied() {
		if ( mBookManager ==null)
				return;

		mBookManager.asBinder().unlinkToDeath(mDeathRecipient 0);
		mBookManager = null;
	}
);
```
其次，在客户端绑定远程服务成功后，给binder设置死亡代理。
``` java
mService = IMessageBoxManager.Stub.asInterface(binder);
binder.linkToDeath(mDeathRecipient,0);
```
其中linkToDeath的第二个参数是个标记位，我们直接设为0即可。经过上面两个步骤，就给我们的Binder设置了死亡代理，当Binder死 亡的时候我们就可以收到通知了。另外，通过Binder的 方法isBinderAlive也可以判断Binder是否死亡。

** 这一篇全部都是抄的《Android开发艺术探索》第2.3.3Binder 章节上的内容，毕竟一直带着书也不现实 **
----
以上