---
title: AndroidService
date: 2018-08-01 10:01:16
tags: [Android]
---
`Service`分为两种工作状态,一种是启动状态，主要用于执行后台计算;另一种是绑定态,主要用于其他组件和`Service`的交互。需要注意的是，`Service`的这两种状态是可以共存的，即`Service`既可以处于启动状态也可以同时处于绑定状态。
通过`Context`的`startService`方法即可启动一个`Service`:
``` java 
Intent intent = new Intent(content,Service.class);
startService(intent);
```
通过`Context`的`bindService`方法可以绑定一个`Service`：
``` java
Intent intent = new Intent(content,Service.class);
bindService(intent,connection,BIND_AUTO_CREATE);
```
<!--more-->
#### Service启动过程

`Service`的启动时从`ContextWrapper`的`startService`开始的：
``` java
@Override
    public ComponentName startService(Intent service) {
        return mBase.startService(service);
    }
```
`mBase`是`Context`的实现类`ContextImpl`，`在Activity`启动的时候会通过`attach`方法关联一个`ContextImpl`，这个`ContextImpl`就是上面的`mBase`，从`ContextWrapper`的实现来看，大部分的实现都是通过`mBase`来实现的，这是一种典型的桥接模式。在`ContextImpl`的`startService`方法中又调用了`startServiceCommon`。
``` java
 @Override
    public ComponentName startService(Intent service) {
        warnIfCallingFromSystemProcess();
        return startServiceCommon(service, false, mUser);
    }
private ComponentName startServiceCommon(Intent service, boolean requireForeground,UserHandle user) {

    validateServiceIntent(service);
    service.prepareToLeaveProcess(this);
    //api 27
    ComponentName cn = ActivityManager.getService().startService(
        mMainThread.getApplicationThread(), service, service.resolveTypeIfNeeded(
                    getContentResolver()), requireForeground,
                    getOpPackageName(), user.getIdentifier());
    //api 25
    //ComponentName cn = ActivityManagerNative.getDefault().startService(
     //   mMainThread.getApplicationThread(), service, service.resolveTypeIfNeeded(
      //  getContentResolver()), getOpPackageName(), user.getIdentifier());

    return cn;

}
```
在`startServiceCommon`中通过`getService`这个对象来启动一个服务，这个对象就是AMS，需要注意的是，在上述代码中通过AMS来启动服务的过程是一个跨进程调用。AMS的`startService`如下：
``` java
@Override
    public ComponentName startService(IApplicationThread caller, Intent service,
            String resolvedType, String callingPackage, int userId)
            throws TransactionTooLargeException {
        synchronized(this) {
            final int callingPid = Binder.getCallingPid();
            final int callingUid = Binder.getCallingUid();
            final long origId = Binder.clearCallingIdentity();
            ComponentName res = mServices.startServiceLocked(caller, service,
                    resolvedType, callingPid, callingUid, callingPackage, userId);
            Binder.restoreCallingIdentity(origId);
            return res;
        }
    }
```
AMS会通过`mServices`来完成`service`的启动过程，`mServices`的对象类型是`ActiveServices`，`ActiveServices`是一个辅助AMS进行`Service`管理的类，包括`Service`的启动、绑定和停止。在`startService`方法的尾部会调用`startServiceInnerLocked`方法
``` java
ComponentName startServiceInnerLocked(ServiceMap smap, Intent service, ServiceRecord r,
            boolean callerFg, boolean addToStarting) throws TransactionTooLargeException {
        ServiceState stracker = r.getTracker();
        if (stracker != null) {
            stracker.setStarted(true, mAm.mProcessStats.getMemFactorLocked(), r.lastActivity);
        }
        r.callStart = false;
        synchronized (r.stats.getBatteryStats()) {
            r.stats.startRunningLocked();
        }
        String error = bringUpServiceLocked(r, service.getFlags(), callerFg, false, false);
        return r.name;
    }
```
上述代码中`ServiceRecord`是描述一个Service记录，`ServiceRecord`一直贯穿整个Service流程，`startServiceInnerLocked`并没有完成启动Service的完整流程，而是将后续的过程交给了`bringUpServiceLocked`，在该方法中又调用了`realStartServiceLocked`方法:
``` java
private final void realStartServiceLocked(ServiceRecord r,
            ProcessRecord app, boolean execInFg) throws RemoteException {
       
        r.app = app;
        r.restartTime = r.lastActivity = SystemClock.uptimeMillis();

        final boolean newService = app.services.add(r);
        bumpServiceExecutingLocked(r, execInFg, "create");
        mAm.updateLruProcessLocked(app, false, null);
        mAm.updateOomAdjLocked();

        boolean created = false;
        try {
            
            synchronized (r.stats.getBatteryStats()) {
                r.stats.startLaunchedLocked();
            }
            mAm.notifyPackageUse(r.serviceInfo.packageName,
                                 PackageManager.NOTIFY_PACKAGE_USE_SERVICE);
            app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_SERVICE);
            app.thread.scheduleCreateService(r, r.serviceInfo,
                    mAm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo),
                    app.repProcState);
            r.postNotification();
            created = true;
        } catch (DeadObjectException e) {
            Slog.w(TAG, "Application dead when creating service " + r);
            mAm.appDiedLocked(app);
            throw e;
        } finally {
            if (!created) {
                // Keep the executeNesting count accurate.
                final boolean inDestroying = mDestroyingServices.contains(r);
                serviceDoneExecutingLocked(r, inDestroying, inDestroying);

                // Cleanup.
                if (newService) {
                    app.services.remove(r);
                    r.app = null;
                }

                // Retry.
                if (!inDestroying) {
                    scheduleServiceRestartLocked(r, false);
                }
            }
        }

        if (r.whitelistManager) {
            app.whitelistManager = true;
        }

        requestServiceBindingsLocked(r, execInFg);

        updateServiceClientActivitiesLocked(app, null, true);

        // If the service is in the started state, and there are no
        // pending arguments, then fake up one so its onStartCommand() will
        // be called.
        if (r.startRequested && r.callStart && r.pendingStarts.size() == 0) {
            r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(),
                    null, null));
        }

        sendServiceArgsLocked(r, execInFg, true);

        if (r.delayed) {
            if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE, "REM FR DELAY LIST (new proc): " + r);
            getServiceMap(r.userId).mDelayedStartList.remove(r);
            r.delayed = false;
        }

        if (r.delayedStop) {
            // Oh and hey we've already been asked to stop!
            r.delayedStop = false;
            if (r.startRequested) {
                if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE,
                        "Applying delayed stop (from start): " + r);
                stopServiceLocked(r);
            }
        }
    }
```
在`realStartServiceLocked`方法中，首先通过`app.thread`的`scheduleCreateService`方法来创建`Service`对象并调用其`onCreate`,接着再通过`sendServiceArgsLocked`方法来调用`Service`的其他方法，比如`onStartCommand`,这两个过程均是进程间通信。`app.thread`对象是`IApplicationThread`类型，它实际上是一个`Binder`,它的具体实现是`ApplicationThread`和`ApplicationThreadNative`。由于`ApplicationThread`继承了`ApplicationThreadNative`,因此只需要看`ApplicationThread`对`Service`启动过程的处理即可，这对应着它的`scheduleCreateService`方法，如下所示：
``` java
 public final void scheduleCreateService(IBinder token,
                ServiceInfo info, CompatibilityInfo compatInfo, int processState) {
            updateProcessState(processState, false);
            CreateServiceData s = new CreateServiceData();
            s.token = token;
            s.info = info;
            s.compatInfo = compatInfo;

            sendMessage(H.CREATE_SERVICE, s);
        }
```
通过发送消息给`Handler H`来完成的。H会接收这个`CREATE_ SERVICE`消息并通过`ActivityThread`的`handleCreateService`方法来完成Service的最终启动，`handleCreateService`的源码如下所示：
``` java
private void handleCreateService(CreateServiceData data) {
        // If we are getting ready to gc after going to the background, well
        // we are back active so skip it.
        unscheduleGcIdler();
        LoadedApk packageInfo = getPackageInfoNoCheck(
                data.info.applicationInfo, data.compatInfo);
        Service service = null;

            java.lang.ClassLoader cl = packageInfo.getClassLoader();
            service = (Service) cl.loadClass(data.info.name).newInstance();

            ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
            context.setOuterContext(service);

            Application app = packageInfo.makeApplication(false, mInstrumentation);
            service.attach(context, this, data.info.name, data.token, app,
                    ActivityManagerNative.getDefault());
            service.onCreate();
            mServices.put(data.token, service);
            ActivityManagerNative.getDefault().serviceDoneExecuting(
                        data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
    }

```
`handleCreateService`主要完成了以下几件事：
首先通过类加载器创建了Service实例，接着创建`ContextImpl`对象和`Application`对象  并通过service.attach方法建立联系，最后调用`service.onCreate`方法，并将`service`存储在`ActivityThread`中的一个列表`mServices`中
``` java
 final ArrayMap<IBinder, Service> mServices = new ArrayMap<>();
```
由于`service`的`onCreate`方法执行了，也就意味着`Service`已经启动了。除此之外，`ActivityThead`中还会通过`handleServiceArgs`方法调用`Service`的`onStartCommand`方法。

#### Service绑定过程
和启动过程一样，也是从`ContextWrapper`开始的：
``` java
@Override
public boolean bindService(Intent service, ServiceConnection conn,
        int flags) {
    return mBase.bindService(service, conn, flags);
}
```
然后是`ContextImpl`的`bindService`方法调用`bindServiceCommon`方法，然后远程调用AMS的`bindService`:
``` java
private boolean bindServiceCommon(Intent service, ServiceConnection conn, int flags, Handler handler, UserHandle user) {
    // Keep this in sync with DevicePolicyManager.bindDeviceAdminServiceAsUser.
    IServiceConnection sd;
    if (conn == null) {
        throw new IllegalArgumentException("connection is null");
    }
    if (mPackageInfo != null) {
        sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(), handler, flags);
    } else {
        throw new RuntimeException("Not supported in system context");
    }
    validateServiceIntent(service);
    try {
        IBinder token = getActivityToken();
        if (token == null && (flags&BIND_AUTO_CREATE) == 0 && mPackageInfo != null
                && mPackageInfo.getApplicationInfo().targetSdkVersion
                < android.os.Build.VERSION_CODES.ICE_CREAM_SANDWICH) {
            flags |= BIND_WAIVE_PRIORITY;
        }
        service.prepareToLeaveProcess(this);
        int res = ActivityManager.getService().bindService(
            mMainThread.getApplicationThread(), getActivityToken(), service,
            service.resolveTypeIfNeeded(getContentResolver()),
            sd, flags, getOpPackageName(), user.getIdentifier());
        if (res < 0) {
            throw new SecurityException(
                    "Not allowed to bind to service " + service);
        }
        return res != 0;
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
}
```
在该方法中，首先将客户端的`ServiceConnection`转化为`ServiceDispatcher.InnerConnection`对象，因为服务绑定是跨进程的，所以`ServiceConnection`对象必须借助`Binder`对象才能让远程服务端调用自己的方法。`ServiceDispatcher`起着连接`ServiceConnection`和`InnerConnection`的作用。这个过程由`LoadedApk.getServiceDispatcher`方法完成：
``` java
public final IServiceConnection getServiceDispatcher(ServiceConnection c,
            Context context, Handler handler, int flags) {
        synchronized (mServices) {
            LoadedApk.ServiceDispatcher sd = null;
            ArrayMap<ServiceConnection, LoadedApk.ServiceDispatcher> map = mServices.get(context);
            if (map != null) {
                if (DEBUG) Slog.d(TAG, "Returning existing dispatcher " + sd + " for conn " + c);
                sd = map.get(c);
            }
            if (sd == null) {
                sd = new ServiceDispatcher(c, context, handler, flags);
                if (DEBUG) Slog.d(TAG, "Creating new dispatcher " + sd + " for conn " + c);
                if (map == null) {
                    map = new ArrayMap<>();
                    mServices.put(context, map);
                }
                map.put(c, sd);
            } else {
                sd.validate(context, handler);
            }
            return sd.getIServiceConnection();
        }
    }
```
在上面的代码中，`mServices`是一个`ArrayMap`，它存储了一个应用当前活动的`ServiceConnection`和`ServiceDispatcher`的映射关系，其声明如下：
``` java
private final ArrayMap<Context, ArrayMap<ServiceConnection, LoadedApk.ServiceDispatcher>> mServices
        = new ArrayMap<>();
```
系统首先会查找是否存在相同的`ServiceConnection`,如果不存在就重新创建一个`ServiceDispatcher`对象，并将其存储在`mServices`中，其中映射关系的key是`ServiceConnection`,value是`ServiceDispatcher`,在`ServiceDispatcher`的内部又保存了`ServiceConnection`和`InnerConnection`对象。当`Service`和客户端建立连接后，系统会通过`InnerConnection`来调用`ServiceConnection`中的`onServiceConnected`方法，这个过程有可能是跨进程的。当`ServiceDispatcher`创建好了以后，`getServiceDispatcher`会 返回其保存的`InnerConnection`对象。
接着调用AMS的`bindService`  方法，该方法又调用了`bindServiceLocked`-->`bringUpServiceLocked`-->`realStartServiceLocked`，这个过程和上面的`StartService`过程逻辑类似，最终都是通过`ApplicationThread`来完成`Service`的实例创建并调用`onCreate`方法。和启动`Service`过程不同的是，绑定过程会调用`app.thread`的`scheduleBindService`方法，这个过程的实现在`ActivityService`的`requestServiceBindingsLocked`方法中：
``` java
 private final void requestServiceBindingsLocked(ServiceRecord r, boolean execInFg)
            throws TransactionTooLargeException {
        for (int i=r.bindings.size()-1; i>=0; i--) {
            IntentBindRecord ibr = r.bindings.valueAt(i);
            if (!requestServiceBindingLocked(r, ibr, execInFg, false)) {
                break;
            }
        }
    }

```
该方法用到了`r.bindings`。它是一个`ArrayMap`，保存了客户端的`bind`消息：
``` java
final ArrayMap<Intent.FilterComparison, IntentBindRecord> bindings = new ArrayMap<Intent.FilterComparison, IntentBindRecord>();
```
具体保存方法在AMS一开始的方法`bindServiceLocked`中：
``` java
AppBindRecord b = s.retrieveAppBindingLocked(service, callerApp);
```
在`requestServiceBindingsLocked`方法中调用了了`requestServiceBindingLocked`方法:
``` java
private final boolean requestServiceBindingLocked(ServiceRecord r, IntentBindRecord i,
            boolean execInFg, boolean rebind) throws TransactionTooLargeException {
        if (r.app == null || r.app.thread == null) {
            // If service is not currently running, can't yet bind.
            return false;
        }
        if (DEBUG_SERVICE) Slog.d(TAG_SERVICE, "requestBind " + i + ": requested=" + i.requested
                + " rebind=" + rebind);
        if ((!i.requested || rebind) && i.apps.size() > 0) {
            try {
                bumpServiceExecutingLocked(r, execInFg, "bind");
                r.app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_SERVICE);
                r.app.thread.scheduleBindService(r, i.intent.getIntent(), rebind,
                        r.app.repProcState);
                if (!rebind) {
                    i.requested = true;
                }
                i.hasBound = true;
                i.doRebind = false;
            } catch (TransactionTooLargeException e) {
                // Keep the executeNesting count accurate.
                if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Crashed while binding " + r, e);
                final boolean inDestroying = mDestroyingServices.contains(r);
                serviceDoneExecutingLocked(r, inDestroying, inDestroying);
                throw e;
            } catch (RemoteException e) {
                if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Crashed while binding " + r);
                // Keep the executeNesting count accurate.
                final boolean inDestroying = mDestroyingServices.contains(r);
                serviceDoneExecutingLocked(r, inDestroying, inDestroying);
                return false;
            }
        }
        return true;
    }
```
在上述代码中，`app.thread`这个对象多次出现过，它实际上就是`ApplicationThread`。`ApplicationThread`的一系列以`schedule`开头的方法，其内部都是通过`Handler H`来中转的，对于`scheduleBindService`方法来说也是如此，它的实现如下所示：
``` java
public final void scheduleBindService(IBinder token, Intent intent,
                boolean rebind, int processState) {
            updateProcessState(processState, false);
            BindServiceData s = new BindServiceData();
            s.token = token;
            s.intent = intent;
            s.rebind = rebind;
            sendMessage(H.BIND_SERVICE, s);
        }
```
在H内部，接收到`BIND_SERVICE`这类消息时，会交给`ActivityThread`的`handleBindService`方法来处理。在`handleBindService`中，首先根据`Service`的`token`取出`Service`对象，然后调用`Service`的`onBind`方法，`Service`的`onBind`方法会返回一个`Binder`对象给客户端使用，原则上来说，`Service`的`onBind`方法被调用以后，`Service`就处于绑定状态了，但是`onBind`方法是`Service`的方法，这个时候客户端并不知道已经成功连接`Service`了，所以还必须调用客户端的`ServiceConnection`中的`onServiceConnected`,这个过程是由`ActivityManager.getService()`的`publishService`方法来完成的，而前面多次提到，`ActivityManager.getService()`就是AMS。`handleBindService`的实现过程如下所示。
``` java
private void handleBindService(BindServiceData data) {
        Service s = mServices.get(data.token);
        if (s != null) {
            try {
                data.intent.setExtrasClassLoader(s.getClassLoader());
                data.intent.prepareToEnterProcess();
                try {
                    if (!data.rebind) {
                        IBinder binder = s.onBind(data.intent);
                        ActivityManager.getService().publishService(
                                data.token, data.intent, binder);
                    } else {
                        s.onRebind(data.intent);
                        ActivityManager.getService().serviceDoneExecuting(
                                data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
                    }
                    ensureJitEnabled();
                } catch (RemoteException ex) {
                    throw ex.rethrowFromSystemServer();
                }
            } catch (Exception e) {
                if (!mInstrumentation.onException(s, e)) {
                    throw new RuntimeException(
                            "Unable to bind to service " + s
                            + " with " + data.intent + ": " + e.toString(), e);
                }
            }
        }
    }
```
Service有一个特性，当多次绑定同一个Service时，Service的`onBind`方法**只会执行一次**，除非Service被终止了。当Service的onBind执行以后，系统还需要告知客户端已经成功连接Service了。根据上面的分析，这个过程由AMS的`publishService`方法来实现:
``` java
public void publishService(IBinder token, Intent intent, IBinder service) {
        // Refuse possible leaked file descriptors
        if (intent != null && intent.hasFileDescriptors() == true) {
            throw new IllegalArgumentException("File descriptors passed in Intent");
        }

        synchronized(this) {
            if (!(token instanceof ServiceRecord)) {
                throw new IllegalArgumentException("Invalid service token");
            }
            mServices.publishServiceLocked((ServiceRecord)token, intent, service);
        }
    }
```
从_上面代码可以看出，AMS的`publishService`方法将具体的工作交给了`ActiveServices`类型的`mServices`对象来处理。`ActiveServices`的`publishServiceLocked`方法看起来很复杂，但其实核心代码就只有一- 句话: `c.conn.connected(r.name,service)`， 其中c的类型是`ConnectionRecord`，`c.comn`的类型是`ServiceDispatcher.InnerConnection`, service就是Service的onBind方法返回的Binder对象。为了分析具体的逻辑，下面看一下`ServiceDispatcher.InnerConnection`的定义：
``` java
private static class InnerConnection extends IServiceConnection.Stub {
            final WeakReference<LoadedApk.ServiceDispatcher> mDispatcher;

            InnerConnection(LoadedApk.ServiceDispatcher sd) {
                mDispatcher = new WeakReference<LoadedApk.ServiceDispatcher>(sd);
            }

            public void connected(ComponentName name, IBinder service, boolean dead)
                    throws RemoteException {
                LoadedApk.ServiceDispatcher sd = mDispatcher.get();
                if (sd != null) {
                    sd.connected(name, service, dead);
                }
            }
        }
```
从`InnerConnection`的定义可以看出来，它的`connected`方法又调用了`ServiceDispatcher`的`connected`方法
``` java
public void connected(ComponentName name, IBinder service, boolean dead) {
            if (mActivityThread != null) {
                mActivityThread.post(new RunConnection(name, service, 0, dead));
            } else {
                doConnected(name, service, dead);
            }
        }
```
对于Service的绑定过程来讲，`ServiceDispatcher`中的`mActivityThread`就是一个`handler`，它就是`ActivityThread`中的`H`，从`Service`的创建过程来讲，`mActivityTHread`不会为`null`，这样一来，`RunConnection`就可以经由`H`的`post`方法从而运行在主线程中，因此，客户端的`ServiceConnection`中的方法回调是在主线程中执行的。
``` java
private final class RunConnection implements Runnable {
            RunConnection(ComponentName name, IBinder service, int command, boolean dead) {
                mName = name;
                mService = service;
                mCommand = command;
                mDead = dead;
            }

            public void run() {
                if (mCommand == 0) {
                    doConnected(mName, mService, mDead);
                } else if (mCommand == 1) {
                    doDeath(mName, mService);
                }
            }

            final ComponentName mName;
            final IBinder mService;
            final int mCommand;
            final boolean mDead;
        }

```
很显然，`RunConnection`的`run`方法也是简单调用了`ServiceDispatcher`的`doConnected`方法，由于`ServiceDispatcher`内部保存了客户端的`ServiceConnection`对象，因此它可以很方便地调用`ServiceConnection`对象的`onServiceConnected`方法，如下所示。
至此，bindService的过程完成。

----
以上