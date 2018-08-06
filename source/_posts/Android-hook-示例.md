---
title: Android hook--示例
date: 2018-08-06 15:46:06
tags: [Android]
keywords: AndroidHook,反射,动态代理
photos:
  - /image/Android/hook/guide.jpg
---
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

```
