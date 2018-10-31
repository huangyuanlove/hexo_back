---
title: 对应用中Alarm优化
date: 2018-10-31 11:00:13
tags: [Android]
keywords: Alarm,Alarm次数优化
---
起因：华为应用市场反馈Alarm唤醒次数过多，需要优化。
未优化之前通过华为的DevEco进行功耗测试，在Mate 9上每小时唤醒71次，在p10上每小时唤醒62，妥妥的手机没办法进入休眠状态，而他们的标准是每个应用每小时唤醒不超过20次。
![测试结果](/image/Android/alarm/未优化.png)
<!--more-->

#### 查找使用Alarm的代码
1. Alarm的使用需要AlarmManager，而AlarmManager的获取是通过getSystemService(ALARM_SERVICE)获取到的，找到了某个类中中的doMyJob方法，其中每隔5分钟唤醒一次设备。其他的Alarm服务间隔时间比较长，有的是一天唤醒一次，有的是定时上午九点唤醒，这些闹钟全部加起来一小时也不会超过15次。
2. 使用 adb shell dumpsys alarm | grep "包名" 查看系统中的存在哪些Alarm并通过项目包名过滤掉不是自己工程的Alarm，发现了 ` *walarm*:包名.service_alive_alarm_filter`,查看相关代码发现每隔十分钟唤醒一次设备，一方面是为了保活，一方面是为了检查计步传感器是否开启，因为有些设备在息屏之后会关闭计步传感器，这时候我们需要切换到自己的计步算法，通过加速度传感器来计算运动步数。
3. 调整这两个Alarm的唤醒频率， 提交到DevEco进行测试，结果如下：
![测试结果](/image/Android/alarm/第一次优化.png)

#### 查找三方使用的Alarm
1. 仔细查看使用adb shell找到的alarm，发现了这货 `*walarm*:AlarmNioTaskSchedule.包名`，如果是我们自己在工程里面设置的Alarm，一般是以自己的项目包名命名的，接着查dump出来的信息，找到了两个唤醒间隔时间非常短的Alarm，粗略估计每个的唤醒间隔在4分钟半。下面日志中when就是alarm唤醒的时间点
``` log
ELAPSED_WAKEUP #0: Alarm{a359f35 type 2 when 1911290944 包名}
      operation=PendingIntent{7551fca: PendingIntentRecord{eb70106 包名 startService}}
ELAPSED_WAKEUP #1: Alarm{80150f type 2 when 1911660925 包名}
      operation=PendingIntent{58db99c: PendingIntentRecord{35972a0 包名 broadcastIntent}}
```

通过PendingIntent和PendingIntentRecord的编号，结合  adb shell dumpsys activity intents | grep "包名"
找到设置该Alarm的包名：
``` log
#4: PendingIntentRecord{eb70106 包名 startService}
      uid=10415 packageName=包名 type=startService flags=0x0
      requestIntent=act=com.qiyukf.desk.ACTION.KEEP_ALIVE cmp=包名/com.qiyukf.nimlib.service.NimService (has extras)
 
#8: PendingIntentRecord{35972a0 包名 broadcastIntent}
      uid=10415 packageName=包名 type=broadcastIntent flags=0x0
      requestIntent=act=com.qiyukf.nim.ACTION.ALARM.REPEATING cmp=包名/com.qiyukf.nimlib.service.NimReceiver
```
由于是三方的包，没办法修改代码来完成，只能用其他方法。
翻看设置Alarm的源码发现：由于设置Alarm的时候需要调用set或者setRepeating方法，最终都会调用setImpl方法，最后通过进程间通信，调IAlarmManager的set方法。
IAlarmManager的实现类是AlarmManagerService,只需要替换掉这个类的实例，在调用它的set方法时，替换掉alarm的时间间隔
就可以减少一部分alarm的唤醒。
由于hook的时间越早越好，于是选在Application初始化的时候进行hook，在Application的onCreate方法中：
``` java
//替换IAlarmService
try {
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

} catch (Exception e) {
    Log.e("alarm_manager", "替换出错");
    e.printStackTrace();
}
class AlarmManagerInvocationHandler implements InvocationHandler {

    private Object iAlarmManagerObject;

    private AlarmManagerInvocationHandler(Object iAlarmManagerObject) {
        this.iAlarmManagerObject = iAlarmManagerObject;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {


        Log.i("alarm_manager", method.getName());
        //小于五小时间隔的设置为五小时
        if ("set".equals(method.getName())) {
            Log.e("alarm_manager", "调用了mService.set()");
            int minAlarmInterval = 300 * 60 * 1000;
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
                            if(interval < minAlarmInterval){
                                interval = minAlarmInterval;
                                args[2] = System.currentTimeMillis() + interval;
                            }
                        break;
                    case AlarmManager.ELAPSED_REALTIME:
                    case AlarmManager.ELAPSED_REALTIME_WAKEUP:
                        Log.e("alarm_manager_interval", "currentTimeMillis--ELAPSED_REALTIME:" +SystemClock.elapsedRealtime());
                        interval = alarmManagerAtTime - SystemClock.elapsedRealtime();
                        if(interval < minAlarmInterval){
                            interval = minAlarmInterval;
                            args[2] = SystemClock.elapsedRealtime() + interval;
                        }
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
打包提交DevEco进行测试，结果如下：
![测试结果](/image/Android/alarm/第二次优化.png)

#### 当前结果

从最初的60-70次优化到现在的19-23次，但还是不符合华为的标准每小时小于20次，原因在于有一个Alarm没有找到所属的程序包，不知道是在哪里设置的
``` log
RTC_WAKEUP #0: Alarm{c5091d9 type 0 when 1534851817125 包名}
      tag=*walarm*:AlarmTaskSchedule.包名
      operation=PendingIntent{194679e: PendingIntentRecord{e513b24 包名 broadcastIntent}}
*walarm*:AlarmTaskSchedule.包名
```
这个Alarm粗略估计每小时唤醒次数在15-20次之间，既然通过hook底层代码的方法没有拦截下它，估计是通过jni在更加底层进行的操作，目前还在排查