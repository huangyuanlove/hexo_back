---
title: Android N 应用内更新
date: 2017-03-28 10:30:31
tags: [Android爬坑之旅,Android]
keywords: 应用内更新
---
继之前跪在Android M的动态权限之后，最近又跪在了Android N的`StrictMode`上了。所以啊，要对技术持有敬畏的态度。
场景如下：
我司内部员工使用的APP需要有应用内更新的功能，意思就是在应用内下载最新版本的应用并且调起安装界面。
方案：由于每次从新打开app都需要重新登录，那就在登录界面加上检查更新的接口请求，后台对比当前版本App的VersionCode 和 数据库存储的VersionCode对比，如果需要更新，则返回最新版本软件的下载地址，前端进行下载安装。
当前端解析出下载地址后，弹出提示框，下载或者取消。点击下载则开启线程下载，同时在界面上显示下载进度，下载完成后，调起安装界面进行安装。
代码很简单，这里放出不涉及我司业务的代码：
<!--more-->
``` java
private void downLoadAPK() {
        downLoadThread = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    URL url = new URL(downLoadUrl);

                    HttpURLConnection conn = (HttpURLConnection) url
                            .openConnection();
                    conn.connect();
                    int length = conn.getContentLength();
                    InputStream is = conn.getInputStream();

                    File file = new File("");
                    if (!file.exists()) {
                        file.mkdir();
                    }
                    File apkFile = new File(saveFilePath);
                    if (apkFile.exists()) {
                        apkFile.delete();
                    }
                    FileOutputStream fos = new FileOutputStream(apkFile);

                    int count = 0;
                    byte buf[] = new byte[1024];

                    // 点击取消就停止下载.
                    while (!interceptFlag) {
                        int numread = is.read(buf);
                        count += numread;
                        progress = (int) (((float) count / length) * 100);
                        // 更新进度
                        getHandler().sendEmptyMessage(DOWN_UPDATE);
                        if (numread <= 0) {
                            // 下载完成通知安装
                            getHandler().sendEmptyMessage(DOWN_OVER);
                            interceptFlag = false;
                        }
                        fos.write(buf, 0, numread);
                    }
                    fos.close();
                    is.close();
                } catch (Exception e) {
                    e.printStackTrace();
                }

            }
        });
        downLoadThread.start();
    }
```
以上为下载文件的代码，逻辑很简单，起一个新线程，使用HttpURLConnection进行文件下载。
``` java
    private void installAPK(String filePath) {
        File apkFile = new File(filePath);
        Intent intent = new Intent(Intent.ACTION_VIEW);
        if (!apkFile.exists()) {
            return;
        }
        intent.setDataAndType(Uri.fromFile(apkFile), "application/vnd.android.package-archive");
        context.startActivity(intent);
    }
```
以上代码是刚开始写的安装软件的代码，在Android N 以下运行正常，但是在Android N上却爆出了如下错误，
``` java
android.os.FileUriExposedException: file: exposed beyond app through Intent.getData()
	at android.os.StrictMode.onFileUriExposed(StrictMode.java:1799)
	at android.net.Uri.checkFileUriExposed(Uri.java:2346)
	at android.content.Intent.prepareToLeaveProcess(Intent.java:8949)
	at android.content.Intent.prepareToLeaveProcess(Intent.java:8908)
	at android.app.Instrumentation.execStartActivity(Instrumentation.java:1519)
	at android.app.ContextImpl.startActivity(ContextImpl.java:829)
	at android.app.ContextImpl.startActivity(ContextImpl.java:806)
	at android.content.ContextWrapper.startActivity(ContextWrapper.java:366)
	at com.mmuu.travel.service.ui.LoginFrg.installAPK(LoginFrg.java:349)
	at com.mmuu.travel.service.ui.LoginFrg.access$200(LoginFrg.java:66)
	at com.mmuu.travel.service.ui.LoginFrg$1.onFinish(LoginFrg.java:134)
	at android.os.CountDownTimer$1.handleMessage(CountDownTimer.java:127)
	at android.os.Handler.dispatchMessage(Handler.java:102)
	at android.os.Looper.loop(Looper.java:154)
	at android.app.ActivityThread.main(ActivityThread.java:6114)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:874)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:764)
```
网上搜了一下，是Android N在权限上做了一些修改：[参考链接](https://developer.android.google.cn/about/versions/nougat/android-7.0-changes.html) https://developer.android.google.cn/about/versions/nougat/android-7.0-changes.html
>系统权限更改
为了提高私有文件的安全性，面向 Android 7.0 或更高版本的应用私有目录被限制访问　(0700)。此设置可防止私有文件的元数据泄漏，如它们的大小或存在性。此权限更改有多重副作用：
私有文件的文件权限不应再由所有者放宽，为使用 MODE_WORLD_READABLE 和/或 MODE_WORLD_WRITEABLE 而进行的此类尝试将触发 SecurityException。
>>注：迄今为止，这种限制尚不能完全执行。应用仍可能使用原生 API 或 File API 来修改它们的私有目录权限。但是，我们强烈反对放宽私有目录的权限。

>传递软件包网域外的 file:// URI 可能给接收器留下无法访问的路径。因此，尝试传递 file:// URI 会触发 FileUriExposedException。分享私有文件内容的推荐方法是使用 FileProvider。
DownloadManager 不再按文件名分享私人存储的文件。旧版应用在访问 COLUMN_LOCAL_FILENAME 时可能出现无法访问的路径。面向 Android 7.0 或更高版本的应用在尝试访问 COLUMN_LOCAL_FILENAME 时会触发 SecurityException。通过使用 DownloadManager.Request.setDestinationInExternalFilesDir() 或 DownloadManager.Request.setDestinationInExternalPublicDir() 将下载位置设置为公共位置的旧版应用仍可以访问 COLUMN_LOCAL_FILENAME 中的路径，但是我们强烈反对使用这种方法。对于由 DownloadManager 公开的文件，首选的访问方式是使用ContentResolver.openFileDescriptor()。

解决方案：
1. FileProvider
1.1 在mainfest中加入FileProvider注册
``` xml
<application>
     <provider
         android:authorities="你的应用名.fileprovider"
         android:name="android.support.v4.content.FileProvider"
         android:grantUriPermissions="true"
         android:exported="false">
         <meta-data
           android:name="android.support.FILE_PROVIDER_PATHS"
           android:resource="@xml/filepaths"/>
    </provider>

</application>
```
1.2 在`res`文件夹下新建`xml`文件夹，在`xml`文件夹中新建`filepaths`文件，这个文件名字和上面的 Android:resource后面的名字要一致
编辑该文件：
``` xml
<?xml version="1.0" encoding="utf-8"?>
<paths>
    <external-path
        name="external_storage_root"
        path="" />
</paths>
```
1.3 修改安装代码
``` java
private void installAPK(String filePath) {
        File apkFile = new File(filePath);
        Intent intent = new Intent(Intent.ACTION_VIEW);
        if (!apkFile.exists()) {
            return;
        }

        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {
            intent.setFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
            Uri contentUri = FileProvider.getUriForFile(context, BuildConfig.APPLICATION_ID + ".fileProvider", apkFile);
            intent.setDataAndType(contentUri, "application/vnd.android.package-archive");
        } else {
            intent.setDataAndType(Uri.fromFile(apkFile), "application/vnd.android.package-archive");
        }
        getActivity().getApplicationContext().startActivity(intent);
        context.finish();
    }
```
首先判断设备的Android版本，N或者N以上使用`FileProvider`进行安装，N一下还是原来的方式。注意调用startActivity要使用ApplicationContext，使用Activity.this会报错。
2. 使用DownloadManager
``` java
public class ApkDownLoad {

    public static final String DOWNLOAD_FOLDER_NAME = getLocalForderPath();
    public static final String DOWNLOAD_FILE_NAME = "XXX.apk";
    public static final String APK_DOWNLOAD_ID = "apkDownloadId";
    private Context context;
    private String url;
    private String notificationTitle;
    private String notificationDescription;

    private DownloadManager downloadManager;
    private CompleteReceiver completeReceiver;

    /**
     * @param context
     * @param url                     下载apk的url
     * @param notificationTitle       通知栏标题
     * @param notificationDescription 通知栏描述
     */
    public ApkDownLoad(Context context, String url, String notificationTitle,
                       String notificationDescription) {
        super();
        this.context = context;
        this.url = url;
        this.notificationTitle = notificationTitle;
        this.notificationDescription = notificationDescription;
        downloadManager = (DownloadManager) context
                .getSystemService(Context.DOWNLOAD_SERVICE);
        completeReceiver = new CompleteReceiver();

        /** register download success broadcast **/
        context.registerReceiver(completeReceiver, new IntentFilter(
                DownloadManager.ACTION_DOWNLOAD_COMPLETE));
    }

    public void execute() {

        // 清除已下载的内容重新下载
        long downloadId = UpdateUtils.getLong(context, APK_DOWNLOAD_ID);
        if (downloadId != -1) {
            downloadManager.remove(downloadId);
            UpdateUtils.removeSharedPreferenceByKey(context, APK_DOWNLOAD_ID);
        }
        Request request = new Request(Uri.parse(url));
        // 设置Notification中显示的文字
        request.setTitle(notificationTitle);
        request.setDescription(notificationDescription);
        // 设置可用的网络类型
        request.setAllowedNetworkTypes(Request.NETWORK_MOBILE
                | Request.NETWORK_WIFI);
        // 设置状态栏中显示Notification
        request.setNotificationVisibility(DownloadManager.Request.VISIBILITY_VISIBLE_NOTIFY_COMPLETED);
        // 不显示下载界面
        request.setVisibleInDownloadsUi(false);
        // 设置下载后文件存放的位置
        File folder = Environment
                .getExternalStoragePublicDirectory(DOWNLOAD_FOLDER_NAME);
        if (!folder.exists() || !folder.isDirectory()) {
            folder.mkdirs();
        }
        // 设置下载文件的保存路径
        request.setDestinationInExternalPublicDir(DOWNLOAD_FOLDER_NAME,
                DOWNLOAD_FILE_NAME);
        // 设置文件类型
        MimeTypeMap mimeTypeMap = MimeTypeMap.getSingleton();
        String mimeString = mimeTypeMap.getMimeTypeFromExtension(MimeTypeMap
                .getFileExtensionFromUrl(url));
        request.setMimeType(mimeString);
        // 保存返回唯一的downloadId
        UpdateUtils.putLong(context, APK_DOWNLOAD_ID,
                downloadManager.enqueue(request));
    }

    class CompleteReceiver extends BroadcastReceiver {

        @Override
        public void onReceive(Context context, Intent intent) {
            /**
             * get the id of download which have download success, if the id is
             * my id and it's status is successful, then install it
             **/
            long completeDownloadId = intent.getLongExtra(
                    DownloadManager.EXTRA_DOWNLOAD_ID, 0);
            long downloadId = UpdateUtils.getLong(context, APK_DOWNLOAD_ID);

            if (completeDownloadId == downloadId) {

                // if download successful
                if (queryDownloadStatus(downloadManager, downloadId) == DownloadManager.STATUS_SUCCESSFUL) {

                    // clear downloadId
                    UpdateUtils.removeSharedPreferenceByKey(context,
                            APK_DOWNLOAD_ID);

                    // unregisterReceiver
                    context.unregisterReceiver(completeReceiver);

                    // install apk
                    String apkFilePath = new StringBuilder(Environment
                            .getExternalStorageDirectory().getAbsolutePath())
                            .append(File.separator)
                            .append(DOWNLOAD_FOLDER_NAME)
                            .append(File.separator).append(DOWNLOAD_FILE_NAME)
                            .toString();
                    install(context, apkFilePath);
                }
            }
        }
    }

    /**
     * 查询下载状态
     */
    public static int queryDownloadStatus(DownloadManager downloadManager,
                                          long downloadId) {
        int result = -1;
        DownloadManager.Query query = new DownloadManager.Query()
                .setFilterById(downloadId);
        Cursor c = null;
        try {
            c = downloadManager.query(query);
            if (c != null && c.moveToFirst()) {
                result = c.getInt(c
                        .getColumnIndex(DownloadManager.COLUMN_STATUS));
            }
        } finally {
            if (c != null) {
                c.close();
            }
        }
        return result;
    }

    /**
     * install app
     *
     * @param context
     * @param filePath
     * @return whether apk exist
     */
    public static boolean install(Context context, String filePath) {
        Intent i = new Intent(Intent.ACTION_VIEW);
        File file = new File(filePath);
        if (file != null && file.length() > 0 && file.exists() && file.isFile()) {
            i.setDataAndType(Uri.parse("file://" + filePath),
                    "application/vnd.android.package-archive");
            i.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
            context.startActivity(i);
            return true;
        }
        return false;
    }

}
```
检测到需要升级时  `new ApkDownLoad().execute()`就可以了，其中`UpdateUtils.getLong()`是一个`SharedPreferences`封装。
----
以上两种方式在小米5Android N 上实测有效
----
以上