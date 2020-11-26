---
title: JetPack中的WorkManager
tags: [Android]
date: 2020-02-25 22:31:04
keywords: Android,Jetpack,WorkManager
---

2018年谷歌I/O 发布了一系列辅助android开发者的实用工具，合称Jetpack，以帮助开发者构建出色的 Android 应用。
这次发布的 Android Jetpack 组件覆盖以下 4 个方面：Architecture、Foundation、Behavior 以及 UI。
该系列博客介绍一下Jetpack中常用组件，本篇介绍LiveData、ViewModel、LifeCycle。最后借助于https://github.com/android/sunflower 来写一个完整的应用

<!--more-->

原文 https://developer.android.com/topic/libraries/architecture/workmanager?hl=zh-cn

使用 WorkManager API 可以轻松地调度即使在应用退出或设备重启时仍应运行的可延迟异步任务。

**主要功能**：

- 最高向后兼容到 API 14
  - 在运行 API 23 及以上级别的设备上使用 JobScheduler
  - 在运行 API 14-22 的设备上结合使用 BroadcastReceiver 和 AlarmManager
- 添加网络可用性或充电状态等工作约束
- 调度一次性或周期性异步任务
- 监控和管理计划任务
- 将任务链接起来
- 确保任务执行，即使应用或设备重启也同样执行任务
- 遵循低电耗模式等省电功能

WorkManager 旨在用于**可延迟**运行（即不需要立即运行）并且在应用退出或设备重启时必须能够可靠运行的任务。例如：

- 向后端服务发送日志或分析数据
- 定期将应用数据与服务器同步

### 基础

相关类：

* Worker
 任务的执行者，是一个抽象类，需要继承它实现要执行的任务。

* WorkRequest
 指定让哪个 Woker 执行任务，指定执行的环境，执行的顺序等。
 要使用它的子类 OneTimeWorkRequest 或 PeriodicWorkRequest。

* WorkManager
 管理任务请求和任务队列，发起的 WorkRequest 会进入它的任务队列。

* WorkStatus
 包含有任务的状态和任务的信息，以 LiveData 的形式提供给观察者。

### 入门指南
#### 添加依赖 
`implementation "androidx.work:work-runtime:2.3.2"`
#### 创建后台任务
创建一个类，继承自`androidx.work.Worker`，并覆写其`public Result doWork()`方法，从 `doWork()` 返回的 `Result`会通知 WorkManager 任务是否：

- 已成功完成：`Result.success()`
- 已失败：`Result.failure()`
- 需要稍后重试：`Result.retry()`

``` java
public class UploadWorker extends Worker {

    public UploadWorker(@NonNull Context context, @NonNull WorkerParameters workerParams) {
        super(context, workerParams);
    }

    @NonNull
    @Override
    public Result doWork() {
        Log.e("UploadWorker input",getInputData().getString("time"));
				//do something
  	    return Result.success();
    }
}

```

#### 配置运行任务的方式和时间

`Worker` 定义工作单元，[`WorkRequest`](https://developer.android.com/reference/androidx/work/WorkRequest?hl=zh-cn) 则定义工作的运行方式和时间。任务可以是一次性的，也可以是周期性的。对于一次性 `WorkRequest`，请使用 [`OneTimeWorkRequest`](https://developer.android.com/reference/androidx/work/OneTimeWorkRequest?hl=zh-cn)，对于周期性工作，请使用 [`PeriodicWorkRequest`](https://developer.android.com/reference/androidx/work/PeriodicWorkRequest?hl=zh-cn)。

``` java
OneTimeWorkRequest uploadWorkRequest = new OneTimeWorkRequest.Builder(UploadWorker.class)
            .build()
    
```

#### 将任务提交给系统

``` java
WorkManager.getInstance(myContext).enqueue(uploadWorkRequest);
```

执行 Worker 的确切时间取决于 `WorkRequest` 中使用的约束以及系统优化。



### 定义工作请求

#### 工作约束

可以向工作添加 `Constraints`，以指明工作何时可以运行。

``` java
Constraints constraints=    new Constraints.Builder()
  .setRequiredNetworkType(NetworkType.CONNECTED)  // 网络状态
  .setRequiresBatteryNotLow(true)                 // 不在电量不足时执行
  .setRequiresCharging(true)                      // 在充电时执行
  .setRequiresStorageNotLow(true)                 // 不在存储容量不足时执行
  //.setRequiresDeviceIdle(true)                    // 在待机状态下执行，需要 API 23
  .build();
OneTimeWorkRequest compressionWork = new OneTimeWorkRequest.Builder(UploadWorker.class)
  .setConstraints(constraints)
  .build();
```

如果指定了多个约束，任务将仅在满足所有约束时才会运行。

如果在任务运行期间某个约束不再得到满足，则 WorkManager 将停止工作器。当约束继续得到满足时，系统将重新尝试执行该任务。

#### 初始延迟

如果Worker没有约束，或者工作加入队列时所有约束均已得到满足，则系统可能会选择立即运行任务。如果不希望任务立即运行，则可以将工作指定为在经过最短的初始延迟后启动。

下面的示例展示了如何将任务设置为在加入队列后至少经过 10 分钟再运行。

``` java
OneTimeWorkRequest uploadWorkRequest = new OneTimeWorkRequest.Builder(UploadWorker.class)
  .setInitialDelay(10, TimeUnit.MINUTES)
  .build();
```

#### 重试和退避政策

如果需要让 WorkManager 重新尝试执行任务，可以从工作器返回 [`Result.retry()`](https://developer.android.com/reference/androidx/work/ListenableWorker.Result?hl=zh-cn#retry())。

然后，系统会根据默认的退避延迟时间和政策重新调度工作。退避延迟时间指定重试工作前的最短等待时间。[退避政策](https://developer.android.com/reference/androidx/work/BackoffPolicy?hl=zh-cn)定义了在后续重试的尝试过程中，退避延迟时间随时间以怎样的方式增长；默认情况下按 [`EXPONENTIAL`](https://developer.android.com/reference/androidx/work/BackoffPolicy?hl=zh-cn) 延长。

以下是自定义退避延迟时间和政策的示例。

``` java
OneTimeWorkRequest uploadWorkRequest =
  new OneTimeWorkRequest.Builder(UploadWorker.class)
  .setBackoffCriteria(BackoffPolicy.LINEAR,
  OneTimeWorkRequest.MIN_BACKOFF_MILLIS,
  TimeUnit.MILLISECONDS)
  .build();
```

#### 定义任务的输入/输出

可以通过使用`androidx.work.Data`来定义输入或者输出

在`UploadWorker`中

``` java
@Override
public Result doWork() {
  Log.e("UploadWorker input",getInputData().getString("time"));
  Data outputData = new Data.Builder()
    .putLong("timestamp",System.currentTimeMillis())
    .build();
  return Result.success(outputData); 
}
```

在调用时 

``` java
Data data = new Data.Builder()
  .putString("time",simpleDateFormat.format(new Date()))
  .build();

OneTimeWorkRequest uploadWorkRequest = new OneTimeWorkRequest.Builder(UploadWorker.class)
  .setConstraints(constraints)
  .setInputData(data) //定义输入
  .build();
WorkManager.getInstance(this).enqueue(uploadWorkRequest);
        WorkManager.getInstance(this).getWorkInfoByIdLiveData(uploadWorkRequest.getId()).observe(this, new Observer<WorkInfo>() {
            @Override
            public void onChanged(WorkInfo workInfo) {

                 long timeStamp = workInfo.getOutputData().getLong("timestamp",0);
                Log.e("UploadWorker output",simpleDateFormat.format(timeStamp) +"--->"+workInfo.getState());
            }
        });
```

非常重要的一点：

**按照设计，`Data` 对象应该很小，值可以是字符串、基元类型或数组变体。如果需要将更多数据传入和传出工作器，应该将数据放在其他位置，例如 Room 数据库。Data 对象的大小上限为 10KB。**

#### 标记工作

可以通过为任意 [`WorkRequest`](https://developer.android.com/reference/androidx/work/WorkRequest?hl=zh-cn) 对象分配标记字符串，按逻辑对任务进行分组。这样就可以对使用特定标记的所有任务执行操作。

例如，[`WorkManager.cancelAllWorkByTag(String)`](https://developer.android.com/reference/androidx/work/WorkManager?hl=zh-cn#cancelAllWorkByTag(java.lang.String)) 会取消使用特定标记的所有任务，而 [`WorkManager.getWorkInfosByTagLiveData(String)`](https://developer.android.com/reference/androidx/work/WorkManager?hl=zh-cn#getWorkInfosByTagLiveData(java.lang.String)) 会返回 [`LiveData`](https://developer.android.com/reference/androidx/lifecycle/LiveData?hl=zh-cn) 和具有该标记的所有任务的状态列表。

``` java
OneTimeWorkRequest cacheCleanupTask =
  new OneTimeWorkRequest.Builder(CacheCleanupWorker.class)
  .setConstraints(constraints)
  .addTag("cleanup")
  .build();
```

#### 工作状态

- 如果有尚未完成的[前提性工作](https://developer.android.com/topic/libraries/architecture/workmanager/how-to/chain-work?hl=zh-cn)，则工作处于 [`BLOCKED`](https://developer.android.com/reference/androidx/work/WorkInfo.State?hl=zh-cn#BLOCKED) `State`。
- 如果工作能够在满足 [`Constraints`](https://developer.android.com/reference/androidx/work/Constraints?hl=zh-cn) 和时机条件后立即运行，则被视为处于 [`ENQUEUED`](https://developer.android.com/reference/androidx/work/WorkInfo.State?hl=zh-cn#ENQUEUED) 状态。
- 当工作器在活跃地执行时，其处于 [`RUNNING`](https://developer.android.com/reference/androidx/work/WorkInfo.State?hl=zh-cn#RUNNING) `State`。
- 如果工作器返回 [`Result.success()`](https://developer.android.com/reference/androidx/work/ListenableWorker.Result?hl=zh-cn#success())，则被视为处于 [`SUCCEEDED`](https://developer.android.com/reference/androidx/work/WorkInfo.State?hl=zh-cn#SUCCEEDED) 状态。这是一种终止 `State`；只有 [`OneTimeWorkRequest`](https://developer.android.com/reference/androidx/work/OneTimeWorkRequest?hl=zh-cn) 可以进入这种 `State`。
- 相反，如果工作器返回 [`Result.failure()`](https://developer.android.com/reference/androidx/work/ListenableWorker.Result?hl=zh-cn#failure())，则被视为处于 [`FAILED`](https://developer.android.com/reference/androidx/work/WorkInfo.State?hl=zh-cn#FAILED) 状态。这也是一个终止 `State`；只有 [`OneTimeWorkRequest`](https://developer.android.com/reference/androidx/work/OneTimeWorkRequest?hl=zh-cn) 可以进入这种 `State`。所有依赖工作也会被标记为 `FAILED`，并且不会运行。
- 当明确[取消](https://developer.android.com/topic/libraries/architecture/workmanager/how-to/cancel-stop-work?hl=zh-cn)尚未终止的 `WorkRequest` 时，它会进入 [`CANCELLED`](https://developer.android.com/reference/androidx/work/WorkInfo.State?hl=zh-cn#CANCELLED) `State`。所有依赖工作也会被标记为 `CANCELLED`，并且不会运行。

#### 观察工作

将工作加入队列后，可以通过 WorkManager 检查其状态。相关信息在 [`WorkInfo`](https://developer.android.com/reference/androidx/work/WorkInfo?hl=zh-cn) 对象中提供，包括工作的 `id`、标签、当前 [`State`](https://developer.android.com/reference/androidx/work/WorkInfo.State?hl=zh-cn) 和任何输出数据。

可以通过以下三种方式之一来获取 `WorkInfo`：

- 对于特定的 [`WorkRequest`](https://developer.android.com/reference/androidx/work/WorkRequest?hl=zh-cn)，可以利用 [`WorkManager.getWorkInfoById(UUID)`](https://developer.android.com/reference/androidx/work/WorkManager?hl=zh-cn#getWorkInfoById(java.util.UUID)) 或 [`WorkManager.getWorkInfoByIdLiveData(UUID)`](https://developer.android.com/reference/androidx/work/WorkManager?hl=zh-cn#getWorkInfoByIdLiveData(java.util.UUID)) 来通过 `WorkRequest` `id` 检索其 `WorkInfo`。
- 对于指定的[标记](https://developer.android.com/topic/libraries/architecture/workmanager/how-to/define-work?hl=zh-cn#tag)，可以利用 [`WorkManager.getWorkInfosByTag(String)`](https://developer.android.com/reference/androidx/work/WorkManager?hl=zh-cn#getWorkInfosByTag(java.lang.String)) 或 [`WorkManager.getWorkInfosByTagLiveData(String)`](https://developer.android.com/reference/androidx/work/WorkManager?hl=zh-cn#getWorkInfosByTagLiveData(java.lang.String)) 检索所有匹配的 `WorkRequest` 的 `WorkInfo` 对象。
- 对于[唯一工作名称](https://developer.android.com/topic/libraries/architecture/workmanager/how-to/unique-work?hl=zh-cn)，可以利用 [`WorkManager.getWorkInfosForUniqueWork(String)`](https://developer.android.com/reference/androidx/work/WorkManager?hl=zh-cn#getWorkInfosForUniqueWork(java.lang.String)) 或 [`WorkManager.getWorkInfosForUniqueWorkLiveData(String)`](https://developer.android.com/reference/androidx/work/WorkManager?hl=zh-cn#getWorkInfosForUniqueWorkLiveData(java.lang.String)) 检索所有匹配的 `WorkRequest` 的 `WorkInfo` 对象。

利用每个方法的 [`LiveData`](https://developer.android.com/topic/libraries/architecture/livedata?hl=zh-cn) 变量，可以通过注册监听器来观察 `WorkInfo` 的变化。

``` java
WorkManager.getInstance(myContext).getWorkInfoByIdLiveData(uploadWorkRequest.getId())
  .observe(lifecycleOwner, new Observer<WorkInfo>() {
    @Override
    public void onChanged(@Nullable WorkInfo workInfo) {
      if (workInfo != null && workInfo.state == WorkInfo.State.SUCCEEDED) {
        displayMessage("Work finished!")
      }
    }
  });
```

#### 更新进度

对于使用 [`ListenableWorker`](https://developer.android.com/reference/androidx/work/ListenableWorker?hl=zh-cn) 或 [`Worker`](https://developer.android.com/reference/androidx/work/Worker?hl=zh-cn) 的 Java 开发者，[`setProgressAsync()`](https://developer.android.com/reference/androidx/work/ListenableWorker?hl=zh-cn#setProgressAsync(androidx.work.Data)) API 会返回 `ListenableFuture`；更新进度是异步过程，因为更新过程包括将进度信息存储在数据库中。

此示例展示了一个简单的 `ProgressWorker`。该 `Worker` 启动时将进度设置为 0，完成时将进度值更新为 100。

``` java
public class ProgressWorker extends Worker {

  private static final String PROGRESS = "PROGRESS";
  private static final long DELAY = 1000L;

  public ProgressWorker(
    @NonNull Context context,
    @NonNull WorkerParameters parameters) {
    super(context, parameters);
    // Set initial progress to 0
    setProgressAsync(new Data.Builder().putInt(PROGRESS, 0).build());
  }

  @NonNull
  @Override
  public Result doWork() {
    try {
      // Doing work.
      Thread.sleep(DELAY);
    } catch (InterruptedException exception) {
      // ... handle exception
    }
    // Set progress to 100 after you are done doing your work.
    setProgressAsync(new Data.Builder().putInt(PROGRESS, 100).build());
    return Result.success();
  }
}
```

#### 观察进度

```java
WorkManager.getInstance(getApplicationContext())
  .getWorkInfoByIdLiveData(uploadWorkRequest.getId())
  .observe(lifecycleOwner, new Observer<WorkInfo>() {
    @Override
    public void onChanged(@Nullable WorkInfo workInfo) {
      if (workInfo != null) {
        Data progress = workInfo.getProgress();
        int value = progress.getInt(PROGRESS, 0)
          // Do something with progress
      }
    }
  });
    
```

### 链接工作

就像动画的执行可以确定顺序、依赖一样，Worker的执行也可以这么操作

可以使用 [WorkManager](https://developer.android.com/reference/androidx/work/WorkManager?hl=zh-cn) 创建工作链并为其排队。工作链用于指定多个关联任务并定义这些任务的运行顺序。当需要以特定的顺序运行多个任务时，这尤其有用。

要创建工作链，可以使用 [`WorkManager.beginWith(OneTimeWorkRequest)`](https://developer.android.com/reference/androidx/work/WorkManager?hl=zh-cn#beginWith(androidx.work.OneTimeWorkRequest)) 或 [`WorkManager.beginWith(List)`](https://developer.android.com/reference/androidx/work/WorkManager?hl=zh-cn#beginWith(java.util.List))，这会返回 [`WorkContinuation`](https://developer.android.com/reference/androidx/work/WorkContinuation?hl=zh-cn) 实例。

然后，可以通过 `WorkContinuation` 使用 [`WorkContinuation.then(OneTimeWorkRequest)`](https://developer.android.com/reference/androidx/work/WorkContinuation?hl=zh-cn#then(androidx.work.OneTimeWorkRequest)) 或 [`WorkContinuation.then(List)`](https://developer.android.com/reference/androidx/work/WorkContinuation?hl=zh-cn#then(java.util.List)) 来添加从属 `OneTimeWorkRequest`。

每次调用 `WorkContinuation.then(...)` 都会返回一个新的 `WorkContinuation` 实例。如果添加了 `OneTimeWorkRequest` 的 `List`，这些请求可能会并行运行。

最后，可以使用 [`WorkContinuation.enqueue()`](https://developer.android.com/reference/androidx/work/WorkContinuation?hl=zh-cn#enqueue()) 方法为 `WorkContinuation` 链排队。

```java
WorkManager.getInstance(myContext)
  .beginWith(Arrays.asList(filter1, filter2, filter3)) 
  .then(compress)
  .then(upload)
  .enqueue();
```

这里的filter是OneTimeWorkRequest实例对象

#### Input Merger

在使用 `OneTimeWorkRequest` 链时，父级 `OneTimeWorkRequest` 的输出将作为输入传递给子级。因此在上面的示例中，`filter1`、`filter2` 和 `filter3` 的输出将作为输入传递给 `compress` 请求。

为了管理来自多个父级 `OneTimeWorkRequest` 的输入，WorkManager 使用 [`InputMerger`](https://developer.android.com/reference/androidx/work/InputMerger?hl=zh-cn)。

WorkManager 提供两种不同类型的 `InputMerger`：

- [`OverwritingInputMerger`](https://developer.android.com/reference/androidx/work/OverwritingInputMerger?hl=zh-cn) 会尝试将所有输入中的所有键添加到输出中。如果发生冲突，它会覆盖先前设置的键。
- [`ArrayCreatingInputMerger`](https://developer.android.com/reference/androidx/work/ArrayCreatingInputMerger?hl=zh-cn) 会尝试合并输入，并在必要时创建数组。

对于上面的示例，假设我们要保留所有图像滤镜的输出，则应使用 `ArrayCreatingInputMerger`。

```java
OneTimeWorkRequest compress =
  new OneTimeWorkRequest.Builder(CompressWorker.class)
  .setInputMerger(ArrayCreatingInputMerger.class)
  .setConstraints(constraints)
  .build();
```

#### 链接和工作状态

创建 `OneTimeWorkRequest` 链时，需要注意以下几点：

- 从属 `OneTimeWorkRequest` 仅在其所有父级 `OneTimeWorkRequest` 都成功完成（即返回 `Result.success()`）时才会被解除阻塞（变为 `ENQUEUED` 状态）。
- 如果有任何父级 `OneTimeWorkRequest` 失败（返回 `Result.failure()`），则所有从属 `OneTimeWorkRequest` 也会被标记为 `FAILED`。
- 如果有任何父级 `OneTimeWorkRequest` 被取消，则所有从属 `OneTimeWorkRequest` 也会被标记为 `CANCELLED`

### 取消和停止工作

如果不再需要运行先前加入队列的作业，则可以申请取消。最简单的方法是使用其 `id` 并调用 [`WorkManager.cancelWorkById(UUID)`](https://developer.android.com/reference/androidx/work/WorkManager?hl=zh-cn#cancelWorkById(java.util.UUID)) 来取消单个 WorkRequest：

```java
WorkManager.cancelWorkById(workRequest.getId());
```

在后台，WorkManager 会检查工作的 [`State`](https://developer.android.com/reference/androidx/work/WorkInfo.State?hl=zh-cn)。如果工作已经[完成](https://developer.android.com/reference/androidx/work/WorkInfo.State?hl=zh-cn#isFinished())，则不会发生任何变化。否则，其状态将更改为 [`CANCELLED`](https://developer.android.com/reference/androidx/work/WorkInfo.State?hl=zh-cn#CANCELLED)，之后就不会运行这个工作。任何[依赖于这项工作](https://developer.android.com/topic/libraries/architecture/workmanager/how-to/chain-work?hl=zh-cn)的 [`WorkRequests`](https://developer.android.com/reference/androidx/work/WorkRequest?hl=zh-cn) 的状态也将变为 `CANCELLED`。

此外，如果工作当前的状态为 [`RUNNING`](https://developer.android.com/reference/androidx/work/WorkInfo.State?hl=zh-cn#RUNNING)，则工作器也会收到对 [`ListenableWorker.onStopped()`](https://developer.android.com/reference/androidx/work/ListenableWorker?hl=zh-cn#onStopped()) 的调用。替换此方法以处理任何可能的清理操作。

也可以使用 [`WorkManager.cancelAllWorkByTag(String)`](https://developer.android.com/reference/androidx/work/WorkManager?hl=zh-cn#cancelAllWorkByTag(java.lang.String)) 按标记取消 WorkRequest。请注意，此方法会取消所有具有此标记的工作。此外，还可以使用 [`WorkManager.cancelUniqueWork(String)`](https://developer.android.com/reference/androidx/work/WorkManager?hl=zh-cn#cancelUniqueWork(java.lang.String)) 取消具有唯一名称的所有工作。

#### 停止正在运行的工作器

WorkManager 停止正在运行的工作器可能有几种不同的原因：

- 明确要求取消它（例如，通过调用 `WorkManager.cancelWorkById(UUID)` 取消）。
- 如果是[唯一工作](https://developer.android.com/topic/libraries/architecture/workmanager/how-to/unique-work?hl=zh-cn)，使用 [`ExistingWorkPolicy`](https://developer.android.com/reference/androidx/work/ExistingWorkPolicy?hl=zh-cn) [`REPLACE`](https://developer.android.com/topic/libraries/architecture/workmanager/how-to/'reference/androidx/work/ExistingWorkPolicy#REPLACE) 明确地将新的 `WorkRequest` 加入队列。旧的 `WorkRequest` 会立即被视为已终止。
- 工作约束已不再得到满足。
- 系统出于某种原因指示您的应用停止工作。如果超过 10 分钟的执行期限，可能会发生这种情况。系统将工作安排在稍后重试。

在这些情况下，会收到对 [`ListenableWorker.onStopped()`](https://developer.android.com/reference/androidx/work/ListenableWorker?hl=zh-cn#onStopped()) 的调用。如果操作系统决定关闭应用，应执行清理工作并以协作方式完成工作器。例如，应该在此时或者尽早关闭数据库和文件的打开句柄。此外，如果想要确认系统是否已经停止你应用，都可以调用 [`ListenableWorker.isStopped()`](https://developer.android.com/reference/androidx/work/ListenableWorker?hl=zh-cn#isStopped())。即使通过在调用 `onStopped()` 后返回 [`Result`](https://developer.android.com/reference/androidx/work/Result?hl=zh-cn) 来指示工作已完成，WorkManager 都会忽略该 `Result`，因为工作器已经被视为停止。

### 重复性工作

应用有时可能需要定期运行某些任务。例如，您可能要定期备份数据、下载应用中的新鲜内容，或者上传日志到服务器。 [`PeriodicWorkRequest`](https://developer.android.com/reference/androidx/work/PeriodicWorkRequest?hl=zh-cn) 用于这种需要定期执行的任务。需要注意的是`PeriodicWorkRequest` 无法和其他任务进行链接。

```java
Constraints constraints = new Constraints.Builder().setRequiresCharging(true).build();

PeriodicWorkRequest saveRequest =
  new PeriodicWorkRequest.Builder(SaveImageFileWorker.class, 1, TimeUnit.HOURS)
  .setConstraints(constraints)
  .build();

WorkManager.getInstance(myContext)
  .enqueue(saveRequest);
```

### 唯一工作

唯一工作是一个概念性非常强的术语，可确保一次只有一个具有特定名称的工作链。与 `id` 不同的是，唯一名称是人类可读的，由开发者指定，而不是由 WorkManager 自动生成。与[标记](https://developer.android.com/topic/libraries/architecture/workmanager/how-to/define-work?hl=zh-cn#tag)不同，唯一名称仅与“一个”工作链关联。

可以通过调用 [`WorkManager.enqueueUniqueWork(String, ExistingWorkPolicy, OneTimeWorkRequest)`](https://developer.android.com/reference/androidx/work/WorkManager?hl=zh-cn#enqueueUniqueWork(java.lang.String, androidx.work.ExistingWorkPolicy, androidx.work.OneTimeWorkRequest)) 或 [`WorkManager.enqueueUniquePeriodicWork(String, ExistingPeriodicWorkPolicy, PeriodicWorkRequest)`](https://developer.android.com/reference/androidx/work/WorkManager?hl=zh-cn#enqueueUniquePeriodicWork(java.lang.String, androidx.work.ExistingPeriodicWorkPolicy, androidx.work.PeriodicWorkRequest)) 创建唯一工作序列。第一个参数是唯一名称 - 这是我们用来标识 `WorkRequest` 的键。第二个参数是冲突解决策略，它指定了如果已经存在一个具有该唯一名称的未完成工作链，WorkManager 应该如何处理：

- 取消现有工作链，并将其 [`REPLACE`](https://developer.android.com/reference/androidx/work/ExistingWorkPolicy?hl=zh-cn#REPLACE) 为新工作链。
- [`KEEP`](https://developer.android.com/reference/androidx/work/ExistingWorkPolicy?hl=zh-cn#KEEP) 现有序列并忽略您的新请求。
- 将新序列 [`APPEND`](https://developer.android.com/reference/androidx/work/ExistingWorkPolicy?hl=zh-cn#APPEND) 到现有序列，在现有序列的最后一个任务完成后运行新序列的第一个任务。您不能将 `APPEND` 与 `PeriodicWorkRequest` 一起使用。

当有不能够多次排队的任务时，唯一工作将非常有用。例如，如果应用需要将其数据同步到网络，可能需要对一个名为“sync”的序列进行排队，并指定当已经存在具有该名称的序列时，应该忽略新的任务。当需要逐步构建一个长任务链时，也可以利用唯一工作序列。例如，照片编辑应用可能允许用户撤消一长串操作。其中的每一项撤消操作可能都需要一些时间来完成，但必须按正确的顺序执行。在这种情况下，应用可以创建一个“撤消”链，并根据需要将每个撤消操作附加到该链上。

最后，如果需要创建一个唯一工作链，可以使用 [`WorkManager.beginUniqueWork(String, ExistingWorkPolicy, OneTimeWorkRequest)`](https://developer.android.com/reference/androidx/work/WorkManager?hl=en#beginUniqueWork(java.lang.String, androidx.work.ExistingWorkPolicy, androidx.work.OneTimeWorkRequest)) 代替 `beginWith()`。



### 测试

https://developer.android.com/topic/libraries/architecture/workmanager/how-to/testing?hl=zh-cn

#### 简介与设置

[WorkManager](https://developer.android.com/topic/libraries/architecture/workmanager?hl=zh-cn) 提供了一个 `work-testing` 工件，可以协助进行 Android 插桩测试中的工作器单元测试。

要使用 `work-testing` 工件，您应该将它作为 `androidTestImplementation` 依赖项添加到 `build.gradle` 中。

#### 概念

`work-testing` 为测试模式提供了一种特殊的 WorkManager 实现，它通过使用 [`WorkManagerTestInitHelper`](https://developer.android.com/reference/androidx/work/testing/WorkManagerTestInitHelper?hl=zh-cn) 来初始化。

`work-testing` 工件还提供了 [`SynchronousExecutor`](https://developer.android.com/reference/androidx/work/testing/SynchronousExecutor?hl=zh-cn)，让您可以更加轻松地以同步方式编写测试，而无需处理多个线程、锁定或锁存。

以下示例展示了如何将所有这些类一起使用。

```java
@RunWith(AndroidJUnit4.class)
public class BasicInstrumentationTest {
  @Before
  public void setup() {
    Context context = InstrumentationRegistry.getTargetContext();
    Configuration config = new Configuration.Builder()
      // Set log level to Log.DEBUG to
      // make it easier to see why tests failed
      .setMinimumLoggingLevel(Log.DEBUG)
      // Use a SynchronousExecutor to make it easier to write tests
      .setExecutor(new SynchronousExecutor())
      .build();

    // Initialize WorkManager for instrumentation tests.
    WorkManagerTestInitHelper.initializeTestWorkManager(
      context, config);
  }
}
```

#### 构造测试

现在 WorkManager 已在测试模式中初始化，您可以测试您的工作器了。

假设您有一个 `EchoWorker`，它需要一些 `inputData` 并简单地将其输入复制（回显）到其 `outputData`。

```java
public class EchoWorker extends Worker {
  public EchoWorker(Context context, WorkerParameters parameters) {
    super(context, parameters);
  }

  @NonNull
  @Override
  public Result doWork() {
    Data input = getInputData();
    if (input.size() == 0) {
      return Result.failure();
    } else {
      return Result.success(input);
    }
  }
}
```

#### 基本测试

以下是一个对 `EchoWorker` 进行测试的 Android 插桩测试。这里的要点是，在测试模式中测试 `EchoWorker` 与在真实应用中使用 `EchoWorker` 非常相似。

```java
@Test
public void testSimpleEchoWorker() throws Exception {
  // Define input data
  Data input = new Data.Builder()
    .put(KEY_1, 1)
    .put(KEY_2, 2)
    .build();

  // Create request
  OneTimeWorkRequest request =
    new OneTimeWorkRequest.Builder(EchoWorker.class)
    .setInputData(input)
    .build();

  WorkManager workManager = WorkManager.getInstance(getApplicationContext());
  // Enqueue and wait for result. This also runs the Worker synchronously
  // because we are using a SynchronousExecutor.
  workManager.enqueue(request).getResult().get();
  // Get WorkInfo and outputData
  WorkInfo workInfo = workManager.getWorkInfoById(request.getId()).get();
  Data outputData = workInfo.getOutputData();
  // Assert
  assertThat(workInfo.getState(), is(WorkInfo.State.SUCCEEDED));
  assertThat(outputData, is(input));
}
```

我们来编写另一个测试，它将确保在 `EchoWorker` 没有获得任何输入数据时，其预期的 `Result` 为 `Result.failure()`。

```java
@Test
public void testEchoWorkerNoInput() throws Exception {
  // Create request
  OneTimeWorkRequest request =
    new OneTimeWorkRequest.Builder(EchoWorker.class)
    .build();

  WorkManager workManager = WorkManager.getInstance(getApplicationContext());
  // Enqueue and wait for result. This also runs the Worker synchronously
  // because we are using a SynchronousExecutor.
  workManager.enqueue(request).getResult().get();
  // Get WorkInfo
  WorkInfo workInfo = workManager.getWorkInfoById(request.getId()).get();
  // Assert
  assertThat(workInfo.getState(), is(WorkInfo.State.FAILED));
}
```

#### 模拟约束、延迟和定期工作

`WorkManagerTestInitHelper` 为您提供了一个 [`TestDriver`](https://developer.android.com/reference/androidx/work/testing/TestDriver?hl=zh-cn) 实例，可用于模拟 `initialDelay`、`ListenableWorker` 满足 `Constraint` 的条件，以及 `PeriodicWorkRequest` 的间隔。

测试初始延迟

`Worker` 可以具有初始延迟。要测试含有 `EchoWorker` 的 `initialDelay`，而不必在测试中等待 `initialDelay`，您可以使用 `TestDriver` 将 `WorkRequest` 初始延迟标记为已满足条件。

```java
@Test
public void testWithInitialDelay() throws Exception {
  // Define input data
  Data input = new Data.Builder()
    .put(KEY_1, 1)
    .put(KEY_2, 2)
    .build();

  // Create request
  OneTimeWorkRequest request = new OneTimeWorkRequest.Builder(EchoWorker.class)
    .setInputData(input)
    .setInitialDelay(10, TimeUnit.SECONDS)
    .build();

  WorkManager workManager = WorkManager.getInstance(myContext);
  // Get the TestDriver
  TestDriver testDriver = WorkManagerTestInitHelper.getTestDriver();
  // Enqueue
  workManager.enqueue(request).getResult().get();
  // Tells the WorkManager test framework that initial delays are now met.
  testDriver.setInitialDelayMet(request.getId());
  // Get WorkInfo and outputData
  WorkInfo workInfo = workManager.getWorkInfoById(request.getId()).get();
  Data outputData = workInfo.getOutputData();
  // Assert
  assertThat(workInfo.getState(), is(WorkInfo.State.SUCCEEDED));
  assertThat(outputData, is(input));
}
```

#### 测试约束

`TestDriver` 也可用于利用 `setAllConstraintsMet` 将约束标记为已满足条件。以下示例展示了如何测试含有约束的 `Worker`。

```java
@Test
public void testWithConstraints() throws Exception {
  // Define input data
  Data input = new Data.Builder()
    .put(KEY_1, 1)
    .put(KEY_2, 2)
    .build();

  // Define constraints
  Constraints constraints = new Constraints.Builder()
    .setRequiresDeviceIdle(true)
    .build();

  // Create request
  OneTimeWorkRequest request = new OneTimeWorkRequest.Builder(EchoWorker.class)
    .setInputData(input)
    .setConstraints(constraints)
    .build();

  WorkManager workManager = WorkManager.getInstance(myContext);
  TestDriver testDriver = WorkManagerTestInitHelper.getTestDriver();
  // Enqueue
  workManager.enqueue(request).getResult().get();
  // Tells the testing framework that all constraints are met.
  testDriver.setAllConstraintsMet(request.getId());
  // Get WorkInfo and outputData
  WorkInfo workInfo = workManager.getWorkInfoById(request.getId()).get();
  Data outputData = workInfo.getOutputData();
  // Assert
  assertThat(workInfo.getState(), is(WorkInfo.State.SUCCEEDED));
  assertThat(outputData, is(input));
}
```

#### 测试定期工作

```java
@Test
public void testPeriodicWork() throws Exception {
  // Define input data
  Data input = new Data.Builder()
    .put(KEY_1, 1)
    .put(KEY_2, 2)
    .build();

  // Create request
  PeriodicWorkRequest request =
    new PeriodicWorkRequest.Builder(EchoWorker.class, 15, MINUTES)
    .setInputData(input)
    .build();

  WorkManager workManager = WorkManager.getInstance(myContext);
  TestDriver testDriver = WorkManagerTestInitHelper.getTestDriver();
  // Enqueue
  workManager.enqueue(request).getResult().get();
  // Tells the testing framework the period delay is met
  testDriver.setPeriodDelayMet(request.getId());
  // Get WorkInfo and outputData
  WorkInfo workInfo = workManager.getWorkInfoById(request.getId()).get();
  // Assert
  assertThat(workInfo.getState(), is(WorkInfo.State.ENQUEUED));
}
```



----

以上