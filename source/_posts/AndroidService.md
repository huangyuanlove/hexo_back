---
title: AndroidService
date: 2018-08-01 10:01:16
tags: [Android]
---
过程。在分析Service的工作过程之前，先看一~下如何使用- -个Service。Service分 为两种工作状态,一种是启动状态，主要用于执行后台计算;另一种是绑定态,主要用于其他组件和Service的交互。需要注意的是，Service的这两种状态是可以共存的，即Service既可以处于启动状态也可以同时处于绑定状态。
通过Context的startService方法即可启动一个Service:
``` java 
Intent intent = new Intent(content,Service.class);
startService(intent);
```

通过Context的bindService方法可以绑定一个Service：
``` java
Intent intent = new Intent(content,Service.class);
bindService(intent,connection,BIND_AUTO_CREATE);
```
<!--more-->
#### Service启动过程

Service的启动时从ContextWrapper的startService开始的：
``` java
@Override
    public ComponentName startService(Intent service) {
        return mBase.startService(service);
    }
```
mBase是Context的实现类ContextImpl，在Activity启动的时候会通过attach方法关联一个ContextImpl，这个ContextImpl就是上面的mBase，从ContextWrapper的实现来看，期大部分的实现都是通过mBase来实现的，这是一种典型的桥接模式。在ContextImpl的startService方法中又调用了startServiceCommon。
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
在startServiceCommon中通过getService这个对象来启动一个服务，这个对象就是AMS，需要注意的是，在上述代码中通过AMS来启动服务的过程是一个跨进程调用。AMS的startService如下：
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
AMS会通过mServices来完成service的启动过程，mServices的对象类型是ActiveServices，ActiveServices是一个辅助AMS进行Service管理的类，包括Service的启动、绑定和停止。在startService方法的尾部会调用`startServiceInnerLocked`方法
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
上述代码中ServiceRecord是描述一个Service记录，ServiceRecord一直贯穿整个Service流程，startServiceInnerLocked并没有完成启动Service的完整流程，而是将后续的过程交给了bringUpServiceLocked，在该方法中又调用了realStartServiceLocked方法:
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
在realStartServiceLocked方法中，首先通过app.thread的scheduleCreateService方法来创建Service对象并调用其onCreate,接着再通过sendServiceArgsLocked方法来调用Service的其他方法，比如onStartCommand,这两个过程均是进程间通信。app.thread对 象是IApplicationThread类型，它实际上是一个Binder,它的具体实现是ApplicationThread和ApplicationThreadNative。由于pplicationThread继承了ApplicationThreadNative,因此只需要看ApplicationThread对Service启动过程的处理即可，这对应着它的scheduleCreateService方法，如下所示：
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
通过发送消息给Handler H来完成的。H会接收这个CREATE_ SERVICE消息并通过ActivityThread的handleCreateService方法来完成Service的最终启动，
handleCreateService的源码如下所示：
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
handleCreateService主要完成了以下几件事：
首先通过类加载器创建了Service实例，然后创建Application对象对象并调用