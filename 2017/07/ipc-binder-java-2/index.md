# IPC_Binder_java_2

# 概述
本文作为第一篇的补充,补充一下第一篇遗漏的内容,主要谈一下,缺少的概念,技术背景等内容.

<!-- more -->

# Why为什么需要Binder

Binder 是 Android 系统进程间通信（IPC）方式之一。Android 是基于 Linux 内核的,Linux 已经有很多 IPC 方式,为何还需要一个新的 IPC 方式.
Linux已经拥有
- 管道
- system V IPC
- Socket 等IPC手段。

却还要倚赖Binder来实现进程间通信。
- Binder具有无可比拟的优势。
- 或者可以说，Android系统对进程间有什么特殊的需求是传统其他 IPC 无法完成或者无法很好完成。

基于Client-Server的通信方式广泛应用于从互联网和数据库访问到嵌入式手持设备内部通信等各个领域。

智能手机平台特别是Android系统中，为了向应用开发者提供丰富多样的功能，这种通信方式更是无处不在，诸如媒体播放，视音频频捕获，到各种让手机更智能的传感器（加速度，方位，温度，光亮度等）都由不同的Server负责管理，应用程序只需做为Client与这些Server建立连接便可以使用这些服务，花很少的时间和精力就能开发出令人眩目的功能。

Client-Server 方式的广泛采用对进程间通信（IPC）机制是一个挑战。

只有socket支持Client-Server的通信方式。当然也可以在这些底层机制上架设一套协议来实现Client-Server通信，但这样增加了系统的复杂性，在手机这种条件复杂，资源稀缺的环境下可靠性也难以保证.

## 传输性能角度：

- socket作为一款通用接口，其传输效率低，开销大，主要用在跨网络的进程间通信和本机上进程间的低速通信。

- 消息队列和管道采用存储-转发方式，即数据先从发送方缓存区拷贝到内核开辟的缓存区中，然后再从内核缓存区拷贝到接收方缓存区，至少有两次拷贝过程。

- 共享内存虽然无需拷贝，但控制复杂，难以使用。



## 安全性角度：
Android作为一个开放式，拥有众多开发者的的平台，应用程序的来源广泛，确保智能终端的安全是非常重要的。终端用户不希望从网上下载的程序在不知情的情况下偷窥隐私数据，连接无线网络，长期操作底层设备导致电池很快耗尽等等。
1.  传统IPC没有任何安全措施，完全依赖上层协议来确保。
2.  传统IPC的接收方无法获得对方进程可靠的UID/PID（用户ID/进程ID），从而无法鉴别对方身份。Android为每个安装好的应用程序分配了自己的UID，故进程的UID是鉴别进程身份的重要标志。使用传统IPC只能由用户在数据包里填入UID/PID，但这样不可靠，容易被恶意程序利用。可靠的身份标记只有由IPC机制本身在内核中添加才能确保安全性。
3.  传统IPC访问接入点是开放的，无法建立私有通道。比如命名管道的名称，system V的键值，socket的ip地址或文件名都是开放的，只要知道这些接入点的程序都可以和对端建立连接，不管怎样都无法阻止恶意程序通过猜测接收方地址获得连接。



## 效率角度
![IPC 内存拷贝次数对比](https://github.com/tinggengyan/tinggengyan.github.io/blob/source/imgur/IPC%E6%96%B9%E5%BC%8F%E6%95%B0%E6%8D%AE%E6%8B%B7%E8%B4%9D%E6%AC%A1%E6%95%B0.png?raw=true)
从对比图可以看出,Binder 在效率上是有优势的.

# What Binder 是什么

因为 Binder 也是 CS 架构的一种,而 CS 架构最典型的就是 TCP/IP 请求了.下面做个对比,顺带类比以下 Binder 中的几个关键的概念.
![TCP/IP访问过程](https://github.com/tinggengyan/tinggengyan.github.io/blob/source/imgur/TCP%252FIP%E8%AE%BF%E9%97%AE%E8%BF%87%E7%A8%8B.png?raw=true)

![Binder类比TCP请求](https://github.com/tinggengyan/tinggengyan.github.io/blob/source/imgur/Binder%E7%B1%BB%E6%AF%94TCP%E8%BF%87%E7%A8%8B.png?raw=true)


# 背景

在开发中，经常需要通过 getSystemService 的方式获取一个系统服务,那么这些系统服务的 Binder 引用是如何传递给客户端的呢？要知道，系统服务并不是通过 startService() 启动的。

# ServiceManager 管理的服务

ServiceManager 是一个独立进程，其作用如名称所示，管理各种系统服务.

ServiceManager 本身也是一个 Service ，Framework 提供了一个系统函数，可以获取该 Service 对应的 Binder 引用，那就是 BinderInternal.getContextObject().该静态函数返回 ServiceManager 后，就可以通过 ServiceManager 提供的方法获取其他系统 Service 的 Binder 引用。这种涉及模式在日常中是可见的，
ServiceManager 就像一个公司的总机，这个号码是公开的，系统中任何进程都可以使用 BinderInternal.getContextObject() 获取该总机的 Binder 对象，而当用户想联系公司中的其他任何人(服务),则要警告总机再获得分机号。这种设计的好处是系统中仅暴露了一个全局 Binder 引用 (ServiceManager),而其他系统服务则可以隐藏起来，从而有助于系统服务的扩展，以及调用系统服务的安全检查。其他系统服务在启动时，首先把自己的 Binder 对象传递给 ServiceManager,即所谓的注册(addService).


举个具体的例子:

- ContextImpl.java
```java
 @Override
    public Object getSystemService(String name) {
        if (WINDOW_SERVICE.equals(name)) {
            return WindowManagerImpl.getDefault();
        } else if (LAYOUT_INFLATER_SERVICE.equals(name)) {
            synchronized (mSync) {
                LayoutInflater inflater = mLayoutInflater;
                if (inflater != null) {
                    return inflater;
                }
                mLayoutInflater = inflater =
                    PolicyManager.makeNewLayoutInflater(getOuterContext());
                return inflater;
            }
        } else if (ACTIVITY_SERVICE.equals(name)) {
            return getActivityManager();
        } else if (INPUT_METHOD_SERVICE.equals(name)) {
            return InputMethodManager.getInstance(this);
        } else if (ALARM_SERVICE.equals(name)) {
            return getAlarmManager();
        } else if (ACCOUNT_SERVICE.equals(name)) {
            return getAccountManager();
        } else if (POWER_SERVICE.equals(name)) {
            return getPowerManager();
        } else if (CONNECTIVITY_SERVICE.equals(name)) {
            return getConnectivityManager();
        } else if (THROTTLE_SERVICE.equals(name)) {
            return getThrottleManager();
        } else if (WIFI_SERVICE.equals(name)) {
            return getWifiManager();
        } else if (NOTIFICATION_SERVICE.equals(name)) {
            return getNotificationManager();
        } else if (KEYGUARD_SERVICE.equals(name)) {
            return new KeyguardManager();
        } else if (ACCESSIBILITY_SERVICE.equals(name)) {
            return AccessibilityManager.getInstance(this);
        } else if (LOCATION_SERVICE.equals(name)) {
            return getLocationManager();
        } else if (SEARCH_SERVICE.equals(name)) {
            return getSearchManager();
        } else if (SENSOR_SERVICE.equals(name)) {
            return getSensorManager();
        } else if (STORAGE_SERVICE.equals(name)) {
            return getStorageManager();
        } else if (VIBRATOR_SERVICE.equals(name)) {
            return getVibrator();
        } else if (STATUS_BAR_SERVICE.equals(name)) {
            synchronized (mSync) {
                if (mStatusBarManager == null) {
                    mStatusBarManager = new StatusBarManager(getOuterContext());
                }
                return mStatusBarManager;
            }
        } else if (AUDIO_SERVICE.equals(name)) {
            return getAudioManager();
        } else if (TELEPHONY_SERVICE.equals(name)) {
            return getTelephonyManager();
        } else if (CLIPBOARD_SERVICE.equals(name)) {
            return getClipboardManager();
        } else if (WALLPAPER_SERVICE.equals(name)) {
            return getWallpaperManager();
        } else if (DROPBOX_SERVICE.equals(name)) {
            return getDropBoxManager();
        } else if (DEVICE_POLICY_SERVICE.equals(name)) {
            return getDevicePolicyManager();
        } else if (UI_MODE_SERVICE.equals(name)) {
            return getUiModeManager();
        }

        return null;
    }
```

- InputMethodManager.java

```java
   /**
     * Retrieve the global InputMethodManager instance, creating it if it
     * doesn't already exist.
     * @hide
     */
    static public InputMethodManager getInstance(Context context) {
        return getInstance(context.getMainLooper());
    }
    
    /**
     * Internally, the input method manager can't be context-dependent, so
     * we have this here for the places that need it.
     * @hide
     */
    static public InputMethodManager getInstance(Looper mainLooper) {
        synchronized (mInstanceSync) {
            if (mInstance != null) {
                return mInstance;
            }
            IBinder b = ServiceManager.getService(Context.INPUT_METHOD_SERVICE);
            IInputMethodManager service = IInputMethodManager.Stub.asInterface(b);
            mInstance = new InputMethodManager(service, mainLooper);
        }
        return mInstance;
    }
```

- ServiceManager.java

```java
    /**
     * Returns a reference to a service with the given name.
     * 
     * @param name the name of the service to get
     * @return a reference to the service, or <code>null</code> if the service doesn't exist
     */
    public static IBinder getService(String name) {
        try {
            IBinder service = sCache.get(name);
            if (service != null) {
                return service;
            } else {
                return getIServiceManager().getService(name);
            }
        } catch (RemoteException e) {
            Log.e(TAG, "error in getService", e);
        }
        return null;
    }

```

通过 ServiceManager 获取 InputMethod Service 对应的 Binder 对象 b,然后再将该 Binder 对象作为 IInputMethodManager.Stub.asInterface 的参数，返回一个 IInputMethodManager 的统一接口。

ServiceManager 的 getService 方法，首先从 sCache 缓存中查看是否有对应的 Binder 对象，有则返回，没有则调用 getIServiceManager().getService().

getIServiceManager()用于返回系统中唯一的 ServiceManager 对应的 Binder。

对应的代码:

```java
private static IServiceManager getIServiceManager() {
        if (sServiceManager != null) {
            return sServiceManager;
        }
        // Find the service manager
        sServiceManager = ServiceManagerNative.asInterface(BinderInternal.getContextObject());
        return sServiceManager;
    }
```

BinderInternal.getContextObject() 方法就是用于获取 ServiceManager 对应的全局 Binder  对象。

至于关于 注册 服务，等到分析 Framework 启动过程的时候描述。


## 理解 Manager

ServiceManager 所管理的所有 Service 都是以相应的 Manager 返回给客户端，所以简述下 Framework 层对于 Manager 的定义。在Android 中，Manager 的含义类似于现实生活中的经纪人，Manager 所 manage 的对象是服务本身，因为每个具体的服务一般都会提供多个 API 接口，而 manager 所 manage 的正式这些 API。客户端一般不能直接通过  Binder  引用去访问具体的服务，而是要警告一个 Manager，相应的 Manager 类对客户端是可见的，而远程的服务端对客户端来说则是隐藏的。

而这些 Manager 的类内部都会有一个远程服务 Binder 的变量，而且在一般情况下，这些 Manager 的构造函数参数中都会包含这个 Binder  对象。简单讲，即先通过 ServiceManager 获取远程服务的 Binder 引用，然后使用这个 Binder 引用构造一个客户端本地可以访问的经纪人，然后客户端就可以通过该经纪人访问远程的服务了。


这种设计的作用是屏蔽了直接访问远程服务，从而给应用程序提供灵活的，可控的 API 接口，比如 AMS ，吸引不希望用户直接访问 AMS，而是经过 ActivityManager 类去访问，而 ActivityManager 内部提供了一些更具可操作性的数据结构化，不如 RecentTaskInfo 数据类封装了最近访问过的 Task 列表；MemoryInfo  数据类封装了和内存相关的信息。



# 感激,非常感激，万分的感激！
* [柯元旦](https://book.douban.com/subject/6811238/)
