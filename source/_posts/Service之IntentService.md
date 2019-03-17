title: Service之IntentService
date: 2016-07-02 21:35:56
tags: [Service,IntentService]
categories: [Mobile,Android]
---
# 概述
本文记录 IntentService 的使用方式,以及可能产生的问题.

<!-- more -->

# IntentService

关于IntentService本身的使用很简单，官方的解释也说的很清楚。IntentService是Service类的子类，用来处理异步请求。和正常启动一个Service一样，可以通过startService(Intent)方法启动一个IntentService，同时通过Intent传递数据。IntentService在onCreate()函数中通过HandlerThread开启一个线程来处理Intent请求对象，这样就可以在非主线程执行任务。这里处理消息的时候，也是通过为新创建的线程新建了Handler和Looper对象从消息队列中取出消息进行执行。处理每个Intent所对应的事务都需要调用 onHandleIntent 这个抽象方法。所以，将对不同任务的不同操作通过实现 onHandleIntent 方法就可完成。
执行完这个任务(Intent)就会自动停止 Service 。这里有一点需要注意，如果这个任务的执行本身就是异步的，所以，假如添加的任务也是异步的，很难保证能正常执行结束。

```java
private final class ServiceHandler extends Handler {
        public ServiceHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            onHandleIntent((Intent)msg.obj);
            stopSelf(msg.arg1);
        }
    }
```

从代码可以看出，每执行一次onHandleIntent 都会调用执行 stopSelf 来停止当前的service。也就是假如子类实现的 onHandleIntent 中执行的是异步任务，异步任务可能得不到预期的结果。

举个例子:
```java
public static void startActionBaz(Context context, String param1, String param2) {
    Intent intent = new Intent(context, MyIntentService.class);
    intent.setAction(ACTION_BAZ);
    intent.putExtra(EXTRA_PARAM1, param1);
    intent.putExtra(EXTRA_PARAM2, param2);
    context.startService(intent);
  }

  @Override
  public void onDestroy() {
    super.onDestroy();
    Log.d(TAG, "onDestroy() dead");
  }

  @Override
  protected void onHandleIntent(Intent intent) {
    if (intent != null) {
      final String action = intent.getAction();
      if (ACTION_BAZ.equals(action)) {
        final String param1 = intent.getStringExtra(EXTRA_PARAM1);
        final String param2 = intent.getStringExtra(EXTRA_PARAM2);
        handleActionBaz(param1, param2);
      }
    }
  }

  private void handleActionBaz(String param1, String param2) {
    Log.d(TAG, "handleActionBaz() called with: " + "param1 = [" + param1 + "], param2 = [" + param2 + "]");
    new Thread(new Runnable() {
      @Override
      public void run() {
        try {
          //这里的代码是无法执行的。
          Thread.sleep(3000);
          Log.d(TAG, "handleActionBaz() called at Thread");
        } catch (InterruptedException e) {
          e.printStackTrace();
        }
      }
    });
  }

```

以上代码中的 Runnable 中的代码是不会执行的。

看下启动两次 IntentService 的log输出。

```java
07-02 10:04:03.799 4785-4785/cn.steve.study D/MyIntentService: MyIntentService() called with: cn.steve.service.MyIntentService@f26cde3
07-02 10:04:03.811 4785-5288/cn.steve.study D/MyIntentService: handleActionBaz() called with: param1 = [], param2 = []
07-02 10:04:03.840 4785-4785/cn.steve.study D/MyIntentService: onDestroy() dead
07-02 10:04:04.079 4785-4785/cn.steve.study D/MyIntentService: MyIntentService() called with: cn.steve.service.MyIntentService@47d6d5e
07-02 10:04:04.089 4785-5293/cn.steve.study D/MyIntentService: handleActionBaz() called with: param1 = [], param2 = []
07-02 10:04:04.114 4785-4785/cn.steve.study D/MyIntentService: onDestroy() dead
```

发现每次都是新建 IntentService 对象，执行完后就会销往。

# 总结
系统为我们提供这个 IntentService 能在完成任务后自动停止销往，不需要我们手动停止，但是这个特性只能对一些简单的同步任务而已，对于异步任务，还是需要我们手动去 stop 。
