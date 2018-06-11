---
title: Window和WindowManager
date: 2017-03-21 20:05:01
tags: [Android]
---
　　Window表示一个窗口的概念，在日常开发中直接接触WIndow的机会并不对，再试在某些特殊时候我们需要在桌面上显示一个类似悬浮窗的东西，那么这种效果就需要用到Window来实现。
　　Window只是个抽象类，它的具体实现是PhoneWindow。创建一个Window是很简单的事，只需要通过WindowManager即可完成，WindowManager是外界访问Window的入口，Window的具体实现位于WindowMangerService中，WindowMnager和WWindowMangerService的交互是一个IPC过程，Android中所有的视图都是通过Window来呈现的，不管是Activity、Dialog还是Toast，它们的视图实际上都是附加在Window上的，因此Window实际是View的直接管理者。
<!--more-->
### Window的内部机制
　　Window是一个抽象的概念，每一个Window都对应着一个View和一个ViewRootImpl，Window和View通过ViewRootImpl来建立联系，因此Window并不是实际存在的，它是以View的形式存在，。这点从WindowManager的定义也可以看出，它提供的三个接口方法addView,updateViewLayout以及removeView都是针对View，这说明View才是Window存在的实体。在实际使用中无法直接访问Window，对Window的访问必须通过WindowManger。

#### Window的添加过程
　　Window的添加过程需要通过WindowManager的addView来实现，WindowManager是一个接口，它的真正实现是WindowManagerImpl类，在WindowManagerImpl中Window的三大操作实现如下：
```java
	@Override
    public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
    }
    
	@Override
    public void updateViewLayout(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.updateViewLayout(view, params);
    }
	
	@Override
    public void removeView(View view) {
        mGlobal.removeView(view, false);
    }

    @Override
    public void removeViewImmediate(View view) {
        mGlobal.removeView(view, true);
    }
```
　　WindowManagerImpl并没有直接实现Window的三大操作，而是全部交给了WindowManagerGlobal来处理，WindowManagerGlobal以工厂的形式向外提供自己的实现，WindowManagerImpl这种工作模式是典型的桥接模式，将所有的操作全部委托给WindowManagerGlobal来实现。在WindowManagerGlobal的addView方法主要分为如下几步：
1. **检查参数是否合法，如果是子Window，那么还需要调整一些布局参数。**

``` java
 public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {
        if (view == null) {
            throw new IllegalArgumentException("view must not be null");
        }
        if (display == null) {
            throw new IllegalArgumentException("display must not be null");
        }
        if (!(params instanceof WindowManager.LayoutParams)) {
            throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
        }

        final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
        if (parentWindow != null) {
            parentWindow.adjustLayoutParamsForSubWindow(wparams);
        } else {
            // If there's no parent, then hardware acceleration for this view is
            // set from the application's hardware acceleration setting.
            final Context context = view.getContext();
            if (context != null
                    && (context.getApplicationInfo().flags
                            & ApplicationInfo.FLAG_HARDWARE_ACCELERATED) != 0) {
                wparams.flags |= WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED;
            }
        }
	}
```

2. **创建ViewRootImpl并将View添加到列表中。**
在WindowManagerGlobal内部有如下几个列表比较重要：

``` java
	private final ArrayList<View> mViews = new ArrayList<View>();
    private final ArrayList<ViewRootImpl> mRoots = new ArrayList<ViewRootImpl>();
    private final ArrayList<WindowManager.LayoutParams> mParams =
            new ArrayList<WindowManager.LayoutParams>();
    private final ArraySet<View> mDyingViews = new ArraySet<View>();
```
　　在以上声明中，mView存储的是多有WIndow所对应的View,mRoots存储的是所有Window所对应的ViewRootImpl，mParams存储的是所有Window所对应的布局参数，而mDyingView则存储了那些正在被删除的View对象，或者说是那些已经调用removeView方法但是删除操作还未完成的Window对象，在addView中通过如下方式将Window的一系列对象添加到列表中：
``` java
	root = new ViewRootImpl(view.getContext(), display);
    view.setLayoutParams(wparams);
    mViews.add(view);
    mRoots.add(root);
    mParams.add(wparams);
```
3. **通过ViewRootImpl来更新界面并完成Window的添加过程。**
这个过程由ViewRootImpl的setView方法来完成：
``` java
 // do this last because it fires off messages to start doing things
        try {
            root.setView(view, wparams, panelParentView);
        } catch (RuntimeException e) {
            // BadTokenException or InvalidDisplayException, clean up.
            synchronized (mLock) {
                final int index = findViewLocked(view, false);
                if (index >= 0) {
                    removeViewLocked(index, true);
                }
            }
            throw e;
        }
```
在setView内部会通过requestLayout来完成异步刷新请求。在下面的代码中，scheduleTraversals实际是View绘制的入口：
``` java
 @Override
    public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            checkThread();
            mLayoutRequested = true;
            scheduleTraversals();
        }
    }
```
接着会通过WindowSession最终来完成Window的添加过程。
#### Window的删除过程
Window的删除过程和添加过程一样，都是先通过WindowManagerImpl后，再进一步通过WindowManagerGlobal来实现的，如下：
``` java
public void removeView(View view, boolean immediate) {
        if (view == null) {
            throw new IllegalArgumentException("view must not be null");
        }

        synchronized (mLock) {
            int index = findViewLocked(view, true);
            View curView = mRoots.get(index).getView();
            removeViewLocked(index, immediate);
            if (curView == view) {
                return;
            }

            throw new IllegalStateException("Calling with view " + view
                    + " but the ViewAncestor is attached to " + curView);
        }
    }
```
　　removeView的逻辑很清晰，首先通过findViewLocked来查找待删除的View的索引，这个查找过程就是建立的数据遍历，然后再通过调用removeViewLocked来做进一步的删除，如下：
``` java
private void removeViewLocked(int index, boolean immediate) {
        ViewRootImpl root = mRoots.get(index);
        View view = root.getView();

        if (view != null) {
            InputMethodManager imm = InputMethodManager.getInstance();
            if (imm != null) {
                imm.windowDismissed(mViews.get(index).getWindowToken());
            }
        }
        boolean deferred = root.die(immediate);
        if (view != null) {
            view.assignParent(null);
            if (deferred) {
                mDyingViews.add(view);
            }
        }
    }
```
　　removeViewLocked是通过ViewRootImpl来完成删除操作的，在WindowManager中提供了两种删除接口removeView和removeViewImmediate,它们分别表示异步删除和同步删除，一般不需要使用removeViewImmediate这个方法。这里主要说异步删除的情况：具体的删除操作由ViewRootImpl的die方法来完成，在异步删除的情况下，die方法只是发送了一个请求删除的消息后就立刻返回了，**这个时候View并没有完成删除操作**，所以最后会将其添加到mDyingView中。ViewRootImpl的die方法如下：
``` java
	boolean die(boolean immediate) {
    // Make sure we do execute immediately if we are in the middle of a traversal or the damage
    // done by dispatchDetachedFromWindow will cause havoc on return.
    if (immediate && !mIsInTraversal) {
        doDie();
        return false;
    }

    if (!mIsDrawing) {
        destroyHardwareRenderer();
    } else {
        Log.e(mTag, "Attempting to destroy the window while drawing!\n" +
                "  window=" + this + ", title=" + mWindowAttributes.getTitle());
    }
    mHandler.sendEmptyMessage(MSG_DIE);
    return true;
    }
```
　　在die方法中只是做了简单的判断，如果是异步删除，那么就发送一个MSG_DIE的消息，ViewRootImpl中的Handler会处理此消息并调用doDie方法，如果是同步删除，那么就不发送消息直接调用doDie方法。在doDie内部会调用dispatchDetachedFromWindow方法，真正删除View的逻辑在dispatchDetachedFromWindow方法内部实现。dispatchDetachedFromWindow方法主要做四件事：
1. 垃圾回收相关的工作，比如清除数据和消息，移除回调等。
2. 通过Session的remove方法删除Window：mWindowSession.remove(mWindow)，这同样是一个IPC过程，最终会调用WindowManagerService的removeWindow方法。
3. 调用View的dispatchDetachedFromWindow方法，在内部会调用View的onDetachedFromWindow以及onDetachedFromWindowInternal()。
4. 调用WindowManagerGlobal的doRemoveView方法刷新数据，包括mRoots、mParams以及mDyingViews，需要将当前Window锁关联的这三类对象从列表中删除。
#### Window的更新过程
``` java
	public void updateViewLayout(View view, ViewGroup.LayoutParams params) {
        if (view == null) {
            throw new IllegalArgumentException("view must not be null");
        }
        if (!(params instanceof WindowManager.LayoutParams)) {
            throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
        }

        final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams)params;

        view.setLayoutParams(wparams);

        synchronized (mLock) {
            int index = findViewLocked(view, true);
            ViewRootImpl root = mRoots.get(index);
            mParams.remove(index);
            mParams.add(index, wparams);
            root.setLayoutParams(wparams, false);
        }
    }
```
首先它需要更新View的LayoutParams并替换掉老的LayoutParams，接着再更新ViewRootImpl中的layoutParams，这一步是通过ViewRootImpl的setLayoutParams方法来实现的。在ViewRootImpl中会通过scheduleTraversals方法来对View重新布局，包括测量、布局、重绘这三个过程，除了View本身的重绘以外，ViewRootImpl还会通过WindowSession开更新Window的视图，这个过程最终是由WindowManagerService的relayoutWindow()来具体实现，它同样是一个IPC的过程。
***
以上
