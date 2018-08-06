---
title: Toast显示过程
date: 2018-08-03 11:35:14
tags: [Android]
keywords: Toast显示过程
---
抄书抄的有点烦，自己也学着分析一下源码，挑了个在我看来比较简单的Toast显示过程来玩一玩。
``` java
Toast.makeText(context, text, duration).show();
```
先了解一下makeText过程，该过程最终都会调用下面的方法：
``` java
/**
     * Make a standard toast to display using the specified looper.
     * If looper is null, Looper.myLooper() is used.
     * @hide
     */
    public static Toast makeText(@NonNull Context context, @Nullable Looper looper,
            @NonNull CharSequence text, @Duration int duration) {
        Toast result = new Toast(context, looper);

        LayoutInflater inflate = (LayoutInflater)
                context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
        View v = inflate.inflate(com.android.internal.R.layout.transient_notification, null);
        TextView tv = (TextView)v.findViewById(com.android.internal.R.id.message);
        tv.setText(text);

        result.mNextView = v;
        result.mDuration = duration;

        return result;
    }
```
创建一个新的Toast对象，在Toast的构造方法中有一个TN类型的mTN对象：
``` java
mTN = new TN(context.getPackageName(), looper);
```
`TN`是`ITransientNotification.Stub`的子类，注意一下这个对象，这对后面的过程很重要。
<!--more-->
得到Toast对象后接着调用`show`方法：
``` java
/**
     * Show the view for the specified duration.
     */
    public void show() {
        if (mNextView == null) {
            throw new RuntimeException("setView must have been called");
        }

        INotificationManager service = getService();
        String pkg = mContext.getOpPackageName();
        TN tn = mTN;
        tn.mNextView = mNextView;

        try {
            service.enqueueToast(pkg, tn, mDuration);
        } catch (RemoteException e) {
            // Empty
        }
    }
```
重点在`service`上，它是`INotificationManager`的实例对象，是个用于Binder机制IPC通信的接口，实际上getService返回了一个以"notification"为标记的远程服务对象的代理。这个远程服务对象就是NotificationManagerService，以下简称NMS。NMS中的enqueueToast方法：
``` java
@Override
public void enqueueToast(String pkg, ITransientNotification callback, int duration)
{
    ......
    synchronized (mToastQueue) {
        int callingPid = Binder.getCallingPid();
        long callingId = Binder.clearCallingIdentity();
        try {
            ToastRecord record;
            int index;
            // All packages aside from the android package can enqueue one toast at a time
            if (!isSystemToast) {
                index = indexOfToastPackageLocked(pkg);
            } else {
                index = indexOfToastLocked(pkg, callback);
            }

            // If the package already has a toast, we update its toast
            // in the queue, we don't move it to the end of the queue.
            if (index >= 0) {
                record = mToastQueue.get(index);
                record.update(duration);
                record.update(callback);
            } else {
                Binder token = new Binder();
                mWindowManagerInternal.addWindowToken(token, TYPE_TOAST, DEFAULT_DISPLAY);
                record = new ToastRecord(callingPid, pkg, callback, duration, token);
                mToastQueue.add(record);
                index = mToastQueue.size() - 1;
            }
            keepProcessAliveIfNeededLocked(callingPid);
            // If it's at index 0, it's the current toast.  It doesn't matter if it's
            // new or just been updated.  Call back and tell it to show itself.
            // If the callback fails, this will remove it from the list, so don't
            // assume that it's valid after this.
            if (index == 0) {
                showNextToastLocked();
            }
        } finally {
            Binder.restoreCallingIdentity(callingId);
        }
    }
}
```
方法中的第二个参数callBack就是上面的TN对象，关键在于下面的index的判断：
mToastQueue是个list，如果是非系统tost，并且该taost存在于list中(根据pkg判断)，就取出该ToastRecord并且更新，如果不存在，则新建一个ToastRecord存入list中。
如果ToastRecord是在list的第一个位置，接着调用`showNextToastLocaked`方法
``` java
@GuardedBy("mToastQueue")
void showNextToastLocked() {
    ToastRecord record = mToastQueue.get(0);
    while (record != null) {
        if (DBG) Slog.d(TAG, "Show pkg=" + record.pkg + " callback=" + record.callback);
        try {
            record.callback.show(record.token);
            scheduleTimeoutLocked(record);
            return;
        } catch (RemoteException e) {
            Slog.w(TAG, "Object died trying to show notification " + record.callback
                    + " in package " + record.pkg);
            // remove it from the list and let the process die
            int index = mToastQueue.indexOf(record);
            if (index >= 0) {
                mToastQueue.remove(index);
            }
            keepProcessAliveIfNeededLocked(record.pid);
            if (mToastQueue.size() > 0) {
                record = mToastQueue.get(0);
            } else {
                record = null;
            }
        }
    }
}
```
关键的两行代码
``` java
  record.callback.show(record.token);
  scheduleTimeoutLocked(record);
```
第一行负责显示，第二行负责超时关闭并显示队列中的下一个Toast。
在第一行中，callBack就是我们上面提到的TN对象的实例mTN：
``` java
/**
* schedule handleShow into the right thread
*/
@Override
public void show(IBinder windowToken) {
    if (localLOGV) Log.v(TAG, "SHOW: " + this);
    mHandler.obtainMessage(SHOW, windowToken).sendToTarget();
}
```
在`mHandler`中调用了`handleShow`方法：
``` java 
public void handleShow(IBinder windowToken) {
            if (localLOGV) Log.v(TAG, "HANDLE SHOW: " + this + " mView=" + mView
                    + " mNextView=" + mNextView);
            // If a cancel/hide is pending - no need to show - at this point
            // the window token is already invalid and no need to do any work.
            if (mHandler.hasMessages(CANCEL) || mHandler.hasMessages(HIDE)) {
                return;
            }
            if (mView != mNextView) {
                // remove the old view if necessary
                handleHide();
                mView = mNextView;
                Context context = mView.getContext().getApplicationContext();
                String packageName = mView.getContext().getOpPackageName();
                if (context == null) {
                    context = mView.getContext();
                }
                mWM = (WindowManager)context.getSystemService(Context.WINDOW_SERVICE);
                // We can resolve the Gravity here by using the Locale for getting
                // the layout direction
                final Configuration config = mView.getContext().getResources().getConfiguration();
                final int gravity = Gravity.getAbsoluteGravity(mGravity, config.getLayoutDirection());
                mParams.gravity = gravity;
                if ((gravity & Gravity.HORIZONTAL_GRAVITY_MASK) == Gravity.FILL_HORIZONTAL) {
                    mParams.horizontalWeight = 1.0f;
                }
                if ((gravity & Gravity.VERTICAL_GRAVITY_MASK) == Gravity.FILL_VERTICAL) {
                    mParams.verticalWeight = 1.0f;
                }
                mParams.x = mX;
                mParams.y = mY;
                mParams.verticalMargin = mVerticalMargin;
                mParams.horizontalMargin = mHorizontalMargin;
                mParams.packageName = packageName;
                mParams.hideTimeoutMilliseconds = mDuration ==
                    Toast.LENGTH_LONG ? LONG_DURATION_TIMEOUT : SHORT_DURATION_TIMEOUT;
                mParams.token = windowToken;
                if (mView.getParent() != null) {
                    if (localLOGV) Log.v(TAG, "REMOVE! " + mView + " in " + this);
                    mWM.removeView(mView);
                }
                if (localLOGV) Log.v(TAG, "ADD! " + mView + " in " + this);
                // Since the notification manager service cancels the token right
                // after it notifies us to cancel the toast there is an inherent
                // race and we may attempt to add a window after the token has been
                // invalidated. Let us hedge against that.
                try {
                    mWM.addView(mView, mParams);
                    trySendAccessibilityEvent();
                } catch (WindowManager.BadTokenException e) {
                    /* ignore */
                }
            }
        }
```
上面的一坨是获取到`WindowManager`然后通过`mWM.addView`将Toast显示到窗口上。
至于`WindowManager`如何添加view的，可以看这个：http://blog.huangyuanlove.com/2017/03/21/Window和CWindowManager/
接下来看一下怎么取消Toast弹窗，上面提到了调用了`scheduleTimeoutLocked`方法。
``` java
@GuardedBy("mToastQueue")
private void scheduleTimeoutLocked(ToastRecord r){
    mHandler.removeCallbacksAndMessages(r);
    Message m = Message.obtain(mHandler, MESSAGE_TIMEOUT, r);
    long delay = r.duration == Toast.LENGTH_LONG ? LONG_DELAY : SHORT_DELAY;
    mHandler.sendMessageDelayed(m, delay);
}
```
调用mHandler发送了一个延时消息去调用`handleTimeout`方法，延迟时间就是根据duration来判断的。这里的`mHandler`是`WorkerHandler`继承自`Handler`
``` java
private void handleTimeout(ToastRecord record)
    {
        if (DBG) Slog.d(TAG, "Timeout pkg=" + record.pkg + " callback=" + record.callback);
        synchronized (mToastQueue) {
            int index = indexOfToastLocked(record.pkg, record.callback);
            if (index >= 0) {
                cancelToastLocked(index);
            }
        }
    }
```
接着调用了`cancelToastLocked`方法
``` java
@GuardedBy("mToastQueue")
    void cancelToastLocked(int index) {
        ToastRecord record = mToastQueue.get(index);
        try {
            record.callback.hide();
        } catch (RemoteException e) {
            Slog.w(TAG, "Object died trying to hide notification " + record.callback
                    + " in package " + record.pkg);
            // don't worry about this, we're about to remove it from
            // the list anyway
        }

        ToastRecord lastToast = mToastQueue.remove(index);
        mWindowManagerInternal.removeWindowToken(lastToast.token, true, DEFAULT_DISPLAY);

        keepProcessAliveIfNeededLocked(record.pid);
        if (mToastQueue.size() > 0) {
            // Show the next one. If the callback fails, this will remove
            // it from the list, so don't assume that the list hasn't changed
            // after this point.
            showNextToastLocked();
        }
    }
```
在该方法中调用了`callback.hide()`方法移除toast的显示。
``` java
 public void handleHide() {
    if (localLOGV) Log.v(TAG, "HANDLE HIDE: " + this + " mView=" + mView);
    if (mView != null) {
        // note: checking parent() just to make sure the view has
        // been added...  i have seen cases where we get here when
        // the view isn't yet added, so let's try not to crash.
        if (mView.getParent() != null) {
            if (localLOGV) Log.v(TAG, "REMOVE! " + mView + " in " + this);
            mWM.removeViewImmediate(mView);
        }

        mView = null;
    }
}
```
如果这时候mToastQueue中还有ToastRecord，则调用`showNextToastLocked`方法显示下一个。

----
以上