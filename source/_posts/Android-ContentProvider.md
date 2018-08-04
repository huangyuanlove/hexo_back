---
title: Android ContentProvider
date: 2018-08-02 22:06:07
tags: [Android]
keywords: ContentProvider
---
《Android开发艺术探索》9.5章

系统预置了许多`ContentProvider`，比如通讯录信息、日程表信息等，要跨进程访问这些信息，只需要通过`ContentResolver`的query、update、insert和delete方法即可。虽然`ContentProvider`的底层
实现是`Binder`，但是它的使用过程要比`AIDL`简单许多，这是因为系统已经为我们做了封装，使得我们无须关心底层细节即可轻松实现IPC。系统预置了许多`ContentProvider`，比如通讯录信息、日程表信息等，要跨进程访问这些信息，只需要通过`ContentResolver`的query、update、insert和delete方法即可。
<!--more-->

#### 使用ContentResolver读取联系人

``` java
private ArrayList<HashMap<String, String>> readContact() {

    String NUM = ContactsContract.CommonDataKinds.Phone.NUMBER;
    String NAME = ContactsContract.CommonDataKinds.Phone.DISPLAY_NAME;
    Uri uri = ContactsContract.CommonDataKinds.Phone.CONTENT_URI;

    ArrayList<HashMap<String, String>> list = new ArrayList<HashMap<String, String>>();
    ContentResolver cr = getContentResolver();
    Cursor cursor = cr.query(uri,new String[]{NUM,NAME},null,null,null);
    while (cursor.moveToNext()){
        String name = cursor.getString(cursor.getColumnIndex(NAME));
        String phone = cursor.getString(cursor.getColumnIndex(NUM));
        HashMap<String,String> contact = new HashMap<>();
        contact.put("name",name);
        contact.put("phone",phone);
        list.add(contact);
    }
    return list;
```

#### 工作过程
`ContentProvider`是一种内容共享型组件，它通过`Binder`向其他组件乃至其他应用提供数据。当`ContentProvider`所在的进程启动时，`ContentProvider`会同时启动并被发布到AMS中。需要注意的是，这个时候`ContentProvider`的`onCreate`要先于`Application`的`onCreate`而执行。
当一个应用启动时，入口方法为`ActivityThread`的`main`方法，`main`方法是一个静态方法，在`main`方法中会创建`ActivityThread`的实例并创建主线程的消息队列，然后在`ActivityThread`的`attach`方法中会远程调用`AMS`的`attachApplication`方法并将`ApplicationThread`对象提供给`AMS`。`ApplicationThread`是一个`Binder`对象，它的`Binder`接口是`IApplicationThread`，它主要用于`ActivityThread`和`AMS`之间的通信，这一点在前面多次提到。在`AMS`的`attachApplication`方法中，会调用`ApplicationThread`的`bindApplication`方法，注意这个过程同样是跨进程完成的，`bindApplication`的逻辑会经过`ActivityThread`中的`mH  Handler`切换到`ActivityThread`中去执行，具体的方法是`handleBindApplication`。在`handleBindApplication`方法中，`ActivityThread`会创建`Application`对象并加载`ContentProvider`。需要注意的是，`ActivityThread`会先加载`ContentProvider`，然后再调用`Application`的`onCreate`方法。
这就是`ContentProvider`的启动过程，`ContentProvider`启动后，外界就可以通过它所提供的增删改查这四个接口来操作`ContentProvider`中的数据源，即insert、delete、update和query四个方法。这四个方法都是通过`Binder`来调用的，外界无法直接访问`ContentProvider`，它只能通过AMS根据Uri来获取对应的`ContentProvider`的`Binder`接口`IConentProvider`，然后再通过`IConentProvider`来访问`ContentProvider`中的数据源。
一般来说，`ContentProvide`r都应该是单实例的。`ContentProvider`到底是不是单实例，这是由它的`android:multiprocess`属性来决定的，当`android:multiprocess`为`false`时，`ContentProvider`是单实例，这也是默认值；当`android:multiprocess`为`true`时，`ContentProvider`为多实例，这个时候在每个调用者的进程中都存在一个`ContentProvider`对象。
访问`ContentProvide`r需要通过`ContentResolver`，`ContentResolver`是一个抽象类，通过`Context`的`getContentResolver`方法获取的实际上是`ApplicationContentResolver`对象，`ApplicationContentResolver`类继承了`ContentResolver`并实现了`ContentResolver`中的抽象方法。当`ContentProvider`所在的进程未启动时，第一次访问它时就会触发`ContentProvider`的创建，当然这也伴随着`ContentProvider`所在进程的启动。通过`ContentProvider`的四个方法的任何一个都可以触发`ContentProvider`的启动过程，这里选择`query`方法。`ContentProvider`的`query`方法中，首先会获取`IContentProvider`对象，不管是通过`acquireUnstableProvider`方法还是直接通过`acquireProvider`方法，它们的本质都是一样的，最终都是通过`acquireProvider`方法来获取`ContentProvider`。下面是`ApplicationContentResolver`的`acquireProvider`方法的具体实现:
``` java
 @Override
    protected IContentProvider acquireProvider(Context context, String auth) {
        return mMainThread.acquireProvider(context,
                ContentProvider.getAuthorityWithoutUserId(auth),
                resolveUserIdFromAuthority(auth), true);
    }
```
`ApplicationContentResolver`的`acquireProvider`方法并没有处理任何逻辑，它直接调用了`ActivityThread`的`acquireProvider`方法，`ActivityThread`的`acquireProvider`方法的源码如下:
``` java
public final IContentProvider acquireProvider(
            Context c, String auth, int userId, boolean stable) {
    final IContentProvider provider = acquireExistingProvider(c, auth, userId, stable);
    if (provider != null) {
        return provider;
    }

    // There is a possible race here.  Another thread may try to acquire
    // the same provider at the same time.  When this happens, we want to ensure
    // that the first one wins.
    // Note that we cannot hold the lock while acquiring and installing the
    // provider since it might take a long time to run and it could also potentially
    // be re-entrant in the case where the provider is in the same process.
    ContentProviderHolder holder = null;
    try {
        holder = ActivityManager.getService().getContentProvider(
                getApplicationThread(), auth, userId, stable);
    } catch (RemoteException ex) {
        throw ex.rethrowFromSystemServer();
    }
    if (holder == null) {
        Slog.e(TAG, "Failed to find provider info for " + auth);
        return null;
    }

    // Install provider will increment the reference count for us, and break
    // any ties in the race.
    holder = installProvider(c, holder, holder.info,
            true /*noisy*/, holder.noReleaseNeeded, stable);
    return holder.provider;
}
```
上面的代码首先会从`ActivityThread`中查找是否已经存在目标`ContentProvider`了，如果存在就直接返回。`ActivityThread`中通过`mProviderMap`来存储已经启动的`ContentProvider`对象，`mProviderMap`的声明如下所示:
``` java
inal ArrayMap<providerKey,ProviderClientRecord> mProviderMap = new ArrayMap<providerKey,ProviderClientRecord>();
```
如果目前`ContentProvider`没有启动，那么就发送一个进程间请求给AMS让其启动目标`ContentProvider`，最后再通过`installProvider`方法来修改引用计数。`ContentProvider`被启动时会伴随着进程的启动，在AMS中，首先会启动`ContentProvider`所在的进程，然后再启动`ContentProvider`。启动进程是由AMS的`startProcessLocked`方法来完成的，其内部主要是通过`Process`的`start`方法来完成一个新进程的启动，新进程启动后其入口方法为`ActivityThread`的main方法，如下所示:
``` java
public static void main(String[] args) {
    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");

    // CloseGuard defaults to true and can be quite spammy.  We
    // disable it here, but selectively enable it later (via
    // StrictMode) on debug builds, but using DropBox, not logs.
    CloseGuard.setEnabled(false);

    Environment.initForCurrentUser();

    // Set the reporter for event logging in libcore
    EventLogger.setReporter(new EventLoggingReporter());

    // Make sure TrustedCertificateStore looks in the right place for CA certificates
    final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
    TrustedCertificateStore.setDefaultUserDirectory(configDir);

    Process.setArgV0("<pre-initialized>");

    Looper.prepareMainLooper();

    ActivityThread thread = new ActivityThread();
    thread.attach(false);

    if (sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();
    }

    if (false) {
        Looper.myLooper().setMessageLogging(new
                LogPrinter(Log.DEBUG, "ActivityThread"));
    }

    // End of event ActivityThreadMain.
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    Looper.loop();

    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```
可以看到，`ActivityThread`的`main`方法是一个静态方法，在它内部首先会创建`ActivityThread`的实例并调用`attach`方法来进行一系列初始化，接着就开始进行消息循环了。`ActivityThread`的`attach`方法会将`ApplicationThread`对象通过`AMS`的`attachApplication`方法跨进程传递给AMS，最终AMS会完成`ContentProvider`的创建过程，AMS的`attachApplication`方法调用了`attachApplicationLocked`方法，`attachApplicationLocked`中又调用了`ApplicationThread`的`bindApplication`，注意这个过程也是进程间调用，
``` java
try {
    mgr.attachApplication(mAppThread);
} catch (RemoteException ex) {
    throw ex.rethrowFromSystemServer();
}

@Override
    public final void attachApplication(IApplicationThread thread) {
        synchronized (this) {
            int callingPid = Binder.getCallingPid();
            final long origId = Binder.clearCallingIdentity();
            attachApplicationLocked(thread, callingPid);
            Binder.restoreCallingIdentity(origId);
        }
    }
private final boolean attachApplicationLocked(IApplicationThread thread,int pid) {
    ......
if (app.instr != null) {
                thread.bindApplication(processName, appInfo, providers,
                        app.instr.mClass,
                        profilerInfo, app.instr.mArguments,
                        app.instr.mWatcher,
                        app.instr.mUiAutomationConnection, testMode,
                        mBinderTransactionTrackingEnabled, enableTrackAllocation,
                        isRestrictedBackupMode || !normalMode, app.persistent,
                        new Configuration(getGlobalConfiguration()), app.compat,
                        getCommonServicesLocked(app.isolated),
                        mCoreSettingsObserver.getCoreSettingsLocked(),
                        buildSerial);
            } else {
                thread.bindApplication(processName, appInfo, providers, null, profilerInfo,
                        null, null, null, testMode,
                        mBinderTransactionTrackingEnabled, enableTrackAllocation,
                        isRestrictedBackupMode || !normalMode, app.persistent,
                        new Configuration(getGlobalConfiguration()), app.compat,
                        getCommonServicesLocked(app.isolated),
                        mCoreSettingsObserver.getCoreSettingsLocked(),
                        buildSerial);
            }
            ......
}
```
`ActivityThread`的`bindApplication`会发送一个`BIND_APPLICATION`类型的消息给`mH`，`mH`是一个`Handler`，它收到消息后会调用`ActivityThread`的`handleBindApplication`方法，`bindApplication`发送消息的过程如下所示:
``` java
AppBindData data = new AppBindData();
            data.processName = processName;
            data.appInfo = appInfo;
            data.providers = providers;
            data.instrumentationName = instrumentationName;
            data.instrumentationArgs = instrumentationArgs;
            data.instrumentationWatcher = instrumentationWatcher;
            data.instrumentationUiAutomationConnection = instrumentationUiConnection;
            data.debugMode = debugMode;
            data.enableBinderTracking = enableBinderTracking;
            data.trackAllocation = trackAllocation;
            data.restrictedBackupMode = isRestrictedBackupMode;
            data.persistent = persistent;
            data.config = config;
            data.compatInfo = compatInfo;
            data.initProfilerInfo = profilerInfo;
            data.buildSerial = buildSerial;
            sendMessage(H.BIND_APPLICATION, data);
```
`ActivityThread`的`handleBindApplication`则完成了`Application`的创建以及`ContentProvider`的创建，可以分为如下四个步骤:

** 创建ContextImpl和Instrumentation **
``` java
final ContextImpl appContext = ContextImpl.createAppContext(this, data.info);
updateLocaleListFromAppContext(appContext, mResourcesManager.getConfiguration().getLocales());
 try {
    final ClassLoader cl = instrContext.getClassLoader();
    mInstrumentation = (Instrumentation)
        cl.loadClass(data.instrumentationName.getClassName()).newInstance();
} catch (Exception e) {
    throw new RuntimeException(
        "Unable to instantiate instrumentation "
        + data.instrumentationName + ": " + e.toString(), e);
}
final ComponentName component = new ComponentName(ii.packageName, ii.name);
mInstrumentation.init(this, instrContext, appContext, component, data.instrumentationWatcher, data.instrumentationUiAutomationConnection);
```

** 创建Application对象 **
``` java
// If the app is being launched for full backup or restore, bring it up in
// a restricted environment with the base application class.
app = data.info.makeApplication(data.restrictedBackupMode, null);
mInitialApplication = app;
```

** 启动当前进程的ContentProvider并调用其onCreate方法 **
``` java
// don't bring up providers in restricted mode; they may depend on the
// app's custom Application class
if (!data.restrictedBackupMode) {
    if (!ArrayUtils.isEmpty(data.providers)) {
        installContentProviders(app, data.providers);
        // For process that contains content providers, we want to
        // ensure that the JIT is enabled "at some point".
        mH.sendEmptyMessageDelayed(H.ENABLE_JIT, 10*1000);
    }
}
```
`installContentProviders`完成了`ContentProvider`的启动工作，它的实现如下所示。首先会遍历当前进程的`ProviderInfo`的列表并一一调用调用`installProvider`方法来启动它们，接着将已经启动的`ContentProvider`发布到AMS中，AMS会把它们存储在`ProviderMap`中，这样一来外部调用者就可以直接从AMS中获取`ContentProvider`了。
``` java
 private void installContentProviders(
            Context context, List<ProviderInfo> providers) {
        final ArrayList<ContentProviderHolder> results = new ArrayList<>();

        for (ProviderInfo cpi : providers) {
            if (DEBUG_PROVIDER) {
                StringBuilder buf = new StringBuilder(128);
                buf.append("Pub ");
                buf.append(cpi.authority);
                buf.append(": ");
                buf.append(cpi.name);
                Log.i(TAG, buf.toString());
            }
            ContentProviderHolder cph = installProvider(context, null, cpi,
                    false /*noisy*/, true /*noReleaseNeeded*/, true /*stable*/);
            if (cph != null) {
                cph.noReleaseNeeded = true;
                results.add(cph);
            }
        }

        try {
            ActivityManager.getService().publishContentProviders(
                getApplicationThread(), results);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
    }
```
下面看一下`ContentProvider`对象的创建过程，在`installProvider`方法中有下面一段代码，其通过类加载器完成了`ContentProvider`对象的创建:
``` java
final java.lang.ClassLoader cl = c.getClassLoader();
localProvider = (ContentProvider)cl.loadClass(info.name).newInstance();
provider = localProvider.getIContentProvider();
// XXX Need to create the correct context for this provider.
localProvider.attachInfo(c, info);
```
在上述代码中，除了完成`ContentProvider`对象的创建，还会通过`ContentProvider`的`attachInfo`方法来调用它的`onCreate`方法:
``` java
private void attachInfo(Context context, ProviderInfo info, boolean testing) {
    mNoPerms = testing;

    /*
        * Only allow it to be set once, so after the content service gives
        * this to us clients can't change it.
        */
    if (mContext == null) {
        mContext = context;
        if (context != null) {
            mTransport.mAppOpsManager = (AppOpsManager) context.getSystemService(
                    Context.APP_OPS_SERVICE);
        }
        mMyUid = Process.myUid();
        if (info != null) {
            setReadPermission(info.readPermission);
            setWritePermission(info.writePermission);
            setPathPermissions(info.pathPermissions);
            mExported = info.exported;
            mSingleUser = (info.flags & ProviderInfo.FLAG_SINGLE_USER) != 0;
            setAuthorities(info.authority);
        }
        ContentProvider.this.onCreate();
    }
}
```
到此为止，`ContentProvider`已经被创建并且其`onCreate`方法也已经被调用，这意味着`ContentProvider`已经启动完成了。

** 调用Application的onCreate方法 **
``` java
// Do this after providers, since instrumentation tests generally start their
// test thread at this point, and we don't want that racing.
try {
    mInstrumentation.onCreate(data.instrumentationArgs);
}
catch (Exception e) {
    throw new RuntimeException(
        "Exception thrown in onCreate() of "
        + data.instrumentationName + ": " + e.toString(), e);
}
try {
    mInstrumentation.callApplicationOnCreate(app);
} catch (Exception e) {
    if (!mInstrumentation.onException(app, e)) {
        throw new RuntimeException(
            "Unable to create application " + app.getClass().getName()
            + ": " + e.toString(), e);
    }
}
```
经过上面的四个步骤，`ContentProvider`已经成功启动，并且其所在进程的`Application`也已经启动，这意味着`ContentProvider`所在的进程已经完成了整个的启动过程，然后其他应用就可以通过AMS来访问这个`ContentProvider`了。拿到了`ContentProvider`以后，就可以通过它所提供的接口方法来访问它了。需要注意的是，这里的`ContentProvider`并不是原始的`ContentProvider`，而是`ContentProvider`的`Binder`类型的对象`IContentProvider`，`IContentProvider`的具体实现是`ContentProviderNative`和`ContentProvider.Transport`，其中`ContentProvider.Transport`继承了`ContentProviderNative`。这里仍然选择`query`方法，首先其他应用会通过AMS获取到`ContentProvider`的`Binder`对象即`IContentProvider`，而`IContentProvider`的实现者实际上是`ContentProvider.Transport`。因此其他应用调用`IContentProvider`的query方法时最终会以进程间通信的方式调用到`ContentProvider.Transport`的query方法，它的实现如下所示:
``` java
@Override
public Cursor query(String callingPkg, Uri uri, @Nullable String[] projection,
        @Nullable Bundle queryArgs, @Nullable ICancellationSignal cancellationSignal) {
    validateIncomingUri(uri);
    uri = maybeGetUriWithoutUserId(uri);
    if (enforceReadPermission(callingPkg, uri, null) != AppOpsManager.MODE_ALLOWED) {
        // The caller has no access to the data, so return an empty cursor with
        // the columns in the requested order. The caller may ask for an invalid
        // column and we would not catch that but this is not a problem in practice.
        // We do not call ContentProvider#query with a modified where clause since
        // the implementation is not guaranteed to be backed by a SQL database, hence
        // it may not handle properly the tautology where clause we would have created.
        if (projection != null) {
            return new MatrixCursor(projection, 0);
        }

        // Null projection means all columns but we have no idea which they are.
        // However, the caller may be expecting to access them my index. Hence,
        // we have to execute the query as if allowed to get a cursor with the
        // columns. We then use the column names to return an empty cursor.
        Cursor cursor = ContentProvider.this.query(
                uri, projection, queryArgs,
                CancellationSignal.fromTransport(cancellationSignal));
        if (cursor == null) {
            return null;
        }

        // Return an empty cursor for all columns.
        return new MatrixCursor(cursor.getColumnNames(), 0);
    }
    final String original = setCallingPackage(callingPkg);
    try {
        return ContentProvider.this.query(
                uri, projection, queryArgs,
                CancellationSignal.fromTransport(cancellationSignal));
    } finally {
        setCallingPackage(original);
    }
}
```
很显然，`ContentProvider.Transport`的`query`方法调用了`ContentProvider`的`query`方法，`query`方法的执行结果再通过`Binder`返回给调用者，这样一来整个调用过程就完成了。除了query方法，insert、delete和update方法也是类似的。

----
以上