---
title: Android hook--示例
date: 2018-08-06 15:46:06
tags: [Android]
keywords: AndroidHook,反射,动态代理
photos:
  - /image/Android/hook/guide.jpg
---


hookAlarmManager
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

            class AlarmManagerInvocationHandler implements InvocationHandler {

        private Object iAlarmManagerObject;

        private AlarmManagerInvocationHandler(Object iAlarmManagerObject) {
            this.iAlarmManagerObject = iAlarmManagerObject;
        }

        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {


            Log.i("alarm_manager", method.getName());
            //我要在这里搞点事情
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


hookOnClick 和 hookOnLongClick
``` java
hookOnClickListerner(playSound);
hookOnLongCLickListerner(getWeather);


private void hookOnLongCLickListerner(View view) {
        try {
            //拿到mListenerInfo ，可以通过getListenerInfo方法
            Class<?> clazzView = Class.forName("android.view.View");
            Method getListenerInfoMethod = clazzView.getDeclaredMethod("getListenerInfo");
            getListenerInfoMethod.setAccessible(true);
            Object listenerInfo = getListenerInfoMethod.invoke(view);

            Class<?> clazz = Class.forName("android.view.View$ListenerInfo");
            Field field = clazz.getDeclaredField("mOnLongClickListener");
            field.setAccessible(true);
            View.OnLongClickListener raw =(View.OnLongClickListener) field.get(listenerInfo);
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
                raw.onLongClick(v);
            }
            return false;
        }
    }


    private void hookOnClickListerner(View view) {


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


hookAMS
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


            //开始动态代理，用代理对象替换掉真实的ActivityManager，瞒天过海
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

            Log.i("HookUtil", method.getName());
            //我要在这里搞点事情
            if ("startActivity".contains(method.getName())) {
                Log.e("HookUtil","Activity已经开始启动");
                Log.e("HookUtil","小弟到此一游！！！");
            }
            return method.invoke(iActivityManagerObject, args);
        }
    }

```
