---
title: Android hook--示例
date: 2018-08-06 15:46:06
tags: [Android]
keywords: AndroidHook,反射,动态代理
photos:
  - /image/Android/hook/guide.jpg
---

Hook过程：
1. 寻找 Hook 点，原则是静态变量或者单例对象，尽量 Hook public 的对象和方法。
   Hook的选择点：静态变量和单例，因为一旦创建对象，它们不容易变化，非常容易定位。
2. 选择合适的代理方式，如果是接口可以用动态代理。
3. 偷梁换柱——用代理对象替换原始对象。
注意Android 的 API 版本比较多，方法和类可能不一样，所以要做好 API 的兼容工作。还有不要hook太底层的东西，各个厂商的rom代码不一样
<!--more-->

#### hookView的点击事件
先来个简单点的，View的点击事件。

##### hookOnLongClick
顺着View的`setOnClickListener`方法找到了`getListenerInfo`方法，进而找到了`ListenerInfo`类，而view的click，longClick，ScrollChange的监听事件都存放在这里面。
``` java
    private void hookOnLongCLickListener(View view) {
        try {
            //拿到mListenerInfo ，可以通过getListenerInfo方法
            Class<?> clazzView = Class.forName("android.view.View");
            Method getListenerInfoMethod = clazzView.getDeclaredMethod("getListenerInfo");
            getListenerInfoMethod.setAccessible(true);
            Object listenerInfo = getListenerInfoMethod.invoke(view);

            //拿到 mOnLongClickListener字段，这里的ListenerInfo是View的内部类，需要用$符号链接。
            Class<?> clazz = Class.forName("android.view.View$ListenerInfo");
            Field field = clazz.getDeclaredField("mOnLongClickListener");
            field.setAccessible(true);
            //拿到原来的mOnLongClickListener字段的值
            View.OnLongClickListener raw =(View.OnLongClickListener) field.get(listenerInfo);
            //替换成我们自己的
            field.set(listenerInfo, new HookOnLongClickListener(raw));

        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    class HookOnLongClickListener implements View.OnLongClickListener{
        private View.OnLongClickListener raw;
        public HookOnLongClickListener(View.OnLongClickListener raw){
            this.raw = raw;
        }
        @Override
        public boolean onLongClick(View v) {
            Log.e("HookUtil","HookOnLongClickListener");
            Toast.makeText(MainActivity.this,"替换之后",Toast.LENGTH_SHORT).show();
            if(raw!=null){
                //调用原来的onLongClick，保持原有逻辑不变
                raw.onLongClick(v);
            }
            return false;
        }
    }
```

这样调用`hookOnLongCLickListerner(view)`方法即可在原有逻辑不变的情况下添加我们自己的逻辑。

##### hookOnLongClick
```java
    private void hookOnClickListener(View view) {
        try {
            //拿到mListenerInfo ，可以通过getListenerInfo方法
            Method getListenerInfoMethod = view.getClass().getDeclaredMethod("getListenerInfo");
            getListenerInfoMethod.setAccessible(true);
            Object listenerInfo = getListenerInfoMethod.invoke(view);

            // 得到 原始的 OnClickListener 对象
            Class<?> listenerInfoClz = Class.forName("android.view.View$ListenerInfo");
            Field mOnClickListener = listenerInfoClz.getDeclaredField("mOnClickListener");
            mOnClickListener.setAccessible(true);
            View.OnClickListener originOnClickListener = (View.OnClickListener) mOnClickListener.get(listenerInfo);

            // 用自定义的 OnClickListener 替换原始的 OnClickListener
            View.OnClickListener hookedOnClickListener = new HookedOnClickListener(originOnClickListener);
            mOnClickListener.set(listenerInfo, hookedOnClickListener);

        } catch (Exception e) {
        }
    }

    class HookedOnClickListener implements View.OnClickListener {
        private View.OnClickListener origin;

        HookedOnClickListener(View.OnClickListener origin) {
            this.origin = origin;
        }

        @Override
        public void onClick(View v) {
            Toast.makeText(MainActivity.this, "hook click", Toast.LENGTH_SHORT).show();
            Log.i("hook", "Before click, do what you want to to.");
            if (origin != null) {
                origin.onClick(v);
            }
            Log.i("hook", "After click, do what you want to to.");
        }
    }


```
这样调用`hookOnLongClickListerner(view)`方法即可在原有逻辑不变的情况下添加我们自己的逻辑。

#### hookAlarmManager

在设置Alarm的过程中，会调用AlarmManager.set方法，而AlarmManager对象又很方便得到：

``` java
    AlarmManager alarm = (AlarmManager) getSystemService(ALARM_SERVICE);
    Class<?> alarmManagerClass = alarm.getClass();
    Field mService = alarmManagerClass.getDeclaredField("mService");
    mService.setAccessible(true);
    Object mSerViceInstant = mService.get(alarm);

    AlarmManagerInvocationHandler handler = new AlarmManagerInvocationHandler(mSerViceInstant);
    Class<?> IActivityManagerIntercept = Class.forName("android.app.IAlarmManager");
    Object proxy = Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(),
            new Class<?>[]{IActivityManagerIntercept}, handler);
    mService.set(alarm, proxy);

    //动态代理
    class AlarmManagerInvocationHandler implements InvocationHandler {

        private Object iAlarmManagerObject;

        private AlarmManagerInvocationHandler(Object iAlarmManagerObject) {
            this.iAlarmManagerObject = iAlarmManagerObject;
        }

        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {


            Log.i("alarm_manager", method.getName());
            
            if ("set".equals(method.getName())) {
                Log.e("alarm_manager", "调用了mService.set()");

                try {
                    long interval = 0;
                    int alarmManagerTimeType = Integer.valueOf(args[1].toString());
                    long alarmManagerAtTime = Long.valueOf(args[2].toString());
                    Log.e("alarm_manager_interval", "alarmManagerTimeType:" +alarmManagerTimeType);
                    Log.e("alarm_manager_interval", "alarmManagerAtTime:" +alarmManagerAtTime);
                    switch (alarmManagerTimeType) {
                        case AlarmManager.RTC_WAKEUP:
                        case AlarmManager.RTC:
                            Log.e("alarm_manager_interval", "currentTimeMillis--RTC:" +System.currentTimeMillis());
                             interval = alarmManagerAtTime - System.currentTimeMillis();

                            break;
                        case AlarmManager.ELAPSED_REALTIME:
                        case AlarmManager.ELAPSED_REALTIME_WAKEUP:
                            Log.e("alarm_manager_interval", "currentTimeMillis--ELAPSED_REALTIME:" +SystemClock.elapsedRealtime());
                            interval = alarmManagerAtTime - SystemClock.elapsedRealtime();
                            break;
                    }
                    Log.e("alarm_manager_interval",interval+"-->" + interval/1000/60 );


                } catch (Exception e) {
                    e.printStackTrace();
                }


            }
            return method.invoke(iAlarmManagerObject, args);
        }
    }
```

#### hookAMS
对于Activity的启动过程，我们可以hook它的startActivity方法

``` java
    public void hookASM(){
        try {
            Class<?> activityManagerNativeClass = Class.forName("android.app.ActivityManagerNative");
            Field field = activityManagerNativeClass.getDeclaredField("gDefault");
            field.setAccessible(true);
            Object gDefault= field.get(null);

            Class<?> singletonClass = Class.forName("android.util.Singleton");
            Field mInstance = singletonClass.getDeclaredField("mInstance");
            mInstance.setAccessible(true);
            Object iActivityManagerObject = mInstance.get(gDefault);


            //开始动态代理，用代理对象替换掉真实的ActivityManager，
            AmsInvocationHandler amsInvocationHandler = new AmsInvocationHandler(iActivityManagerObject);
            Object proxy = Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(), iActivityManagerObject.getClass().getInterfaces(), amsInvocationHandler);

            //现在替换掉这个对象
            mInstance.set(gDefault, proxy);

        }catch (Exception e){
            e.printStackTrace();
        }
   }

    private class AmsInvocationHandler implements InvocationHandler {

        private Object iActivityManagerObject;
        private AmsInvocationHandler(Object iActivityManagerObject) {
            this.iActivityManagerObject = iActivityManagerObject;
        }

        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

            Log.i("hookASM", method.getName());
            //我要在这里搞点事情
            if ("startActivity".contains(method.getName())) {
                Log.e("hookASM","Activity已经开始启动");
            }
            return method.invoke(iActivityManagerObject, args);
        }
    }

```
既然我们能够接管startActivity方法，我们就可以伪造一个Intent去启动一个没有在清单文件中注册的Activity。
----
以上
