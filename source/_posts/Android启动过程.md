---
title: Android启动过程
tags: [Android]
date: 2021-03-04 12:12:36
keywords: [Android启动过程,Zygote,ZygoteInit,服务启动]
---

## Android启动过程

从Zygote启动过程开始，省略掉了前面的解析.rc文件等步骤
<!--more-->

#### Zygote 启动过程


``` sequence
title:Zygote启动过程
participant App_main.cpp
participant AndroidRuntime.cpp
participant ZygoteInit.java
participant ZygoteConnection.java
participant Zygote.java

App_main.cpp -> App_main.cpp : 1:main
App_main.cpp -> AndroidRuntime.cpp: 2:start
AndroidRuntime.cpp -> AndroidRuntime.cpp:3:startVM
AndroidRuntime.cpp -> AndroidRuntime.cpp: 4:startReg
AndroidRuntime.cpp -> ZygoteInit.java: 5:main
ZygoteInit.java -> ZygoteInit.java : 6:registerZygoteSocket
ZygoteInit.java -> ZygoteInit.java : 7:preload
ZygoteInit.java -> ZygoteInit.java : 8:startSystemServer
ZygoteInit.java -> Zygote.java : 9:forkSystemServer
ZygoteInit.java -> ZygoteInit.java : 10:runSelectLoop
ZygoteInit.java -> ZygoteConnection.java: 11:runOnce
ZygoteConnection.java -> Zygote.java: 12:forkAndSpecialize

```

解释一下：
1. 解析init.zygote.rc中的参数，创建AppRuntime并调用AppRuntime.start()方法
2. 调用AndroidRuntime的startVM()方法创建虚拟机，再调用startReg()注册JNI函数
3. 通过JNI方式调用ZygoteInit.main()，第一次进入java世界
4. registerZygoteSocket()建立socket通道，zygote作为通信的服务端，用于响应客户端请求
5. preload()预加载通用类、drawable和color资源、openGL以及共享库以及WebView，用于提高app启动效率
6. zygote完成大部分工作，接下来再通过startSystemServer(),fork得力助手system-server进行，也是上层framework的运行载体
7. zygote功成身退，调用runSelectLoop()，随时待命，当接收到请求创建新进行请求时立即唤醒并执行相应工作。



#### SystemService 启动流程

![start_system_server](/image/Android/aosp/start_system_server.png)

上图前4个步骤运行在Zygote进行，从第五步开始是运行在新创建的system_server,这是fork机制实现的。

RuntimeInit.java 中 invokeStaticMain 方法通过创建并抛出异常 ZygoteInit.MethodAndArgsCaller，在 ZygoteInit.java 中的 main()方法会捕捉该异常，并调用 caller.run()，再通过反射便会调用到 SystemServer.main()方法，在该方法中创建SystemServer对象并执行run方法。
在该方法中执行了如下操作
* 设置系统时间

  ``` java
  if (System.currentTimeMillis() < EARLIEST_SUPPORTED_TIME) {
                Slog.w(TAG, "System clock is before 1970; setting to 1970.");
                SystemClock.setCurrentTimeMillis(EARLIEST_SUPPORTED_TIME);
            }
  ```

  

* 变更虚拟机库文件

  ``` java
  SystemProperties.set("persist.sys.dalvik.vm.lib.2", VMRuntime.getRuntime().vmLibrary());
  ```

  

* Mmmmmm... more memory!（清除vm内存增长限制）

  ``` java
  VMRuntime.getRuntime().clearGrowthLimit();
  ```

  

* Prepare the main looper thread (this thread)

   ``` java
   android.os.Process.setThreadPriority(
                android.os.Process.THREAD_PRIORITY_FOREGROUND);
            android.os.Process.setCanSelfBackground(false);
            Looper.prepareMainLooper();
            Looper.getMainLooper().setSlowLogThresholdMs(
                    SLOW_DISPATCH_THRESHOLD_MS, SLOW_DELIVERY_THRESHOLD_MS);
   ```
   
* Initialize native services
	``` java
	System.loadLibrary("android_servers");
	```
  
* 检测上次关机过程是否失败，该方法可能不会返回

   ``` java
   performPendingShutdown();
   ```

* Initialize the system context

   ``` java
   createSystemContext();//这里需要区分system_server进程和app进程：http://gityuan.com/2017/04/02/android-application/
   ```

*  Create the system service manager

  ``` java 
  mSystemServiceManager = new SystemServiceManager(mSystemContext);
  //将 mSystemServiceManager 添加到本地服务的成员 sLocalServiceObjects
  LocalServices.addService(SystemServiceManager.class, mSystemServiceManage
  r);
  ```
  
* Start services
	
	startBootstrapServices();
	
	> 该方法所创建的服务:DeviceIdentifiersPolicyService、ActivityManagerService、PowerManagerService、RecoverySystemService、LightsService、DisplayManagerService、PackageManagerService、 UserManagerService、 SensorService.
	
	startCoreServices();
	
	> 启动服务 BatteryService，UsageStatsService，WebViewUpdateService。
	
	startOtherServices();
	
	> 这里启动的服务挺多的，捡主要的写一下：VibratorService、NetworkManagementService、IpSecService、NetworkStatsService、WindowManagerService、InputManagerService、AlarmManagerService

SystemServer 启动各种服务中最后的一个环节便是 AMS.systemReady()，到此, System_server 主线程的启动工作总算完成, 进入 Looper.loop()状态,等待 其他线程通过 handler 发送消息到主线再处理。