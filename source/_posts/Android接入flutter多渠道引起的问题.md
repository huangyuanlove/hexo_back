---
title: Android接入flutter多渠道引起的问题
tags: []
photos:
  - null
date: 2019-06-19 11:02:19
keywords:
---

A/flutter: [FATAL:flutter/shell/common/shell.cc(218)] Check failed: vm. Must be able to initialize the VM.

``` groovy
                // String flavor = ""
                // try{
                //     String tempFlavor = project.rootProject.ext.app.flavor
                //     if(tempFlavor!=null && !tempFlavor.empty){
                //         flavor = tempFlavor
                //     }
                    
                // } catch (Exception e){
                //     println e
                // }

                // Task mergeAssets

                // if(flavor!=null && !flavor.empty){
                //     mergeAssets = project.tasks.findByPath(":${mainModuleName}:merge${flavor.capitalize()}${variant.name.capitalize()}Assets")
                // }else{
                //     mergeAssets = project.tasks.findByPath(":${mainModuleName}:merge${variant.name.capitalize()}Assets")
                // }
```

``` crash
2019-06-19 18:00:22.200 27335-27783/me.chunyu.ChunyuDoctor E/CrashReport: #++++++++++Record By Bugly++++++++++#
2019-06-19 18:00:22.200 27335-27783/me.chunyu.ChunyuDoctor E/CrashReport: # You can use Bugly(http:\\bugly.qq.com) to get more Crash Detail!
2019-06-19 18:00:22.200 27335-27783/me.chunyu.ChunyuDoctor E/CrashReport: # PKG NAME: me.chunyu.ChunyuDoctor
2019-06-19 18:00:22.200 27335-27783/me.chunyu.ChunyuDoctor E/CrashReport: # APP VER: 8.6.0
2019-06-19 18:00:22.202 27335-27783/me.chunyu.ChunyuDoctor E/CrashReport: # LAUNCH TIME: 2019-06-19 18:00:12
2019-06-19 18:00:22.202 27335-27783/me.chunyu.ChunyuDoctor E/CrashReport: # CRASH TYPE: NATIVE_CRASH
2019-06-19 18:00:22.203 27335-27783/me.chunyu.ChunyuDoctor E/CrashReport: # CRASH TIME: 2019-06-19 18:00:22
2019-06-19 18:00:22.203 27335-27783/me.chunyu.ChunyuDoctor E/CrashReport: # CRASH PROCESS: me.chunyu.ChunyuDoctor
2019-06-19 18:00:22.203 27335-27783/me.chunyu.ChunyuDoctor E/CrashReport: # CRASH THREAD: main(2)
2019-06-19 18:00:22.203 27335-27783/me.chunyu.ChunyuDoctor E/CrashReport: # REPORT ID: d88b5d1a-cbb0-4920-b41c-d8ea8eb804bd
2019-06-19 18:00:22.203 27335-27783/me.chunyu.ChunyuDoctor E/CrashReport: # CRASH DEVICE: MI 5 UNROOT
2019-06-19 18:00:22.204 27335-27783/me.chunyu.ChunyuDoctor E/CrashReport: # RUNTIME AVAIL RAM:1411100672 ROM:11107233792 SD:11107233792
2019-06-19 18:00:22.204 27335-27783/me.chunyu.ChunyuDoctor E/CrashReport: # RUNTIME TOTAL RAM:2824822784 ROM:26300211200 SD:26300211200
2019-06-19 18:00:22.204 27335-27783/me.chunyu.ChunyuDoctor E/CrashReport: # EXCEPTION FIRED BY UNKNOWN_USER me.chunyu.ChunyuDoctor(27335)
2019-06-19 18:00:22.204 27335-27783/me.chunyu.ChunyuDoctor E/CrashReport: # CRASH STACK: 
2019-06-19 18:00:22.204 27335-27783/me.chunyu.ChunyuDoctor E/CrashReport: SIGABRT
    0x6ac7
    #00    pc 0004b10c    /system/lib/libc.so (tgkill+12) [armeabi-v7a::d7f479b7abcbff3bc09ca6d100c44333]
    #01    pc 0001a9a3    /system/lib/libc.so (abort+54) [armeabi-v7a::d7f479b7abcbff3bc09ca6d100c44333]
    #02    pc 00af85ab    /data/app/me.chunyu.ChunyuDoctor-n6cyA4a8iSyIl6wZ_2ghlw==/lib/arm/libflutter.so [armeabi-v7a::a12434e0b53806a35730000001000000]
    #03    pc 00b197cf    /data/app/me.chunyu.ChunyuDoctor-n6cyA4a8iSyIl6wZ_2ghlw==/lib/arm/libflutter.so [armeabi-v7a::a12434e0b53806a35730000001000000]
    #04    pc 00aeaf45    /data/app/me.chunyu.ChunyuDoctor-n6cyA4a8iSyIl6wZ_2ghlw==/lib/arm/libflutter.so [armeabi-v7a::a12434e0b53806a35730000001000000]
    #05    pc 00af0507    /data/app/me.chunyu.ChunyuDoctor-n6cyA4a8iSyIl6wZ_2ghlw==/lib/arm/libflutter.so [armeabi-v7a::a12434e0b53806a35730000001000000]
    #06    pc 00aef79b    /data/app/me.chunyu.ChunyuDoctor-n6cyA4a8iSyIl6wZ_2ghlw==/lib/arm/libflutter.so [armeabi-v7a::a12434e0b53806a35730000001000000]
    #07    pc 003d9d29    /system/lib/libart.so (ExecuteMterpImpl+37417) [armeabi-v7a::d2078ba50169919802b98e67fdc80c58]
    #08    pc 003d5fe1    /system/lib/libart.so (ExecuteMterpImpl+21729) [armeabi-v7a::d2078ba50169919802b98e67fdc80c58]
    #09    pc 003da555    /system/lib/libart.so (art_quick_invoke_stub+228) [armeabi-v7a::d2078ba50169919802b98e67fdc80c58]
    #10    pc 000a0e1d    /system/lib/libart.so (_ZN3art9ArtMethod6InvokeEPNS_6ThreadEPjjPNS_6JValueEPKc+140) [armeabi-v7a::d2078ba50169919802b98e67fdc80c58]
    #11    pc 001e6e53    /system/lib/libart.so (_ZN3art11interpreter34ArtInterpreterToCompiledCodeBridgeEPNS_6ThreadEPNS_9ArtMethodEPKNS_7DexFile8CodeItemEPNS_11ShadowFrameEPNS_6JValueE+238) [armeabi-v7a::d2078ba50169919802b98e67fdc80c58]
    #12    pc 001e2403    /system/lib/libart.so (_ZN3art11interpreter6DoCallILb0ELb0EEEbPNS_9ArtMethodEPNS_6ThreadERNS_11ShadowFrameEPKNS_11InstructionEtPNS_6JValueE+574) [armeabi-v7a::d2078ba50169919802b98e67fdc80c58]
    #13    pc 003c0d75    /system/lib/libart.so (MterpInvokeDirect+360) [armeabi-v7a::d2078ba50169919802b98e67fdc80c58]
    #14    pc 003c8394    /system/lib/libart.so (artInvokeSuperTrampolineWithAccessCheck+3599) [armeabi-v7a::d2078ba50169919802b98e67fdc80c58]
    #15    pc 001c9985    /system/lib/libart.so (_ZN3art22IndirectReferenceTable10VisitRootsEPNS_11RootVisitorERKNS_8RootInfoE+16) [armeabi-v7a::d2078ba50169919802b98e67fdc80c58]
    #16    pc 001ced07    /system/lib/libart.so (_ZN3art11interpreter33ArtInterpreterToInterpreterBridgeEPNS_6ThreadEPKNS_7DexFile8CodeItemEPNS_11ShadowFrameEPNS_6JValueE+142) [armeabi-v7a::d2078ba50169919802b98e67fdc80c58]
    #17    pc 001e23ed    /system/lib/libart.so (_ZN3art11interpreter6DoCallILb0ELb0EEEbPNS_9ArtMethodEPNS_6ThreadERNS_11ShadowFrameEPKNS_11InstructionEtPNS_6JValueE+552) [armeabi-v7a::d2078ba50169919802b98e67fdc80c58]
    #18    pc 003bfebf    /system/lib/libart.so (MterpInvokeVirtual+446) [armeabi-v7a::d2078ba50169919802b98e67fdc80c58]
    #19    pc 003c8294    /system/lib/libart.so (artInvokeSuperTrampolineWithAccessCheck+3343) [armeabi-v7a::d2078ba50169919802b98e67fdc80c58]
    #20    pc 001c9985    /system/lib/libart.so (_ZN3art22IndirectReferenceTable10VisitRootsEPNS_11RootVisitorERKNS_8RootInfoE+16) [armeabi-v7a::d2078ba50169919802b98e67fdc80c58]
    #21    pc 001ced07    /system/lib/libart.so (_ZN3art11interpreter33ArtInterpreterToInterpreterBridgeEPNS_6ThreadEPKNS_7DexFile8CodeItemEPNS_11ShadowFrameEPNS_6JValueE+142) [armeabi-v7a::d2078ba50169919802b98e67fdc80c58]
    #22    pc 001e23ed    /system/lib/libart.so (_ZN3art11interpreter6DoCallILb0ELb0EEEbPNS_9ArtMethodEPNS_6ThreadERNS_11ShadowFrameEPKNS_11InstructionEtPNS_6JValueE+552) [armeabi-v7a::d2078ba50169919802b98e67fdc80c58]
    #23    pc 003c0d75    /system/lib/libart.so (MterpInvokeDirect+360) [armeabi-v7a::d2078ba50169919802b98e67fdc80c58]
    #24    pc 003c8394    /system/lib/libart.so (artInvokeSuperTrampolineWithAccessCheck+3599) [armeabi-v7a::d2078ba50169919802b98e67fdc80c58]
    #25    pc 001c9985    /system/lib/libart.so (_ZN3art22IndirectReferenceTable10VisitRootsEPNS_11RootVisitorERKNS_8RootInfoE+16) [armeabi-v7a::
2019-06-19 18:00:22.204 27335-27783/me.chunyu.ChunyuDoctor E/CrashReport: #++++++++++++++++++++++++++++++++++++++++++#
```

https://www.jianshu.com/p/273c462896c3