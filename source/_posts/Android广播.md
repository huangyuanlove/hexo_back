---
title: Android广播
date: 2018-07-31 15:16:25
tags: [Android]
---
面试APUS的时候被问到广播：
面试官：聊一下广播吧。
我：广播啊，四大组件之一，自己创建个类继承自`BroadcastReceiver`，重写`onReceive()`方法，需要注意的是不要在这个方法中做耗时操作。注册的话可以在`AndroidManifest`文件中静态注册，也可以在代码中动态注册。都做完了之后就可以调用sendBroadCast()方法发送广播了。
面试官：了解广播注册过程么？
我：哈~！这个没怎么看过。
面试官：了解怎么接收到的广播么？
我：没有。。。。
<!--more-->

#### 广播的注册过程
广播的注册分为静态注册和动态注册，其中静态注册的广播在应用安装时由系统自动完成注册，具体来说是由PMS ( PackageManagerService)来完成整个注册过程的，除了广播以外，其他三大组件也都是在应用安装时由PMS解析并注册的。这里只分析广播的动态注册的过程，动态注册的过程是从ContextWrapper的registerReceiver方法开始的，和Activity以及Service一样 。ContextWrapper并 没有做实际的工作，而是将注册过程直接交给了ContextImpl来完成，如下所示：
``` java
    @Override
    public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter) {
        return mBase.registerReceiver(receiver, filter);
    }
    @Override
    public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter,String broadcastPermission, Handler scheduler) {
        return registerReceiverInternal(receiver, getUserId(),
                filter, broadcastPermission, scheduler, getOuterContext(), 0);
    }
```
这里的`registerReceiver`方法是重载方法，最终调用了`registerReceiverInternal`方法：
``` java
private Intent registerReceiverInternal(BroadcastReceiver receiver, int userId,
            IntentFilter filter, String broadcastPermission,
            Handler scheduler, Context context, int flags) {
        IIntentReceiver rd = null;
        if (receiver != null) {
            if (mPackageInfo != null && context != null) {
                if (scheduler == null) {
                    scheduler = mMainThread.getHandler();
                }
                rd = mPackageInfo.getReceiverDispatcher(
                    receiver, context, scheduler,
                    mMainThread.getInstrumentation(), true);
            } else {
                if (scheduler == null) {
                    scheduler = mMainThread.getHandler();
                }
                rd = new LoadedApk.ReceiverDispatcher(
                        receiver, context, scheduler, null, true).getIIntentReceiver();
            }
        }
        try {
            final Intent intent = ActivityManager.getService().registerReceiver(
                    mMainThread.getApplicationThread(), mBasePackageName, rd, filter,
                    broadcastPermission, userId, flags);
            if (intent != null) {
                intent.setExtrasClassLoader(getClassLoader());
                intent.prepareToEnterProcess();
            }
            return intent;
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
```
在上面的代码中，系统首先从`mPackageInfo`获取`IIntentReceiver`对象，然后再采用跨进程的方式向AMS发送广播注册的请求。之所以采用`IIntentReceiver`而不是直接采用`BroadcastReceiver` ,这是因为上述注册过程是一个进程间通信的过程，而`BroadcastReceiver`作为Android的一个组件是不能直接跨进程传递的，所以需要通过`IIntentReceiver`来中转一下。毫无疑问，`IIntentReceiver`必须是一个Binder接口，它的具体实现是`LoadedApk.ReceiverDispatcher.InnerReceiver`, `ReceiverDispatcher`的内部同时保存了`BroadcastReceiver`和`InnerReceiver`,这样当接收到广播时，`ReceiverDispatcher`可以很方便地调用`BroadcastReceiver`的`onReceive`方法。
看一下`LoadedApk.getReceiverDispatcher`方法：
``` java
public IIntentReceiver getReceiverDispatcher(BroadcastReceiver r,
            Context context, Handler handler,
            Instrumentation instrumentation, boolean registered) {
        synchronized (mReceivers) {
            LoadedApk.ReceiverDispatcher rd = null;
            ArrayMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher> map = null;
            if (registered) {
                map = mReceivers.get(context);
                if (map != null) {
                    rd = map.get(r);
                }
            }
            if (rd == null) {
                rd = new ReceiverDispatcher(r, context, handler,
                        instrumentation, registered);
                if (registered) {
                    if (map == null) {
                        map = new ArrayMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher>();
                        mReceivers.put(context, map);
                    }
                    map.put(r, rd);
                }
            } else {
                rd.validate(context, handler);
            }
            rd.mForgotten = false;
            return rd.getIIntentReceiver();
        }
    }
```
很显然，`getReceiverDispatcher`方法重新创建了一个`ReceiverDispatcher`对象并将其保存的`InnerReceiver`对象作为返回值返回，其中`InnerReceiver`对象和`BroadcastReceiver`都是在`ReceiverDispatcher`的构造方法中被保存起来的。
由于注册广播的真正实现过程是在AMS中，因此我们需要看一下AMS的具体实现。AMS的`registerReceiver`方法看起来很长，其实关键点就只有下面一部分，最终会把远程的`InnerReceiver`对象以及`IntentFilter`对象存储起来，这样整个广播的注册过程就完成了，代码如下所示。
``` java
public Intent registerReceiver(IApplicationThread caller, String callerPackage,IIntentReceiver receiver, IntentFilter filter, String permission, int userId) {
    // The first sticky in the list is returned directly back to the client.
    Intent sticky = allSticky != null ? allSticky.get(0) : null;
    ReceiverList rl = mRegisteredReceivers.get(receiver.asBinder());
    BroadcastFilter bf = new BroadcastFilter(filter, rl, callerPackage,permission, callingUid, userId);
    rl.add(bf);
    mReceiverResolver.addFilter(bf);
    return sticky;
}
```

#### 广播的发送过程

当通过send方法来发送广播时，AMS会查找出匹配的广播接收者并将广播发送给它们处理。广播的发送有几种类型:普通广播、有序广播和粘性广播，有序广播和粘性广播与普通广播相比具有不同的特性，但是它们的发送/接收过程的流程是类似的，因此这里只分析普通厂播的实现。广播的发送和接收，其本质是一个过程的两个阶段。这里从广播的发送可以说起，广播的发送仍然开始于`ContextWrapper`的`sendBroadcast`方法，之所以不是`Context`,那是因为`Context`的`sendBroadcast`是-一个抽象方法。和广播的注册过程一样，`ContextWrapper`的`sendBroadcast`方法仍然什么都不做，只是把事情交给`ContextImpl`去处理，`ContextImpl的sendBroadcast`方法的源码如下所示。
``` java
@Override
    public void sendBroadcast(Intent intent) {
        warnIfCallingFromSystemProcess();
        String resolvedType = intent.resolveTypeIfNeeded(getContentResolver());
        try {
            intent.prepareToLeaveProcess(this);
            ActivityManager.getService().broadcastIntent(
                    mMainThread.getApplicationThread(), intent, resolvedType, null,
                    Activity.RESULT_OK, null, null, null, AppOpsManager.OP_NONE, null, false, false,
                    getUserId());
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
```
从_上面的代码来看，`ContextImpl`也是几乎什么事都没干，它直接向AMS发起了一个异步请求用于发送广播。因此，下面直接看AMS对广播发送过程的处理，AMS的`broadcastIntent`方法又调用了`broadcastIntentLocked`,在这个方法的开始有这么一行：
``` java
// By default broadcasts do not go to stopped apps.
intent.addFlags(Intent.FLAG_EXCLUDE_STOPPED_PACKAGES);
```
从Android3.1开始广播默认情况下广播不会发送给已经停止的应用。这是因为系统在Android3.1中为Intent添加了两个标记位，分别是`FLAG_INCLUDE_STOPPED_PACKAGES`和`FLAG_EXCLUDE_STOPPED_PACKAGES`，用来控制广播是否要对处于停止状态的应用起作用，它们的含义如下所示。
** FLAG_INCLUDE_STOPPED_PACKAGES **
表示包含已经停止的应用，这个时候广播会发送给已经停止的应用。
** FLAG_EXCLUDE_STOPPED_PACKAGES **
表示不包含已经停止的应用，这个时候广播不会发送给已经停止的应用。

从Android3.1开始，系统为所有广播默认添加了`FLAG_EXCLUDE_STOPPED_PACKAGES`标志，这样做是为了防止广播无意间或者在不必要的时候调起已经停止运行的应用。如果的确需要调起未启动的应用，那么只需要为广播的Intent添加`FLAG_INCLUDE_STOPPED_PACKAGES`标记即可。当`FLAG_EXCLUDE_STOPPED_PACKAGES`和`FLAG_INCLUDE_STOPPED_PACKAGES`两种标记位共存时,以`FLAG_INCLUDE_STOPPED_PACKAGES`为准。这里需要补充一下，一个应用处于停止状态分为两种情形:
第一种是应用安装后未运行，
第二种是应用被手动或者其他应用强停了。
Android3.1中广播的这个特性同样会影响开机广播，从Android3.1开始，处于停止状态的应用同样无法接收到开机广播，而在Android 3.1之前，处于停止状态的应用是可以收到开机广播的。
在`broadcastIntentLocked`的内部，会根据`intent-filter`查找出匹配的广播接收者并经过一系列的条件过滤，最终会将满足条件的广播接收者添加到`BroadcastQueue`中，接着`BroadcastQueue`就会将广播发送给相应的广播接收者，这个过程的源码如下所示。
``` java
if ((receivers != null && receivers.size() > 0)
                || resultTo != null) {
            BroadcastQueue queue = broadcastQueueForIntent(intent);
            BroadcastRecord r = new BroadcastRecord(queue, intent, callerApp,
                    callerPackage, callingPid, callingUid, resolvedType,
                    requiredPermissions, appOp, brOptions, receivers, resultTo, resultCode,
                    resultData, resultExtras, ordered, sticky, false, userId);

            if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, "Enqueueing ordered broadcast " + r
                    + ": prev had " + queue.mOrderedBroadcasts.size());
            if (DEBUG_BROADCAST) Slog.i(TAG_BROADCAST,
                    "Enqueueing broadcast " + r.intent.getAction());

            boolean replaced = replacePending && queue.replaceOrderedBroadcastLocked(r);
            if (!replaced) {
                queue.enqueueOrderedBroadcastLocked(r);
                queue.scheduleBroadcastsLocked();
            }
        } else {
            // There was nobody interested in the broadcast, but we still want to record
            // that it happened.
            if (intent.getComponent() == null && intent.getPackage() == null
                    && (intent.getFlags()&Intent.FLAG_RECEIVER_REGISTERED_ONLY) == 0) {
                // This was an implicit broadcast... let's record it for posterity.
                addBroadcastStatLocked(intent.getAction(), callerPackage, 0, 0, 0);
            }
        }
```
将广播添加到`BroadCastQueue`之后，接着调用了`scheduleBroadcastsLocked`方法：
``` java
 public void scheduleBroadcastsLocked() {
        if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, "Schedule broadcasts ["
                + mQueueName + "]: current="
                + mBroadcastsScheduled);

        if (mBroadcastsScheduled) {
            return;
        }
        mHandler.sendMessage(mHandler.obtainMessage(BROADCAST_INTENT_MSG, this));
        mBroadcastsScheduled = true;
    }
```
方法内发送了一个`BROADCAST_INTENT_MSG`消息，handler接到消息后，调用了`processNextBroadcast`方法

``` java
 private final class BroadcastHandler extends Handler {
        public BroadcastHandler(Looper looper) {
            super(looper, null, true);
        }

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case BROADCAST_INTENT_MSG: {
                    if (DEBUG_BROADCAST) Slog.v(
                            TAG_BROADCAST, "Received BROADCAST_INTENT_MSG");
                    processNextBroadcast(true);
                } break;
                case BROADCAST_TIMEOUT_MSG: {
                    synchronized (mService) {
                        broadcastTimeoutLocked(true);
                    }
                } break;
                case SCHEDULE_TEMP_WHITELIST_MSG: {
                    DeviceIdleController.LocalService dic = mService.mLocalDeviceIdleController;
                    if (dic != null) {
                        dic.addPowerSaveTempWhitelistAppDirect(UserHandle.getAppId(msg.arg1),
                                msg.arg2, true, (String)msg.obj);
                    }
                } break;
            }
        }
    }
```
