# Android开发--使用前台通知实现显示下载进度条和一些坑

实现思路：<br>
1、开启一个后台服务；<br>
2、在后台服务中实现下载的逻辑；<br>
3、在下载时，打开前台通知显示下载的进度；<br>
4、下载完成时，点击通知完成其他操作。<br>

```
//
...
省略后台服务的实现
//

//初始化通知对象
private void initNotification() {
    //获取Application
    AppContext context = AppContext.getInstance();
    //获取通知管理
    notificationManager = (NotificationManager) context.getSystemService(Context.NOTIFICATION_SERVICE);

    String InfoChannel = "0";
    //兼容API26 ，创建NotificationCompat.Builder
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
        NotificationChannel mChannel = new NotificationChannel(InfoChannel, InfoChannel, NotificationManager.IMPORTANCE_HIGH);
        //去掉通知声音
        mChannel.setSound(null, null);
        mChannel.setImportance(NotificationManager.IMPORTANCE_LOW);
        notificationManager.createNotificationChannel(mChannel);
        builder = new NotificationCompat.Builder(context, InfoChannel);
    } else {
        builder = new NotificationCompat.Builder(context);
    }
    builder.setContentTitle("下载更新") //设置通知标题
            .setSmallIcon(R.drawable.ic_launcher_round)
            .setPriority(NotificationCompat.PRIORITY_MAX) //设置通知的优先级
            .setAutoCancel(false)//设置通知被点击是否自动关闭
            .setContentText("0%")
            .setProgress(100, 0, false);
    notification = builder.build();//构建通知对象
}

//notificationManager.notify
//调用这个方法时有些坑，短时间内调用上限为50次，超出会接收不到通知

//通知更新
public void onProgress(int progress) {
    notification = builder.setContentText(progress +"%")
            .setProgress(100, progress, false)
            .setAutoCancel(false).build();
    notificationManager.notify(1, notification);
}

//下载成功时
public void onSuccess(){
  notification = builder.setContentText("下载完成")
            .setProgress(100, 100, false)
            .setAutoCancel(true).build();
            notificationManager.notify(1, notification);
}
```

