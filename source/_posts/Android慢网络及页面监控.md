---
title: Android慢网络及页面监控
tags: [Android]
date: 2021-06-19 14:55:29
keywords: [Android慢网络监控,用户行为监控,OkHttp拦截器]
---

最近在搞Android应用大盘监控，目前需要监控的是慢网络请求及页面的打开和关闭。由于应用中使用的OkHttp进行网络请求，着重看了一下OkHttp源码，还有别人写的一些总结。对于页面的打开关闭行为，我们可以搞个BaseActivity或者在Application中注册生命周期回调就好了。问题在于慢网络监控需要实时上报，而用户行为监控则需要本地落盘保存，在需要的时候再上报，所以如何落盘保存则是一个问题，为此了解了java IO和mmap。

<!--more-->

## 慢网络监控

**嫌长不看直接看结论，代码在这一段的最后。**

在构建`OkHttpClient`对象时加入`eventListener`即可，如果会有多个异步网络同时请求，就添加`eventListenerFactory`。不论是不是同时会有多个异步网络请求，使用`eventListenerFactory`差距不大。

**比较啰嗦的详解篇**

我们先来看下网络基础内容：

* OSI七层模型和实际应用中的五层模型
* 什么是Http
* 请求方式、报文结构
* TCP的三次握手、四次挥手
* 一次完整的http请求过程
* Http不同版本的差异、优缺点
* http和https的区别

相关的知识点放在这里了，别人已经写得比较全面了，再抄一遍也没啥意思 :smiley:

[面试官的这份HTTP灵魂追问你Hold住吗？](https://juejin.cn/post/6877362691350986766)

[关于HTTP请求你需要知道的一切](https://www.jianshu.com/p/b0aa797608c0)

当我们使用OkHttp进行网络请求的时候，过程一般是这样的：

1. 构建`OkHttpClient`
2. 构建`Request`
3. 构建`Call`
4. 使用`Call`对象进行网络请求

``` java
OkHttpClient okHttpClient = new OkHttpClient.Builder()
        .eventListenerFactory(NetworkListener.get())
		.addInterceptor(new Interceptor() {
		    @NotNull
		    @Override
		    public Response intercept(@NotNull Chain chain) throws IOException {
		        Log.e(TAG,"addInterceptor before proceed");
		        Response response = chain.proceed(chain.request());
		        Log.e(TAG,"addInterceptor after proceed ");
		        return response;
		    }
		})
		.addNetworkInterceptor(new Interceptor() {
		    @NotNull
		    @Override
		    public Response intercept(@NotNull Chain chain) throws IOException {
		        Log.e(TAG,"addNetworkInterceptor before proceed");
		        Response response = chain.proceed(chain.request());
		        Log.e(TAG,"addNetworkInterceptor after proceed ");
		        return response;
		    }
		})
        .build();

Request request = new Request.Builder()
        .url(".....")
        .build();
Call call = okHttpClient.newCall(request);
call.enqueue(new Callback() {
    @Override
    public void onFailure(@NotNull Call call, @NotNull IOException e) {
        
    }

    @Override
    public void onResponse(@NotNull Call call, @NotNull Response response) throws IOException {

    }
});
```

流程图如下：

![OkHttp流程图](/image/Android/okhttp/OkHttp流程图.png)

最经典的应该是拦截器部分了，网络对拦截器的分析也挺多了，可以自己翻一下源码总结一下，我也写不出花来 :hushed:

![OkHttp流程图](/image/Android/okhttp/OkHttp拦截器.png)

图片我是用drawio画的，源文件放在了github上，路径 https://github.com/huangyuanlove/huangyuanlove.github.io/tree/master/image/Android/okhttp

下面的源码

``` java

public class NetworkListener extends EventListener {

    private static final String TAG = "NetworkListener";
    public static Factory get(){
        return new Factory() {
            @NotNull
            @Override
            public EventListener create(@NotNull Call call) {
                return new NetworkListener();
            }
        };
    }

    @Override
    public void callStart(@NotNull Call call) {
        super.callStart(call);
        Log.e(TAG,"-------callStart---requestId-----"+mRequestId);
    }

//重写的N个回调方法
  .
  .
  .
  .
  .

    @Override
    public void callFailed(@NotNull Call call, @NotNull IOException ioe) {
        super.callFailed(call, ioe);
        ioe.printStackTrace();
        Log.e(TAG, "callFailed");

    }
}

```

我们可以在这些回调方法中做时间统计，超过指定时长则认为是慢网络请求。

## 页面打开关闭监控

结论：在自定义的application中调用一下`registerActivityLifecycleCallbacks(ActivityLifecycleCallbacks callback);`就好。在每个Activity的生命周期执行的时候都会回调`callback`.

#### 收集信息

在Application中注册一下生命周期回调接口:`registerActivityLifecycleCallbacks(ActivityLifecycleCallbacks callback);`并重写各种回调方法，记录对应的时间戳+类名。由于生命周期都是在主线程回调，我们不必担心多线程竞争问题。搞个list直接存。

#### 落盘保存

1. 在什么时机保存
2. 如何保存

对于问题1，我们在了解Handler机制的时候，提到过在`MessageQueue`里面有个开发过程中不常用的对象:`IdleHandle`r，查看源码和注释我们得知，是在线程的MessageQueue中没有消息的时候，会去调用这个类的"queueIdle()"方法，并且该方法返回true时，不会被移除队列。

我们在Application中向MainLooper的MessageQueue中添加一个IdleHandler

``` java
Looper.myQueue().addIdleHandler(new MessageQueue.IdleHandler() {
            @Override
            public boolean queueIdle() {
                Log.e(TAG,"queueIdle,当前线程名称" + Thread.currentThread().getName()+",线程id:" +Thread.currentThread().getId());
                BehaviorRepository.getInstance(MyApplication.this).storeLifeEvent();
                return true;
            }
        });
```



对于如何保存，一开始考虑的是写到数据库，因为有事务。后来结合业务发现并不需要这么搞，直接写文件就好，需要的时候直接上传文件到服务器，那么我们如何去写文件？想到了mmap这货。当使用mmap方式写文件失败再考虑使用普通java io。于是有了下面：下面代码来源于：https://www.cnblogs.com/rustfisher/p/11551372.html

``` java

public class LogWriter {

    private static final String TAG = "activityLifeRecorder";


    // 注意申请SD卡读写权限
    private static String logFileDir;

    private static String fileName;


    private static HandlerThread handlerThread;
    private static Handler writerHandler;

    private static final int LOG_FILE_GROW_SIZE = 1024 * 10; // log文件每次增长的大小
    private static long gCurrentLogPos = 0;                  // log文件当前写到的位置 - 注意要单线程处理

    /**
     * 使用前必须调用此方法进行准备
     */
    public static void init(Context context){
        gCurrentLogPos = 0;
        logFileDir = context.getCacheDir() + File.separator + "logs";
        if (null == handlerThread) {
            handlerThread = new HandlerThread("LL");
            handlerThread.start();
        }
        writerHandler = new Handler(handlerThread.getLooper());


        //可以保存本次打开的日志，只保存三五次打开的日志
        fileName = "_" +  new SimpleDateFormat("yyyy-MM-dd",Locale.CHINA).format(System.currentTimeMillis()) + ".txt";
        Log.d(TAG, "[prepare] file: " + fileName);
    }

    public static String getFileName() {
        return fileName;
    }

    // 退出
    public static void quit() {
        if (writerHandler != null) {
            writerHandler.removeCallbacksAndMessages(null);
        }
        if (handlerThread != null) {
            handlerThread.quit();
        }
    }




    public static void writeToFile(String content){
        if (writerHandler != null) {
            writerHandler.post(new WriteRunnable(content));
        }
    }



    static class WriteRunnable implements Runnable {
        String  content;

        WriteRunnable( String content) {
            this.content = content;
        }

        @Override
        public void run() {
            try {
                File dir = new File(logFileDir);
                if (!dir.exists()) {
                    boolean mk = dir.mkdirs();
                    Log.d(TAG, "make dir " + mk);
                }
                File eFile = new File(logFileDir + File.separator + fileName);
                byte[] strBytes = content.getBytes();
                try {
                    RandomAccessFile randomAccessFile = new RandomAccessFile(eFile, "rw");
                    MappedByteBuffer mappedByteBuffer;
                    final int inputLen = strBytes.length;
                    if (!eFile.exists()) {
                        boolean nf = eFile.createNewFile();
                        Log.d(TAG, "new log file " + nf);
                        mappedByteBuffer = randomAccessFile.getChannel().map(FileChannel.MapMode.READ_WRITE, gCurrentLogPos, LOG_FILE_GROW_SIZE);
                    } else {
                        mappedByteBuffer = randomAccessFile.getChannel().map(FileChannel.MapMode.READ_WRITE, gCurrentLogPos, inputLen);
                    }
                    if (mappedByteBuffer.remaining() < inputLen) {
                        mappedByteBuffer = randomAccessFile.getChannel().map(FileChannel.MapMode.READ_WRITE, gCurrentLogPos, LOG_FILE_GROW_SIZE + inputLen);
                    }
                    mappedByteBuffer.put(strBytes);
                    gCurrentLogPos += inputLen;
                } catch (Exception e) {
                    Log.e(TAG, "WriteRunnable run: ", e);
                    if (!eFile.exists()) {
                        boolean nf = eFile.createNewFile();
                        Log.d(TAG, "new log file " + nf);
                    }
                    FileOutputStream os = new FileOutputStream(eFile, true);
                    os.write(content.getBytes());
                    os.flush();
                    os.close();
                }
            } catch (Exception e) {
                e.printStackTrace();
                Log.e(TAG, "写log文件出错: ", e);
            }
        }
    }

}

```



----

以上
