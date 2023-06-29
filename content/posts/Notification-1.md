---
title: Notification_之利用TaskStackBuilder返回App主页
date: 2016-07-13 23:46:51
tags: [Notification]
categories: [Mobile,Android]
---
## 概述
本文记录点击 Notification ,然后停留在 APP 内,而不是返回主页.

<!-- more -->

## 背景:
一个App会通过通知(Notification)的形式推广自己产品的内容，点击通知，想要看到推送的详情页，点击返回的时候，我们想让用户返回的是App的主页，而不是桌面。这样可以提高转化率等。以前的实现方式是通过重写了每个activity的返回键的响应，现在可以通过TaskStackBuilder 来实现。在网上查看了很多资料，真正正确的倒是没找到。

### TaskStackBuilder 简介
Utility class for constructing synthetic back stacks for cross-task navigation on Android 3.0 and newer.是一个能够构造返回栈，来实现跨task导航的一个工具类。因为可以构造任务栈，所以，我们可以轻松的实现一个activity返回的上一个任务是什么。
	
### 实现步骤

1.首先创建一个点击了notification之后跳转到的详情页的intent对象。
```java
 Intent resultIntent = new Intent(this, ResultActivity.class);
```
2.在manifest中声明详情页ResultActivity的栈中前一个activity，声明的这个activity就是详情页点击返回键所要跳转的activity。
```xml
<activity 
	android:name="cn.steve.notification.ResultActivity"
	android:parentActivityName="cn.steve.notification.NotificationHandlerActivity">
	<meta-data
            android:name="android.support.PARENT_ACTIVITY"
            android:value="cn.steve.notification.NotificationHandlerActivity"/>
</activity>
```
3. 创建一个PendingIntent对象，这个是创建notification的必备。

```java
 PendingIntent pendingIntent = TaskStackBuilder.create(this)
            .addNextIntentWithParentStack(resultIntent)
            .getPendingIntent(0, PendingIntent.FLAG_UPDATE_CURRENT);
```

4. 剩下的就是正常的启动一个notification了。
```java
        // new notification
        NotificationCompat.Builder mBuilder = new NotificationCompat.Builder(this);
        mBuilder.setSmallIcon(android.R.drawable.ic_dialog_email);
        mBuilder.setContentTitle("My Notification!");
        mBuilder.setContentText("Hello World!");
        mBuilder.setContentIntent(pendingIntent);

        NotificationManager mNotificationManager = (NotificationManager) getSystemService(Context.NOTIFICATION_SERVICE);
        mNotificationManager.notify(0, mBuilder.build());
```
### 总结
实现的要点在于notification的设置。
