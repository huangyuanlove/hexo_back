---
title: Android7.0 MultiWindow
date: 2018-06-11 15:17:54
tags: [Android]
---
Android7.0推出了多窗口(分屏)模式，允许一个屏幕下显示两个应用，可以一边追剧一边看电子书，不过在小一点的手机屏幕上看起来可能有点鸡肋。但是在TV端(这个屏幕就比较大了，一般都是40寸起步)中，画中画模式可以让我们同时进行多个任务了。
<!-- more -->
#### 开启分屏
只要你编译使用sdk版本大于等于7.0就可以支持分屏了：`compileSdkVersion 25`，如果想要禁用分屏，只需要在`AndroidManifest.xml`添加属性：android:resizeableActivity="false"，这个属性使用于application和Activity标签。
7.0中默认是true。
除了分屏模式之外，还有自由模式(Freeform,常见于桌面设备，类似于windows的应用窗口，可以拖拽边缘改变大小)。

#### 生命周期
开启多窗口模式不会更改Activity的生命周期，
在多窗口模式中，在指定时间只有最近与用户交互过的 Activity 为活动状态。 该 Activity 将被视为顶级 Activity。 所有其他 Activity 虽然可见，但均处于暂停状态。 但是，这些已暂停但可见的 Activity 在系统中享有比不可见 Activity 更高的优先级。 如果用户与其中一个暂停的 Activity 交互，该 Activity 将恢复，而之前的顶级 Activity 将暂停。

例如：
![MultiWindow](/image/Android/MultiWindow/1.png)
在上图中，我先打开了上面的Activity，然后又打开了下面Gmail的Activity，这时下面的Activity处于可交互(顶级Activity)状态，上面的Activity虽然课件，但是处于暂停状态，这时按下back键是对下面Activity进行操作。当点击上面的Activity时，上面的Activity处于可交互状态，下面的Activity处于暂停状态。
PS：在多窗口模式中，用户仍可以看到处于暂停状态的应用。 应用在暂停状态下可能仍需要继续其操作。 例如，处于暂停模式但可见的视频播放应用应继续显示视频。 因此，我们建议播放视频的 Activity 不要暂停其 onPause() 处理程序中的视频。 应暂停 onStop() 中的视频，并恢复 onStart() 中的视频播放。

用户使用多窗口模式显示应用时，系统将通知 Activity 发生配置变更。 该变更与系统通知应用设备从纵向模式切换到横向模式时的 Activity 生命周期影响基本相同，但设备不仅仅是交换尺寸，而是会变更尺寸。您的 Activity 可以自行处理配置变更，或允许系统销毁 Activity，并以新的尺寸重新创建该 Activity。
给Activity加上如下配置可以保证切换成多屏模式或者画中画模式时Activity不会销毁重建。
android:configChanges="screenSize|smallestScreenSize|screenLayout|orientation"

#### 针对多窗口模式配置应用

如果应用支持 Android N，您可以对应用的 Activity 是否支持多窗口显示以及显示方式进行配置。 您可以在清单文件中设置属性，以控制大小和布局。 根 Activity 的属性设置适用于其任务栈中的所有 Activity。 例如，如果根 Activity 中 `android:resizeableActivity` 设定为 true，则任务栈中的所有 Activity 都将可以调整大小。
如果使用低于 Android N 版本的 SDK 构建多向应用，则用户在多窗口模式中使用应用时，系统将强制调整应用大小。 系统将显示对话框，提醒用户应用可能会发生异常。 系统不会调整定向应用的大小；如果用户尝试在多窗口模式下打开定向应用，应用将全屏显示。

** android:resizeableActivity **
在清单的 `<activity>` 或 `<application>` 节点中设置该属性，启用或禁用多窗口显示：

``` xml
    android:resizeableActivity=["true" | "false"] 
```
如果该属性设置为 true，Activity 将能以分屏和自由形状模式启动。 如果此属性设置为 false，Activity 将不支持多窗口模式。 如果该值为 false，且用户尝试在多窗口模式下启动 Activity，该 Activity 将全屏显示。
如果应用面向 Android N，但未对该属性指定值，则该属性的值默认设为 true。

** android:supportsPictureInPicture **
在清单文件的 `<activity>` 节点中设置该属性，指明 Activity 是否支持画中画显示。 如果 `android:resizeableActivity` 为 false，将忽略该属性。
``` xml
android:supportsPictureInPicture=["true" | "false"]
```

#### 布局属性
对于 Android N，`<layout>` 清单元素支持以下几种属性，这些属性影响 Activity 在多窗口模式中的行为：

`android:defaultWidth`
以自由形状模式启动时 Activity 的默认宽度。
`android:defaultHeight`
以自由形状模式启动时 Activity 的默认高度。
`android:gravity`
以自由形状模式启动时 Activity 的初始位置。请参阅 Gravity 参考资料，了解合适的值设置。
`android:minimalHeight、android:minimalWidth`
分屏和自由形状模式中 Activity 的最小高度和最小宽度。 如果用户在分屏模式中移动分界线，使 Activity 尺寸低于指定的最小值，系统会将 Activity 裁剪为用户请求的尺寸。
例如，以下节点显示了如何指定 Activity 在自由形状模式中显示时 Activity 的默认大小、位置和最小尺寸：

``` xml
<activity android:name=".MyActivity">
    <layout android:defaultHeight="500dp"
          android:defaultWidth="600dp"
          android:gravity="top|end"
          android:minimalHeight="450dp"
          android:minimalWidth="300dp" />
</activity>
```

#### 在多窗口模式中运行应用
** 多窗口模式中被禁用的功能 **
在设备处于多窗口模式中时，某些功能会被禁用或忽略，因为这些功能对与其他 Activity 或应用共享设备屏幕的 Activity 而言没有意义。 此类功能包括：
1. 某些系统 UI 自定义选项将被禁用；例如，在非全屏模式中，应用无法隐藏状态栏。
2. 系统将忽略对 android:screenOrientation 属性所作的更改。

** 多窗口变更通知和查询 **

1. Activity.isInMultiWindowMode()
调用该方法以确认 Activity 是否处于多窗口模式。
2. Activity.isInPictureInPictureMode()
调用该方法以确认 Activity 是否处于画中画模式。
> 画中画模式是多窗口模式的特例。 如果 myActivity.isInPictureInPictureMode() 返回 true，
> myActivity.isInMultiWindowMode() 也返回 true。
3. Activity.onMultiWindowModeChanged()
Activity 进入或退出多窗口模式时系统将调用此方法。 在 Activity 进入多窗口模式时，系统向该方法传递 true 值，在退出多窗口模式时，则传递 false 值。
4. Activity.onPictureInPictureModeChanged()
Activity 进入或退出画中画模式时系统将调用此方法。 在 Activity 进入画中画模式时，系统向该方法传递 true 值，在退出画中画模式时，则传递 false 值。

每个方法还有 Fragment 版本，例如 Fragment.isInMultiWindowMode()。

#### 进入画中画模式
如需在画中画模式中启动 Activity，请调用新方法 Activity.enterPictureInPictureMode()。 如果设备不支持画中画模式，则此方法无效。 如需了解详细信息，请参阅[画中画](https://developer.android.com/guide/topics/ui/picture-in-picture)文档。
https://developer.android.com/guide/topics/ui/picture-in-picture
#### 在多窗口模式中启动新 Activity
如果只是简单的开启Activity，和在非多窗口模式下是一致的。如果想要在当前Activity的旁边启动Activity，可以添加`FLAG_ACTIVITY_LAUNCH_ADJACENT`标志位(官方文档说是`Intent.FLAG_ACTIVITY_LAUNCH_TO_ADJACENT`,但在实际操作中这个标志位不存在)，传递此标志将请求以下行为：
1. 如果设备处于分屏模式，系统会尝试在启动系统的 Activity 旁创建新 Activity，这样两个 Activity 将共享屏幕。 系统并不一定能实现此操作，但如果可以，系统将使两个 Activity 处于相邻的位置。
2. 如果设备不处于分屏模式，则该标志无效。
如果设备处于自由形状模式，则在启动新 Activity 时，用户可通过调用 ActivityOptions.setLaunchBounds() 指定新 Activity 的尺寸和屏幕位置。 如果设备不处于多窗口模式，则该方法无效。
PS:如果在任务栈中启动 Activity，该 Activity 将替换屏幕上的 Activity，并继承其所有的多窗口属性。 如果要在多窗口模式中以单独的窗口启动新 Activity，则必须在新的任务栈中启动此 Activity。

#### 支持拖放
用户可以在两个 Activity 共享屏幕的同时在这两个 Activity 之间拖放数据 （在此之前，用户只能在一个 Activity 内部拖放数据）。 
1. android.view.DropPermissions 
令牌对象，负责指定对接收拖放数据的应用授予的权限。
2. View.startDragAndDrop()
View.startDrag() 的新别名。要启用跨 Activity 拖放，请传递新标志 View.DRAG_FLAG_GLOBAL。 如需对接收拖放数据的 Activity 授予 URI 权限，可根据情况传递新标志 View.DRAG_FLAG_GLOBAL_URI_READ 或 View.DRAG_FLAG_GLOBAL_URI_WRITE。
3. View.cancelDragAndDrop()
取消当前正在进行的拖动操作。只能由发起拖动操作的应用调用。
4. View.updateDragShadow()
替换当前正在进行的拖动操作的拖动阴影。只能由发起拖动操作的应用调用。
5. Activity.requestDropPermissions()
请求使用 DragEvent 中包含的 ClipData 传递的内容 URI 的权限。

#### 代码
** 在当前Activity旁边开启新界面 **
如下图所示，上面的界面是MainActivity，下面的Activity是在MainActivity中点击`获取实时天气`开启的界面
![MultiWindow](/image/Android/MultiWindow/2.png)
在MainActivity中

``` java
    findViewById(R.id.get_weather).setOnClickListener(new View.OnClickListener() {
        @Override
        public void onClick(View v) {
            Intent intent = new Intent(MainActivity.this,Main2Activity.class);
            intent.addFlags(Intent.FLAG_ACTIVITY_LAUNCH_ADJACENT | Intent.FLAG_ACTIVITY_NEW_TASK);
            startActivity(intent);
        }
    });
```
为什么需要 `FLAG_ACTIVITY_NEW_TASK`:
官方解释
>在同一个Activity返回栈中，打开一个新的Activity时，这个Activity将会继承上一个Activity所有和分屏模式有关的属性。如果你想要在一个独立的窗口以分屏模式打开一个新的Activity，那么必须新建一个Activity返回栈。

** 拖拽 **
首先在多窗口模式下打开新界面，如上面的代码所示。
在MainActivity中发起拖拽
``` java
final Button dragedButton = findViewById(R.id.drag_to_another);//拖拽按钮
        dragedButton.setTag("this is a test");

        dragedButton.setOnTouchListener(new View.OnTouchListener() {
            @Override
            public boolean onTouch(View v, MotionEvent event) {

                if(event.getAction() == MotionEvent.ACTION_DOWN){

                    ClipData.Item item = new ClipData.Item((CharSequence) dragedButton.getTag());
                    String[] mimeTypes = {ClipDescription.MIMETYPE_TEXT_PLAIN};
                    ClipData dragData = new ClipData(v.getTag().toString(), mimeTypes, item);
                    View.DragShadowBuilder shadow = new View.DragShadowBuilder(dragedButton);
                    /** startDragAndDrop是Android N SDK中的新方法，替代了以前的startDrag，flag需要设置为DRAG_FLAG_GLOBAL */
                    v.startDragAndDrop(dragData, shadow, null, View.DRAG_FLAG_GLOBAL);
                    return true;
                }
                else {
                    return false;
                }
            }
        });
```
在第二个界面中接收拖拽结果
``` java
final TextView textView = findViewById(R.id.show_drag_view_tag);
        textView.setText("拖拽到这里");
        textView.setOnDragListener(new View.OnDragListener() {
            @Override
            public boolean onDrag(View view, DragEvent dragEvent) {
                switch (dragEvent.getAction()) {
                    case DragEvent.ACTION_DRAG_STARTED:
                        Log.d(TAG, "Action is DragEvent.ACTION_DRAG_STARTED");
                        break;

                    case DragEvent.ACTION_DRAG_ENTERED:
                        Log.d(TAG, "Action is DragEvent.ACTION_DRAG_ENTERED");
                        break;

                    case DragEvent.ACTION_DRAG_EXITED:
                        Log.d(TAG, "Action is DragEvent.ACTION_DRAG_EXITED");
                        break;

                    case DragEvent.ACTION_DRAG_LOCATION:
                        break;

                    case DragEvent.ACTION_DRAG_ENDED:
                        Log.d(TAG, "Action is DragEvent.ACTION_DRAG_ENDED");
                        break;

                    case DragEvent.ACTION_DROP:
                        Log.d(TAG, "ACTION_DROP event");
                        /** 3.在这里显示接收到的结果 */
                        textView.setText(dragEvent.getClipData().getItemAt(0).getText());
                        break;

                    default:
                        break;
                }

                return true;
            }
        });
    }
```
将MainActivity中的拖拽按钮拖放的第二个界面中`拖拽到这里`view上之后
![MultiWindow](/image/Android/MultiWindow/3.png)
关于startDragAndDrop，官方参考文档在这里
[startDragAndDrop](https://developer.android.com/reference/android/view/View?hl=zh-cn#startDrag)
https://developer.android.com/reference/android/view/View?hl=zh-cn#startDrag

----
以上