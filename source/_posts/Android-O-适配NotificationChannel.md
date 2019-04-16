---
title: Android O---适配NotificationChannel
tags: [Android,Android爬坑之旅]
date: 2018-12-27 19:47:25
mathjax: true
keywords: [invalid channel for service notification,startForeground,NotificationChannel]
---

继之前跪在Android N的`StrictMode`上了。现在又跪在的Android O 的NotificationChannel上了

场景如下：

某些场景中需要上传图片，选择图片或者拍照时使用系统的图库会将自己的app置于后台，若选择图片的时间过长，则可能会导致自己的app会杀死。看了一下传承下来的代码，是在这种情况下发送一个前台通知`startForeground`,使此服务在前台运行。但是会在通知栏上显示一个应用正在运行的通知

![通知](/image/Android/NotificationChanel/startForeground.png  "startForeground通知")

<!--more-->

#### 源码分析

查看Android 28的源码，发现调用链如下：

首先 `Service.java`中调用 

``` java
 public final void startForeground(int id, Notification notification) {
        try {
            mActivityManager.setServiceForeground(
                    new ComponentName(this, mClassName), mToken, id,
                    notification, 0);
        } catch (RemoteException ex) {
        }
    }
```

这里的`mActivityManager`的声明是`private IActivityManager mActivityManager = null;`在这里，`IActivityManager`的实现类是`ActivityManagerService`

``` java
@Override
    public void setServiceForeground(ComponentName className, IBinder token,
            int id, Notification notification, int flags) {
        synchronized(this) {
            mServices.setServiceForegroundLocked(className, token, id, notification, flags);
        }
    }
```

``` java
 public void setServiceForegroundLocked(ComponentName className, IBinder token,
            int id, Notification notification, int flags) {
        final int userId = UserHandle.getCallingUserId();
        final long origId = Binder.clearCallingIdentity();
        try {
            ServiceRecord r = findServiceLocked(className, token, userId);
            if (r != null) {
                setServiceForegroundInnerLocked(r, id, notification, flags);
            }
        } finally {
            Binder.restoreCallingIdentity(origId);
        }
    }
```

finally中调用的Binder方法是一个native方法，主要看一下`setServiceForegroundInnerLocked`:

方法太长，关注一下我们需要的

``` java
                // Apps under strict background restrictions simply don't get to have foreground
                // services, so now that we've enforced the startForegroundService() contract
                // we only do the machinery of making the service foreground when the app
                // is not restricted.
                if (!ignoreForeground) {
                    if (r.foregroundId != id) {
                        cancelForegroundNotificationLocked(r);
                        r.foregroundId = id;
                    }
                    notification.flags |= Notification.FLAG_FOREGROUND_SERVICE;
                    r.foregroundNoti = notification;
                    if (!r.isForeground) {
                        final ServiceMap smap = getServiceMapLocked(r.userId);
                        if (smap != null) {
                            ActiveForegroundApp active = smap.mActiveForegroundApps.get(r.packageName);
                            if (active == null) {
                                active = new ActiveForegroundApp();
                                active.mPackageName = r.packageName;
                                active.mUid = r.appInfo.uid;
                                active.mShownWhileScreenOn = mScreenOn;
                                if (r.app != null) {
                                    active.mAppOnTop = active.mShownWhileTop =
                                            r.app.uidRecord.curProcState
                                                    <= ActivityManager.PROCESS_STATE_TOP;
                                }
                                active.mStartTime = active.mStartVisibleTime
                                        = SystemClock.elapsedRealtime();
                                smap.mActiveForegroundApps.put(r.packageName, active);
                                requestUpdateActiveForegroundAppsLocked(smap, 0);
                            }
                            active.mNumActive++;
                        }
                        r.isForeground = true;
                        mAm.mAppOpsService.startOperation(
                                AppOpsManager.getToken(mAm.mAppOpsService),
                                AppOpsManager.OP_START_FOREGROUND, r.appInfo.uid, r.packageName,
                                true);
                        StatsLog.write(StatsLog.FOREGROUND_SERVICE_STATE_CHANGED,
                                r.appInfo.uid, r.shortName,
                                StatsLog.FOREGROUND_SERVICE_STATE_CHANGED__STATE__ENTER);
                    }
                    r.postNotification();
                    if (r.app != null) {
                        updateServiceForegroundLocked(r.app, true);
                    }
                    getServiceMapLocked(r.userId).ensureNotStartingBackgroundLocked(r);
                    mAm.notifyPackageUse(r.serviceInfo.packageName,
                            PackageManager.NOTIFY_PACKAGE_USE_FOREGROUND_SERVICE);
                }
```

这里面的` r.postNotification();`这里的`r`是`ServiceRecord`的一个实例对象，在该方法中调用了`ams.mHandler.post(new Runnable())`方法，ams是`ActivityManagerService`的一个实例，在这里创建了一个匿名内部类：

``` java
ams.mHandler.post(new Runnable() {
                public void run() {
                    NotificationManagerInternal nm = LocalServices.getService(
                            NotificationManagerInternal.class);
                    if (nm == null) {
                        return;
                    }
                    Notification localForegroundNoti = _foregroundNoti;
                    try {
                        if (localForegroundNoti.getSmallIcon() == null) {
                            // It is not correct for the caller to not supply a notification
                            // icon, but this used to be able to slip through, so for
                            // those dirty apps we will create a notification clearly
                            // blaming the app.
                            Slog.v(TAG, "Attempted to start a foreground service ("
                                    + name
                                    + ") with a broken notification (no icon: "
                                    + localForegroundNoti
                                    + ")");

                            CharSequence appName = appInfo.loadLabel(
                                    ams.mContext.getPackageManager());
                            if (appName == null) {
                                appName = appInfo.packageName;
                            }
                            Context ctx = null;
                            try {
                                ctx = ams.mContext.createPackageContextAsUser(
                                        appInfo.packageName, 0, new UserHandle(userId));

                                Notification.Builder notiBuilder = new Notification.Builder(ctx,
                                        localForegroundNoti.getChannelId());

                                // it's ugly, but it clearly identifies the app
                                notiBuilder.setSmallIcon(appInfo.icon);

                                // mark as foreground
                                notiBuilder.setFlag(Notification.FLAG_FOREGROUND_SERVICE, true);

                                Intent runningIntent = new Intent(
                                        Settings.ACTION_APPLICATION_DETAILS_SETTINGS);
                                runningIntent.setData(Uri.fromParts("package",
                                        appInfo.packageName, null));
                                PendingIntent pi = PendingIntent.getActivityAsUser(ams.mContext, 0,
                                        runningIntent, PendingIntent.FLAG_UPDATE_CURRENT, null,
                                        UserHandle.of(userId));
                                notiBuilder.setColor(ams.mContext.getColor(
                                        com.android.internal
                                                .R.color.system_notification_accent_color));
                                notiBuilder.setContentTitle(
                                        ams.mContext.getString(
                                                com.android.internal.R.string
                                                        .app_running_notification_title,
                                                appName));
                                notiBuilder.setContentText(
                                        ams.mContext.getString(
                                                com.android.internal.R.string
                                                        .app_running_notification_text,
                                                appName));
                                notiBuilder.setContentIntent(pi);

                                localForegroundNoti = notiBuilder.build();
                            } catch (PackageManager.NameNotFoundException e) {
                            }
                        }
                        if (nm.getNotificationChannel(localPackageName, appUid,
                                localForegroundNoti.getChannelId()) == null) {
                            int targetSdkVersion = Build.VERSION_CODES.O_MR1;
                            try {
                                final ApplicationInfo applicationInfo =
                                        ams.mContext.getPackageManager().getApplicationInfoAsUser(
                                                appInfo.packageName, 0, userId);
                                targetSdkVersion = applicationInfo.targetSdkVersion;
                            } catch (PackageManager.NameNotFoundException e) {
                            }
                            if (targetSdkVersion >= Build.VERSION_CODES.O_MR1) {
                                throw new RuntimeException(
                                        "invalid channel for service notification: "
                                                + foregroundNoti);
                            }
                        }
                        if (localForegroundNoti.getSmallIcon() == null) {
                            // Notifications whose icon is 0 are defined to not show
                            // a notification, silently ignoring it.  We don't want to
                            // just ignore it, we want to prevent the service from
                            // being foreground.
                            throw new RuntimeException("invalid service notification: "
                                    + foregroundNoti);
                        }
                        nm.enqueueNotification(localPackageName, localPackageName,
                                appUid, appPid, null, localForegroundId, 												localForegroundNoti,userId);

                        foregroundNoti = localForegroundNoti; // save it for amending next time
                    } catch (RuntimeException e) {
                        Slog.w(TAG, "Error showing notification for service", e);
                        // If it gave us a garbage notification, it doesn't
                        // get to be foreground.
                        ams.setServiceForeground(name, ServiceRecord.this,
                                0, null, 0);
                        ams.crashApplication(appUid, appPid, localPackageName, -1,
                                "Bad notification for startForeground: " + e);
                    }
                }
            });
```

首先检查一下`getSmallIcon`是不是空，如果是空的，打印日志，然后创建一个新的Notification。然后走不为空的判断：检查有没有`NotificationChannel`:

``` java
nm.getNotificationChannel(localPackageName, appUid,localForegroundNoti.getChannelId()) == null
```

这里面如果`targetSdkVersion >= Build.VERSION_CODES.O_MR1`,则会抛出`invalid channel for service notification`异常。

如果一切正常，入队列发通知。

----

至此，应用中发生崩溃的原因找到了，targetSdk是28，又没有NotificationChannel。

那么，NotificationChannel又是个卵？？？？

####  NotificationChannel

从Android 8.0系统开始，Google引入了通知渠道这个概念。

什么是通知渠道呢？顾名思义，就是每条通知都要属于一个对应的渠道。每个App都可以自由地创建当前App拥有哪些通知渠道，但是这些通知渠道的控制权都是掌握在用户手上的。用户可以自由地选择这些通知渠道的重要程度，是否响铃、是否振动、或者是否要关闭这个渠道的通知。

Google这次对于8.0系统通知渠道的推广态度还是比较强硬的。

首先，如果你升级了appcompat库，那么所有使用appcompat库来构建通知的地方全部都会进行废弃方法提示，如下所示：

![通知](/image/Android/NotificationChanel/Notification_build.png  "startForeground通知")

##### 创建NotificationChannel

创建NotificationChannel对象(不必每次创建)，并且创建通知渠道的代码只在第一次执行的时候才会创建，以后每次执行创建代码系统会检测到该通知渠道已经存在了，因此不会重复创建，也并不会影响任何效率。

``` java
 if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            String channelId = "take_photo";
            String channelName = "拍照选择图片";
            int importance = NotificationManager.IMPORTANCE_HIGH;
            NotificationManager
                    notificationManager = (NotificationManager) getSystemService(NOTIFICATION_SERVICE);
            NotificationChannel channel = notificationManager.getNotificationChannel(channelId);
            if (channel == null) {
                channel = new NotificationChannel(channelId, channelName, importance);
            }
            notificationManager.createNotificationChannel(channel);
            if (channel.getImportance() == NotificationManager.IMPORTANCE_NONE) {
                Intent intent = new Intent(Settings.ACTION_CHANNEL_NOTIFICATION_SETTINGS);
                intent.putExtra(Settings.EXTRA_APP_PACKAGE, getPackageName());
                intent.putExtra(Settings.EXTRA_CHANNEL_ID, channel.getId());
                startActivity(intent);
                Toast.makeText(this, "请手动将通知打开", Toast.LENGTH_SHORT).show();
            } else {
                Notification nf = new Notification.Builder(this, channelId)
                        .setContentText("setContentText")
                        .setSettingsText("setSettingsText")
                        .setContentTitle("setContentTitle")
                        .setSmallIcon(R.drawable.ic_launcher_background)
                        .build();
                notificationManager.notify(1, nf);
            }
        }
    }
```

首先要确保的是当前手机的系统版本必须是Android 8.0系统或者更高，因为低版本的手机系统并没有通知渠道这个功能，不做系统版本检查的话会在低版本手机上造成崩溃。创建一个通知渠道至少需要渠道ID、渠道名称以及重要等级这三个参数，其中渠道ID可以随便定义，只要保证全局唯一性就可以。渠道名称是给用户看的，需要能够表达清楚这个渠道的用途。重要等级的不同则会决定通知的不同行为，当然这里只是初始状态下的重要等级，用户可以随时手动更改某个渠道的重要等级，App是无法干预的。

创建通知的代码就不再多做解释了，和传统创建通知的方法没什么两样，只是在NotificationCompat.Builder中需要多传入一个通知渠道ID，那么这里我们传入刚刚创建的渠道ID。

##### 管理通知渠道

通知渠道一旦创建之后就不能再通过代码修改了，如果用户不小心关闭了该渠道，我们可以通知用户手动去修改，就像动态权限申请一样：

``` java
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
	NotificationChannel channel = manager.getNotificationChannel("chat");
    if (channel.getImportance() == NotificationManager.IMPORTANCE_NONE) {
    	Intent intent = new Intent(Settings.ACTION_CHANNEL_NOTIFICATION_SETTINGS);
        intent.putExtra(Settings.EXTRA_APP_PACKAGE, getPackageName());
        intent.putExtra(Settings.EXTRA_CHANNEL_ID, channel.getId());
        startActivity(intent);
        Toast.makeText(this, "请手动将通知打开", Toast.LENGTH_SHORT).show();
    }
}
```

通过getNotificationChannel()方法获取到了NotificationChannel对象，然后就可以读取该通知渠道下的所有配置了。这里我们判断如果通知渠道的importance等于IMPORTANCE_NONE，就说明用户将该渠道的通知给关闭了，这时会跳转到通知的设置界面提醒用户手动打开。

Android 8.0还赋予了我们删除通知渠道的功能，只需使用如下代码即可删除：

``` java
NotificationManager manager = (NotificationManager)getSystemService(NOTIFICATION_SERVICE);
manager.deleteNotificationChannel(channelId);
```

NotificationChannel章节参考 https://mp.weixin.qq.com/s/PG8XxHEHpijL4AMZ4AGP8g

----

以上