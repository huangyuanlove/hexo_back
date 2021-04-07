---
title: PMS及apk安装过程
tags: [Android]
date: 2021-04-05 13:35:36
keywords: [PMS,apk安装过程,apk解析]
---

先从如何使用代码安装一个apk开始。

在7.0之前，我们可以直接指定apk的路径进行安装

``` java
Intent intent = new Intent(Intent.Action_View);
String filepath = "/sdcard/a.apk";
intent.setDataAndType(Uri.parse("file://" + filepath),"application/vnd.android.package-archive");
startActivity(intent);
```

在7.0及以后，需要使用FileProvider进行安装

``` java
File apk = new File(...);
Intent intent = new Intent(Intent.ACTION_VIEW);
intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
Uri uri = FileProvider.getUriForFile(this, "com.example.demo.fileprovider", apk);
intent.setDataAndType(uri, "application/vnd.android.package-archive");
startActivity(intent);
```

不管是哪个版本，我们都需要调用`intent.setDataAndType`方法，我们在aosp源码中找到了对应的Activity：在`packages/apps/PackageInstaller`文件夹下

``` xml
<activity android:name=".InstallStart"
          android:exported="true"
          android:excludeFromRecents="true">
    <intent-filter android:priority="1">
        <action android:name="android.intent.action.VIEW" />
        <action android:name="android.intent.action.INSTALL_PACKAGE" />
        <category android:name="android.intent.category.DEFAULT" />
        <data android:scheme="file" />
        <data android:scheme="content" />
        <data android:mimeType="application/vnd.android.package-archive" />
    </intent-filter>
    <intent-filter android:priority="1">
        <action android:name="android.intent.action.INSTALL_PACKAGE" />
        <category android:name="android.intent.category.DEFAULT" />
        <data android:scheme="file" />
        <data android:scheme="package" />
        <data android:scheme="content" />
    </intent-filter>
    <intent-filter android:priority="1">
        <action android:name="android.content.pm.action.CONFIRM_PERMISSIONS" />
        <category android:name="android.intent.category.DEFAULT" />
    </intent-filter>
</activity>
```

<!--more-->

###  PackageInstaller准备工作



#### com.android.packageinstaller.InstallStart

`InstallStart`继承自Activity，先看注释：

```
/**
 * Select which activity is the first visible activity of the installation and forward the intent to
 * it.
 */
```

很清楚了，判断需要打开哪个页面。

我们来看`onCreate`方法中的关键地方

``` java
Intent nextActivity = new Intent(intent);
nextActivity.setFlags(Intent.FLAG_ACTIVITY_FORWARD_RESULT);

// The the installation source as the nextActivity thinks this activity is the source, hence
// set the originating UID and sourceInfo explicitly
nextActivity.putExtra(PackageInstallerActivity.EXTRA_CALLING_PACKAGE, callingPackage);
nextActivity.putExtra(PackageInstallerActivity.EXTRA_ORIGINAL_SOURCE_INFO, sourceInfo);
nextActivity.putExtra(Intent.EXTRA_ORIGINATING_UID, originatingUid);

if (PackageInstaller.ACTION_CONFIRM_PERMISSIONS.equals(intent.getAction())) {
    nextActivity.setClass(this, PackageInstallerActivity.class);
} else {
    Uri packageUri = intent.getData();

    if (packageUri != null && (packageUri.getScheme().equals(ContentResolver.SCHEME_FILE)
                               || packageUri.getScheme().equals(ContentResolver.SCHEME_CONTENT))) {
        // Copy file to prevent it from being changed underneath this process
        nextActivity.setClass(this, InstallStaging.class);
    } else if (packageUri != null && packageUri.getScheme().equals(
        PackageInstallerActivity.SCHEME_PACKAGE)) {
        nextActivity.setClass(this, PackageInstallerActivity.class);
    } else {
        Intent result = new Intent();
        result.putExtra(Intent.EXTRA_INSTALL_RESULT,
                        PackageManager.INSTALL_FAILED_INVALID_URI);
        setResult(RESULT_FIRST_USER, result);

        nextActivity = null;
    }
}

if (nextActivity != null) {
    startActivity(nextActivity);
}
finish();
```

在7.0之后的版本上，由于使用FIleProvider，会隐藏共享文件的真实路径，并将路径转换成:Uri路径，这样就会跳转到`InstallStaging`这个类

#### com.android.packageinstaller.InstallStaging

同样先看顶部注释

```
/**
 * If a package gets installed from an content URI this step loads the package and turns it into
 * and installation from a file. Then it re-starts the installation as usual.
 */
```

如果是从Uri安装的，会先转化成文件，然后再进行安装

我们找到对应的方法，是在onResume中

``` java
    @Override
    protected void onResume() {
        super.onResume();

        // This is the first onResume in a single life of the activity
        if (mStagingTask == null) {
            // File does not exist, or became invalid
            if (mStagedFile == null) {
                // Create file delayed to be able to show error
                try {
                    mStagedFile = TemporaryFileManager.getStagedFile(this);
                } catch (IOException e) {
                    showError();
                    return;
                }
            }

            mStagingTask = new StagingAsyncTask();
            mStagingTask.execute(getIntent().getData());
        }
    }

    private final class StagingAsyncTask extends AsyncTask<Uri, Void, Boolean> {
        @Override
        protected Boolean doInBackground(Uri... params) {
            if (params == null || params.length <= 0) {
                return false;
            }
            Uri packageUri = params[0];
            try (InputStream in = getContentResolver().openInputStream(packageUri)) {
                // Despite the comments in ContentResolver#openInputStream the returned stream can
                // be null.
                if (in == null) {
                    return false;
                }

                try (OutputStream out = new FileOutputStream(mStagedFile)) {
                    byte[] buffer = new byte[1024 * 1024];
                    int bytesRead;
                    while ((bytesRead = in.read(buffer)) >= 0) {
                        // Be nice and respond to a cancellation
                        if (isCancelled()) {
                            return false;
                        }
                        out.write(buffer, 0, bytesRead);
                    }
                }
            } catch (IOException | SecurityException | IllegalStateException e) {
                Log.w(LOG_TAG, "Error staging apk from content URI", e);
                return false;
            }
            return true;
        }

        @Override
        protected void onPostExecute(Boolean success) {
            if (success) {
                // Now start the installation again from a file
                Intent installIntent = new Intent(getIntent());
                installIntent.setClass(InstallStaging.this, DeleteStagedFileOnResult.class);
                installIntent.setData(Uri.fromFile(mStagedFile));

                if (installIntent.getBooleanExtra(Intent.EXTRA_RETURN_RESULT, false)) {
                    installIntent.addFlags(Intent.FLAG_ACTIVITY_FORWARD_RESULT);
                }

                installIntent.addFlags(Intent.FLAG_ACTIVITY_NO_ANIMATION);
                startActivity(installIntent);

                InstallStaging.this.finish();
            } else {
                showError();
            }
        }
    }
```

这里创建了`StagingAsyncTask`, 将uri(最开始传入的apk文件的uri，也是task中的packageUri)中的内容写入到mStagedFile中；如果写入成功，则跳转到`DeleteStagedFileOnResult`页面

####  com.android.packageinstaller.DeleteStagedFileOnResult

先看顶部注释

```
/**
 * Trampoline activity. Calls PackageInstallerActivity and deletes staged install file onResult.
 */
```

作为中间层的页面，打开PackageInstallerActivity，并且在返回的时候删除暂存文件。没啥好说的，直接到PackageInstallerActivity

#### com.android.packageinstaller.PackageInstallerActivity

```
/**
 * This activity is launched when a new application is installed via side loading
 * The package is first parsed and the user is notified of parse errors via a dialog.
 * If the package is successfully parsed, the user is notified to turn on the install unknown
 * applications setting. A memory check is made at this point and the user is notified of out
 * of memory conditions if any. If the package is already existing on the device,
 * a confirmation dialog (to replace the existing package) is presented to the user.
 * Based on the user response the package is then installed by launching InstallAppConfirm
 * sub activity. All state transitions are handled in this activity
 */
```

粗略的描述了应用程序的安装过程：先对安装包进行解析，解析失败则弹窗通知；解析成功后，通知用户打开"安装未知应用程序"设置，进行内容检查，如果内存不足，则通知用户；如果设备上已经存在该应用，则通知用户进行替换，然后根据用户响应，通过启动InstallAppConfirm来安装软件包。

我们来看onCreate方法

``` java
    @Override
    protected void onCreate(Bundle icicle) {
        super.onCreate(null);

        if (icicle != null) {
            mAllowUnknownSources = icicle.getBoolean(ALLOW_UNKNOWN_SOURCES_KEY);
        }

        mPm = getPackageManager();
        mIpm = AppGlobals.getPackageManager();
        mAppOpsManager = (AppOpsManager) getSystemService(Context.APP_OPS_SERVICE);
        mInstaller = mPm.getPackageInstaller();
        mUserManager = (UserManager) getSystemService(Context.USER_SERVICE);
		。
        。
        。
        boolean wasSetUp = processPackageUri(packageUri);
        if (!wasSetUp) {
            return;
        }

        // load dummy layout with OK button disabled until we override this layout in
        // startInstallConfirm
        bindUi(R.layout.install_confirm, false);
        checkIfAllowedAndInitiateInstall();
    }
```

一开始初始化需要的各种对象，PackageManager、IPackageManager、IPackageManager、UserManager、PackageInstaller等；

瞄一眼`processPackageUri`方法：根据Uri拿到scheme,然后根据scheme类型拿到packageInfo;如果scheme既不是`package`也不是`file`，则抛出`IllegalArgumentException`异常。

`bindUi`是设置页面按钮的点击事件。

看下`checkIfAllowedAndInitiateInstall`方法：如果允许安装未知来源或者该应用不是未知来源，则调用`initiateInstall`方法进行安装；在`initiateInstall`方法中，获取包名信息，进入判断设备上是否已经安装该应用流程，最后调用`startInstallConfirm`初始化确认安装界面，列出应用所需权限信息



总结一下这些步骤：

* 根据Uri中的scheme不同，跳转到不同页面
* InstallStart将content协议转化为file协议，跳转到InstallStaging，然后跳转到PackageInstallerActivity
* 在PackageInstallerActivity，对协议进行处理，解析文件得到PackageInfo
* 对未知来源apk进行处理，初始化页面



###  PackageInstaller安装apk

当我们点击安装页面中确认按钮时，调用`onClick`方法，接着调用`startInstall`打开`InstallInstalling`页面进行安装。老规矩，先看顶部注释

```
/**
 * Send package to the package manager and handle results from package manager. Once the
 * installation succeeds, start {@link InstallSuccess} or {@link InstallFailed}.
 * <p>This has two phases: First send the data to the package manager, then wait until the package
 * manager processed the result.</p>
 */
```

主要用于向包管理器发送包的信息，并处理回调，安装成功则打开`InstallSuccess`,失败则打开`InstallFailed`

#### com.android.packageinstaller.InstallInstalling

看下onCreate做了啥：

``` java
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        setContentView(R.layout.install_installing);

        ApplicationInfo appInfo = getIntent()
                .getParcelableExtra(PackageUtil.INTENT_ATTR_APPLICATION_INFO);
        mPackageURI = getIntent().getData();

        if ("package".equals(mPackageURI.getScheme())) {
            try {
                getPackageManager().installExistingPackage(appInfo.packageName);
                launchSuccess();
            } catch (PackageManager.NameNotFoundException e) {
                launchFailure(PackageManager.INSTALL_FAILED_INTERNAL_ERROR, null);
            }
        } else {
            final File sourceFile = new File(mPackageURI.getPath());
            PackageUtil.initSnippetForNewApp(this, PackageUtil.getAppSnippet(this, appInfo,
                    sourceFile), R.id.app_snippet);

            if (savedInstanceState != null) {//----1------
                mSessionId = savedInstanceState.getInt(SESSION_ID);
                mInstallId = savedInstanceState.getInt(INSTALL_ID);

                // Reregister for result; might instantly call back if result was delivered while
                // activity was destroyed
                try {
                    InstallEventReceiver.addObserver(this, mInstallId,
                            this::launchFinishBasedOnResult);
                } catch (EventResultPersister.OutOfIdsException e) {
                    // Does not happen
                }
            } else {
                PackageInstaller.SessionParams params = new PackageInstaller.SessionParams(
                        PackageInstaller.SessionParams.MODE_FULL_INSTALL);
                params.installFlags = PackageManager.INSTALL_FULL_APP;
                params.referrerUri = getIntent().getParcelableExtra(Intent.EXTRA_REFERRER);
                params.originatingUri = getIntent()
                        .getParcelableExtra(Intent.EXTRA_ORIGINATING_URI);
                params.originatingUid = getIntent().getIntExtra(Intent.EXTRA_ORIGINATING_UID,
                        UID_UNKNOWN);
                params.installerPackageName =
                        getIntent().getStringExtra(Intent.EXTRA_INSTALLER_PACKAGE_NAME);

                File file = new File(mPackageURI.getPath());
                try {
                    PackageParser.PackageLite pkg = PackageParser.parsePackageLite(file, 0);
                    params.setAppPackageName(pkg.packageName);
                    params.setInstallLocation(pkg.installLocation);
                    params.setSize(
                            PackageHelper.calculateInstalledSize(pkg, false, params.abiOverride));
                } catch (PackageParser.PackageParserException e) {
                    Log.e(LOG_TAG, "Cannot parse package " + file + ". Assuming defaults.");
                    Log.e(LOG_TAG,
                            "Cannot calculate installed size " + file + ". Try only apk size.");
                    params.setSize(file.length());
                } catch (IOException e) {
                    Log.e(LOG_TAG,
                            "Cannot calculate installed size " + file + ". Try only apk size.");
                    params.setSize(file.length());
                }

                try {//-------2-------
                    mInstallId = InstallEventReceiver
                            .addObserver(this, EventResultPersister.GENERATE_NEW_ID,
                                    this::launchFinishBasedOnResult);
                } catch (EventResultPersister.OutOfIdsException e) {
                    launchFailure(PackageManager.INSTALL_FAILED_INTERNAL_ERROR, null);
                }

                try {//-------3-------
                    mSessionId = getPackageManager().getPackageInstaller().createSession(params);
                } catch (IOException e) {
                    launchFailure(PackageManager.INSTALL_FAILED_INTERNAL_ERROR, null);
                }
            }

            mSessionCallback = new InstallSessionCallback();
        }
    }
```

在onCreate方法中会对`package`和`content`协议的Uri进袭姑娘处理，我们关注一下content协议的uri处理部分:

首先判断savedInstanceState是否为空，不为空则从中获取mSessionId和mInstallId，然后向InstallEventReceiver注册一个观察者。

如果为空，则构建一个 PackageInstaller.SessionParams对象，在注释2处同样注册了一个观察者，然后在注释3处创建并返回sessionId

* 这里创建mSessionId时，在createSession方法内部会通过`IPackageInstaller`与`PackageInstallerService`进行进程间通信，最终调用的是`PackageInstaller`的`createSession`方法来创建并返回的sessionId

* 这里注册的观察者是`launchFinishBasedOnResult`方法，根据安装结果不同，跳转到不同的页面(安装成功、安装失败)
* 这里的InstallEventReceiver是继承自BroadcastReceiver的广播接收器，可以看作是一个中间层，真正保存观察者的类是EventResultPersister，对于`EventResultPersister`注释是这样的：**Persists results of events and calls back observers when a matching result arrives.**

我们接着看InstallInstalling的onResume方法

#### com.android.packageinstaller.InstallInstalling

``` java
@Override
protected void onResume() {
    super.onResume();

    // This is the first onResume in a single life of the activity
    if (mInstallingTask == null) {
        PackageInstaller installer = getPackageManager().getPackageInstaller();
        PackageInstaller.SessionInfo sessionInfo = installer.getSessionInfo(mSessionId);

        if (sessionInfo != null && !sessionInfo.isActive()) {
            mInstallingTask = new InstallingAsyncTask();
            mInstallingTask.execute();
        } else {
            // we will receive a broadcast when the install is finished
            mCancelButton.setEnabled(false);
            setFinishOnTouchOutside(false);
        }
    }
}
    /**
     * Send the package to the package installer and then register a event result observer that
     * will call {@link #launchFinishBasedOnResult(int, int, String)}
     */
    private final class InstallingAsyncTask extends AsyncTask<Void, Void,
            PackageInstaller.Session> {
        volatile boolean isDone;

        @Override
        protected PackageInstaller.Session doInBackground(Void... params) {
            PackageInstaller.Session session;
            try {
                session = getPackageManager().getPackageInstaller().openSession(mSessionId);
            } catch (IOException e) {
                return null;
            }

            session.setStagingProgress(0);

            try {
                File file = new File(mPackageURI.getPath());

                try (InputStream in = new FileInputStream(file)) {
                    long sizeBytes = file.length();
                    try (OutputStream out = session
                            .openWrite("PackageInstaller", 0, sizeBytes)) {
                        byte[] buffer = new byte[1024 * 1024];
                        while (true) {
                            int numRead = in.read(buffer);

                            if (numRead == -1) {
                                session.fsync(out);
                                break;
                            }

                            if (isCancelled()) {
                                session.close();
                                break;
                            }

                            out.write(buffer, 0, numRead);
                            if (sizeBytes > 0) {
                                float fraction = ((float) numRead / (float) sizeBytes);
                                session.addProgress(fraction);
                            }
                        }
                    }
                }

                return session;
            } catch (IOException | SecurityException e) {
                Log.e(LOG_TAG, "Could not write package", e);

                session.close();

                return null;
            } finally {
                synchronized (this) {
                    isDone = true;
                    notifyAll();
                }
            }
        }

        @Override
        protected void onPostExecute(PackageInstaller.Session session) {
            if (session != null) {
                Intent broadcastIntent = new Intent(BROADCAST_ACTION);
                broadcastIntent.setFlags(Intent.FLAG_RECEIVER_FOREGROUND);
                broadcastIntent.setPackage(
                        getPackageManager().getPermissionControllerPackageName());
                broadcastIntent.putExtra(EventResultPersister.EXTRA_ID, mInstallId);

                PendingIntent pendingIntent = PendingIntent.getBroadcast(
                        InstallInstalling.this,
                        mInstallId,
                        broadcastIntent,
                        PendingIntent.FLAG_UPDATE_CURRENT);

                session.commit(pendingIntent.getIntentSender());
                mCancelButton.setEnabled(false);
                setFinishOnTouchOutside(false);
            } else {
                getPackageManager().getPackageInstaller().abandonSession(mSessionId);

                if (!isCancelled()) {
                    launchFailure(PackageManager.INSTALL_FAILED_INVALID_APK, null);
                }
            }
        }
    }
```

这里根据mSessionId获取到安装会话的详细信息，如果sessionInfo不空并且是活动的，接着会创建`InstallingAsyncTask`任务并立即执行。该任务将APK信息通过IO流的形式写入 PackageInstaller.Session中。

之后，在onPostExecute方法中创建一个PendingIntent，并通过 PackageInstaller.Session 的commit方法将IntentSender发送出去。这里的 `PackageInstaller.Session.commit`方法，调用的是`IPackageInstallerSession`的commit方法，进行进程间通信，最终会调用PackageInstallerSession的commit方法。

####  com.android.server.pm.PackageInstallerSession

``` java
    @Override
    public void commit(@NonNull IntentSender statusReceiver, boolean forTransfer) {
        Preconditions.checkNotNull(statusReceiver);

        final boolean wasSealed;
        synchronized (mLock) {
           
            final PackageInstallObserverAdapter adapter = new PackageInstallObserverAdapter(
                    mContext, statusReceiver, sessionId,
                    isInstallerDeviceOwnerOrAffiliatedProfileOwnerLocked(), userId);
            mRemoteObserver = adapter.getBinder();
			//·······省略一些代码·······//
            // Client staging is fully done at this point
            mClientProgress = 1f;
            computeProgressLocked(true);

            // This ongoing commit should keep session active, even though client
            // will probably close their end.
            mActiveCount.incrementAndGet();

            mCommitted = true;
            mHandler.obtainMessage(MSG_COMMIT).sendToTarget();
        }

        if (!wasSealed) {
            // Persist the fact that we've sealed ourselves to prevent
            // mutations of any hard links we create. We do this without holding
            // the session lock, since otherwise it's a lock inversion.
            mCallback.onSessionSealedBlocking(this);
        }
    }
```

在这里将相关信息封装为PackageInstallObserverAdapter对象，然后再向mHandler发送一个`MSG_COMMIT`的消息，在这个handler的callback中是这么处理的：

``` java
 private final Handler.Callback mHandlerCallback = new Handler.Callback() {
        @Override
        public boolean handleMessage(Message msg) {
            switch (msg.what) {
          
                case MSG_COMMIT:
                    synchronized (mLock) {
                        try {
                            commitLocked();//-------1-------
                        } catch (PackageManagerException e) {
                            final String completeMsg = ExceptionUtils.getCompleteMessage(e);
                            Slog.e(TAG,
                                    "Commit of session " + sessionId + " failed: " + completeMsg);
                            destroyInternal();
                            dispatchSessionFinished(e.error, completeMsg, null);
                        }
                    }
                    break;
            }
            return true;
        }
    };
```

在注释1处调用了commitLocked，在该方法中调用了PackageManagerService对象的installStage方法

```
mPm.installStage(mPackageName, stageDir, localObserver, params,
        mInstallerPackageName, mInstallerUid, user, mSigningDetails);
```

在commitLocked方法中如果抛出了PackageManagerException，则会调用`dispatchSessionFinished`方法，向mHandler发送一个`MSG_ON_PACKAGE_INSTALLED`消息，调用`observer.onPackageInstalled(packageName, returnCode, message, extras);`这里的`observer`对象是`.PackageInstallerSession.commit`方法通过`adapter.getBinder()`方法获取到的`IPackageInstallObserver2`对象。

简单来讲，也就两个步骤：

* 将APK信息通过IO流写入到PackageInstaller.Session中
* 调用PackageInstaller.Session的commit方法，发送消息到handler，最终交给PMS处理。

### PMS介入安装过程

上面提到在commitLocked方法中，调用了`PackageManagerService`对象的installStage方法。老规矩，先看下`PackageManagerService`类的顶部注释，挺长的不贴了，主要介绍了两个锁对象以及这两个锁对象的使用方法注意事项。看下installStage方法



#### com.android.server.pm.PackageManagerService

``` java

void installStage(String packageName, File stagedDir,
                  IPackageInstallObserver2 observer, PackageInstaller.SessionParams sessionParams,
                  String installerPackageName, int installerUid, UserHandle user,
                  PackageParser.SigningDetails signingDetails) {
    if (DEBUG_INSTANT) {
        if ((sessionParams.installFlags & PackageManager.INSTALL_INSTANT_APP) != 0) {
            Slog.d(TAG, "Ephemeral install of " + packageName);
        }
    }
    final VerificationInfo verificationInfo = new VerificationInfo(
        sessionParams.originatingUri, sessionParams.referrerUri,
        sessionParams.originatingUid, installerUid);

    final OriginInfo origin = OriginInfo.fromStagedFile(stagedDir);

    final Message msg = mHandler.obtainMessage(INIT_COPY);
    final int installReason = fixUpInstallReason(installerPackageName, installerUid,
                                                 sessionParams.installReason);
    final InstallParams params = new InstallParams(origin, null, observer,
                                                   sessionParams.installFlags, installerPackageName, sessionParams.volumeUuid,
                                                   verificationInfo, user, sessionParams.abiOverride,
                                                   sessionParams.grantedRuntimePermissions, signingDetails, installReason);
    params.setTraceMethod("installStage").setTraceCookie(System.identityHashCode(params));
    msg.obj = params;//-------1-------

    Trace.asyncTraceBegin(TRACE_TAG_PACKAGE_MANAGER, "installStage",
                          System.identityHashCode(msg.obj));
    Trace.asyncTraceBegin(TRACE_TAG_PACKAGE_MANAGER, "queueInstall",
                          System.identityHashCode(msg.obj));

    mHandler.sendMessage(msg);
}
```

主要是创建了一个类型为`INIT_COPY`的Message对象，然后通过发送到mHandler。注意注释1处的msg.obj的值，是一个`InstallParams`对象(继承自HandlerParams)，后面会用到。我们接着看mHanlder的处理

#### com.android.server.pm.PackageManagerService

这里的mHandler是继承自Handler的`PackageHandler`对象，是PMS的内部类，我们看下具体的处理逻辑

``` java
 void doHandleMessage(Message msg) {
            switch (msg.what) {
                case INIT_COPY: {
                    HandlerParams params = (HandlerParams) msg.obj;
                    int idx = mPendingInstalls.size();
                    if (DEBUG_INSTALL) Slog.i(TAG, "init_copy idx=" + idx + ": " + params);
                    // If a bind was already initiated we dont really
                    // need to do anything. The pending install
                    // will be processed later on.
                    if (!mBound) {
                        Trace.asyncTraceBegin(TRACE_TAG_PACKAGE_MANAGER, "bindingMCS",
                                System.identityHashCode(mHandler));
                        // If this is the only one pending we might
                        // have to bind to the service again.
                        if (!connectToService()) {
                            Slog.e(TAG, "Failed to bind to media container service");
                            params.serviceError();
                            Trace.asyncTraceEnd(TRACE_TAG_PACKAGE_MANAGER, "bindingMCS",
                                    System.identityHashCode(mHandler));
                            if (params.traceMethod != null) {
                                Trace.asyncTraceEnd(TRACE_TAG_PACKAGE_MANAGER, params.traceMethod,
                                        params.traceCookie);
                            }
                            return;
                        } else {
                            // Once we bind to the service, the first
                            // pending request will be processed.
                            mPendingInstalls.add(idx, params);
                        }
                    } else {
                        mPendingInstalls.add(idx, params);
                        // Already bound to the service. Just make
                        // sure we trigger off processing the first request.
                        if (idx == 0) {
                            mHandler.sendEmptyMessage(MCS_BOUND);
                        }
                    }
                    break;
                }
            }
```

mBound表示是否绑定了`DefaultContainerService`服务，没有绑定的话则重新绑定，绑定成功后则将参数添加到`mPendingInstalls`中等待处理；

绑定服务的代码在这里:`DefaultContainerConnection`也是PMS的内部类。

```  java
    final private DefaultContainerConnection mDefContainerConn =
            new DefaultContainerConnection();
    class DefaultContainerConnection implements ServiceConnection {
        public void onServiceConnected(ComponentName name, IBinder service) {
            if (DEBUG_SD_INSTALL) Log.i(TAG, "onServiceConnected");
            final IMediaContainerService imcs = IMediaContainerService.Stub
                    .asInterface(Binder.allowBlocking(service));
            mHandler.sendMessage(mHandler.obtainMessage(MCS_BOUND, imcs));
        }

        public void onServiceDisconnected(ComponentName name) {
            if (DEBUG_SD_INSTALL) Log.i(TAG, "onServiceDisconnected");
        }
    }
```

可以看到，绑定成功后会发送一个类型为`MCS_BOUND`的Message对象到mHandler。当然这里发送的消息是带有Object参数，而上面`INIT_COPY`中的最后发送的消息是不带有Object参数的。

这里我们直接探讨正常流程，也就是服务已经绑定，并且mPendingInstalls中有待处理的数据，也就是走`INIT_COPY`中的最后一条分支。

这时候在PackageHandler里面会走这个分支：

``` java
else if (mPendingInstalls.size() > 0) {
                        HandlerParams params = mPendingInstalls.get(0);
                        if (params != null) {
                            Trace.asyncTraceEnd(TRACE_TAG_PACKAGE_MANAGER, "queueInstall",
                                    System.identityHashCode(params));
                            Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "startCopy");
                            if (params.startCopy()) {
                                // We are done...  look for more work or to
                                // go idle.
                                if (DEBUG_SD_INSTALL) Log.i(TAG,
                                        "Checking for more work or unbind...");
                                // Delete pending install
                                if (mPendingInstalls.size() > 0) {
                                    mPendingInstalls.remove(0);
                                }
                                if (mPendingInstalls.size() == 0) {
                                    if (mBound) {
                                        if (DEBUG_SD_INSTALL) Log.i(TAG,
                                                "Posting delayed MCS_UNBIND");
                                        removeMessages(MCS_UNBIND);
                                        Message ubmsg = obtainMessage(MCS_UNBIND);
                                        // Unbind after a little delay, to avoid
                                        // continual thrashing.
                                        sendMessageDelayed(ubmsg, 10000);
                                    }
                                } else {
                                    // There are more pending requests in queue.
                                    // Just post MCS_BOUND message to trigger processing
                                    // of next pending install.
                                    if (DEBUG_SD_INSTALL) Log.i(TAG,
                                            "Posting MCS_BOUND for next work");
                                    mHandler.sendEmptyMessage(MCS_BOUND);
                                }
                            }
                            Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
                        }
                    }
```

调用`params.startCopy()`之后，会将当前安装任务从列表中移除，接着处理下一个安装任务；这里的params是上面说到的`InstallParams`实例。

看下`HandlerParams.startCopy`方法

``` java
        final boolean startCopy() {
            boolean res;
            try {
                if (DEBUG_INSTALL) Slog.i(TAG, "startCopy " + mUser + ": " + this);

                if (++mRetries > MAX_RETRIES) {
                    Slog.w(TAG, "Failed to invoke remote methods on default container service. Giving up");
                    mHandler.sendEmptyMessage(MCS_GIVE_UP);
                    handleServiceError();
                    return false;
                } else {
                    handleStartCopy();
                    res = true;
                }
            } catch (RemoteException e) {
                if (DEBUG_INSTALL) Slog.i(TAG, "Posting install MCS_RECONNECT");
                mHandler.sendEmptyMessage(MCS_RECONNECT);
                res = false;
            }
            handleReturnCode();
            return res;
        }
```

这里的MAX_RETRIES值被定义为4，尝试次数超过4次，则放弃这个安装请求。如果没有超过，则执行`handleStartCopy`方法，该方法的具体实现在`InstallParams`中。该方法的具体实现代码很长，这里抄一下注释

```
/*
 * Invoke remote method to get package information and install
 * location values. Override install location based on default
 * policy if needed and then create install arguments based
 * on the install location.
 */
```

通过IMediaContainerService跨进程调用DefaultContainerService的getMinimalPackageInfo方法，轻量级解析apk并获取到apk的少量信息，封装到PackageInfoLite对象中。然后确认安装未知，创建InstallArgs对象。这里的InstallArgs是个抽象类，在PMS有三个对应的子类：

* FileInstallArgs:处理安装到非ASEC存储空间的APK，也就是内部存储空间(Data分区)
* AsecInstallArgs:处理安装到ASEC中(mnt/asec，也就是存储卡)中的apk
* MoveInstallArgs:处理已安装的APK在存储中移动的逻辑

复制完成后，会接着调用`handleReturnCode`方法，该方法中只是调用了`processPendingInstall`方法，看下这个方法

``` java
 mHandler.post(new Runnable() {
            public void run() {
                mHandler.removeCallbacks(this);
                 // Result object to be returned
                PackageInstalledInfo res = new PackageInstalledInfo();
                res.setReturnCode(currentStatus);
                res.uid = -1;
                res.pkg = null;
                res.removedInfo = null;
                if (res.returnCode == PackageManager.INSTALL_SUCCEEDED) {
                    args.doPreInstall(res.returnCode);//-------1-------
                    synchronized (mInstallLock) {
                        installPackageTracedLI(args, res);//-------2-------
                    }
                    args.doPostInstall(res.returnCode, res.uid);//-------3-------
                }
                ......
            }
 })
```

注释1处会检查apk状态，确保安装环境可靠，否则清除复制的apk文件，在注释3处进行安装后的收尾工作；

主要看下注释2处的方法，该方法内部会调用PMS的installPackageLI方法，这个方法也挺长的，简单说下这个方法做了什么：

* 创建PackageParser，解析apk
* 检查APK是否已经安装
* 如果PackageSetting中存在要安装的apk信息，则表示要替换安装，需要进行签名校验，确保替换安装是安全的
* 如果是替换安装，则调用replacePackageLIF方法，如果是安装新的apk，调用installNewPackageLIF方法。

我们以安装新的apk为例

1. 扫描APK，将apk信息存储在PackageParser.Package类型的newPackage中，一个Package的信息包含了一个base apk和N个split APK
2. 更新该APK对应的PackageSetting信息
3. 如果安装成功，则为新应用准备数据；如果安装失败，则删除APK

总结一下

1. PackageINstaller安装APK时会将APK的信息交由PMS处理，PMS则通过PackageHandler发送消息来驱动APK的复制和安装工作
2. PMS发送INIT_COPY和MCS_BOUND类型的消息，驱动PackageHandler来绑定DefaultContainerService，完成APK的复制等工作
3. 进行APK的安装前检查，安装APK及安装后的收尾工作



----

以上