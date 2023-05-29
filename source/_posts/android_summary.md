---
title: 一张思维导图看 Android【持续迭代】
tags:
  - Android
categories:
  - Android
date: 2022-04-03 16:19:04
---
# 概述
总结自己的 Android 知识，按照编码 -> 运行，画了一张图，xmind 导出的图比较大，后续持续更新，迭代这部分的内容。

![Android](/img/android/Android_summary.png)


# Handler 与 Binder

handler 处理的是 App 进程内的通信；
binder 处理的是，App 进程间、application 和 framework 之间的通信；


# 任务启动管理 -  启动框架
- 抽象任务 task；
	- 优先级 ；
	- countdownlatch 数值为依赖 task 的数量；
	- 运行的 executor ；
	- 被依赖的 task 列表；
	- toWait 方法；
	- notify 方法，countdownlatch 减一；
- 构造 task 的有向无环图；
- TaskManager：
	- 管理所有的 task，及其拓扑关系；
	- 管理需要执行的 task；
	- countdownlatch 值为 

# Activity 跳转的生命周期
- ActivityA跳转到ActivityB：
```java
Activity A：onPause
Activity B：onCreate
Activity B：onStart
Activity B：onResume
Activity A：onStop
```

- ActivityB返回ActivityA：
```java
Activity B：onPause
Activity A：onRestart
Activity A：onStart
Activity A：onResume
Activity B：onStop
Activity B：onDestroy
```

## 旋转屏幕

- 不改配置，默认配置
```java
onPause-->
onStop-->
onDestroy-->
onCreate-->
onStart-->
onRestoreInstanceState-->
onResume-->

```


- 修改配置
```java
onConfigChanged-->
```


# Activity 的启动模式
[参考链接1](https://www.jianshu.com/p/b3a95747ee91)

**仅仅适用于Activity启动Activity，并且采用的都是默认Intent，没有额外添加任何Flag**
-   standard：标准启动模式（默认启动模式），每次都会启动一个新的activity实例。
-   singleTop：单独使用使用这种模式时，如果**Activity实例位于当前任务栈顶**，就重用栈顶实例，而不新建，并回调该实例onNewIntent()方法，否则走新建流程。
-   singleTask：这种模式启动的Activity**只会存在相应的Activity的taskAffinit任务栈中**，同一时刻系统中只会存在一个实例，已存在的实例被再次启动时，会重新唤起该实例，并清理当前Task任务栈该实例之上的所有Activity，同时回调onNewIntent()方法。
-   singleInstance：这种模式启动的Activity独自占用一个Task任务栈，同一时刻系统中只会存在一个实例，已存在的实例被再次启动时，只会唤起原实例，并回调onNewIntent()方法。

## Intent.FLAG_ACTIVITY_NEW_TASK分析
当我们用ApplicationContext去启动standard模式的Activity会报错，这是因为standard模式的Activity默认会进入启动它的Activity所属的任务栈中，但是由于非Activity类型的Context(如ApplicationContext)并没有所谓的任务栈，这就是问题所在。解决这个问题的方法是为待启动Activity指定FLAG_ACTIVITY_NEW_TASK标记位，这样启动的时候就会为它创建一个新的任务栈，如果设置待启动的Activity的taskAffinity时，这个时候待启动Activity实际上是以singleTask模式启动的。
## FLAG_ACTIVITY_SINGLE_TOP

为Activity指定singleTop启动模式，效果和在xml中指定该模式相同。
## FLAG_ACTIVITY_CLEAR_TOP

具有此标记的Activity，当它启动时，在同一个任务栈中所有位于它上面的Activity都要出栈。
## FLAG_ACTIVITY_CLEAR_TASK

FLAG_ACTIVITY_CLEAR_TASK只能和FLAG_ACTIVITY_NEW_TASK配合使用哦
## FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS

具有这个标记的Activity不会出现在历史Activity的列表中，在某些情况下我们不希望用户通过历史列表回到我们的Activity的时候这个标记比较有用。它等同于在xml中指定Activity的属性`android``:excludeFromRecents="true"`


# Activity 的启动流程
- [参考链接1](https://linguanghua.github.io/Programming-Tech-Books/advanced/Activity%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B.html)

![启动流程图](https://raw.githubusercontent.com/sucese/android-open-source-project-analysis/master/art/app/component/activity_start_flow.png)

点击应用图标后会去启动应用的Launcher Activity，如果Launcer Activity所在的进程没有创建，还会创建新进程，整体的流程就是一个Activity的启动流程。

-   Instrumentation: 监控应用与系统相关的交互行为。
-   AMS：组件管理调度中心，什么都不干，但是什么都管。
-   ActivityStarter：Activity启动的控制器，处理Intent与Flag对Activity启动的影响，具体说来有：
	- 1 寻找符合启动条件的Activity，如果有多个，让用户选择；
	- 2 校验启动参数的合法性；
	- 3 返回int参数，代表Activity是否启动成功。
-   ActivityStackSupervisior：这个类的作用你从它的名字就可以看出来，它用来管理任务栈。
-   ActivityStack：用来管理任务栈里的Activity。
-   ActivityThread：最终干活的人，Activity、Service、BroadcastReceiver的启动、切换、调度等各种操作都在这个类里完成。

注：这里单独提一下ActivityStackSupervisior，这是高版本才有的类，它用来管理多个ActivityStack，早期的版本只有一个ActivityStack对应着手机屏幕，后来高版本支持多屏以后，就有了多个ActivityStack，于是就引入了ActivityStackSupervisior用来管理多个ActivityStack。


有了以上的理解，整个流程可以概括如下：
- 1、点击桌面应用图标，Launcher 进程将启动 Activity（MainActivity）的请求以 Binder 的方式发送给了 AMS。
- 2、AMS 接收到启动请求后，交付 ActivityStarter 处理 Intent 和 Flag 等信息，然后再交给 ActivityStackSupervisior/ActivityStack 处理 Activity 进栈相关流程。同时以 Socket 方式请求 Zygote 进程 fork 新进程。
- 3、Zygote 接收到新进程创建请求后 fork 出新进程。
- 4、在新进程里创建 ActivityThread 对象，新创建的进程就是应用的主线程，在主线程里开启 Looper 消息循环，开始处理创建 Activity。
- 5、ActivityThread 利用 ClassLoader 去加载 Activity、创建 Activity 实例，并回调 Activity 的 onCreate() 方法，这样便完成了Activity 的启动。

![详细的交互图](https://camo.githubusercontent.com/11c57b4fbb9ff82426792bac32fe8de173949dccc19a4f0c0278ef62bf67203f/687474703a2f2f696d672e6d702e6974632e636e2f75706c6f61642f32303137303332392f63613935363763653362663034633461626462346431323463656266656537365f74682e6a706567)




- 从开发层面来看，启动另一个 activity 通常会使用 context.startActivity()，最终去到Instrumentation 的 execStartActivitiesAsUser 方法中。
```java
int result = ActivityManagerNative.getDefault()
    .startActivities(whoThread, who.getBasePackageName(), intents, resolvedTypes,token, options, userId);
```
- ActivityManagerNative 获取 AMS 进程代理，进而告知 AMS 将要启动另一个 activity。
```java
static public IActivityManager getDefault() {
    return gDefault.get();
}

private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
    protected IActivityManager create() {
        IBinder b = ServiceManager.getService("activity");
        if (false) {
            Log.v("ActivityManager", "default service binder = " + b);
        }
        IActivityManager am = asInterface(b);
        if (false) {
            Log.v("ActivityManager", "default service = " + am);
        }
        return am;
    }
};
```
- 启动如此，Activity 的生命周期又是如何被 AMS 进程管理的呢？在 ActivityThread 的 attach 方法中
```java
final IActivityManager mgr = ActivityManagerNative.getDefault();
try {
    mgr.attachApplication(mAppThread);
} catch (RemoteException ex) {
    throw ex.rethrowFromSystemServer();
}
```
- ActivityManagerNative 把 ApplicationThread 的代理写给了AMS进程
```java
public void attachApplication(IApplicationThread app) throws RemoteException
{
    Parcel data = Parcel.obtain();
    Parcel reply = Parcel.obtain();
    data.writeInterfaceToken(IActivityManager.descriptor);
    data.writeStrongBinder(app.asBinder());
    mRemote.transact(ATTACH_APPLICATION_TRANSACTION, data, reply, 0);
    reply.readException();
    data.recycle();
    reply.recycle();
}
```

进而会有一系列scheduleXXXActivity回调。

```java
public final void schedulePauseActivity(IBinder token, boolean finished,
        boolean userLeaving, int configChanges, boolean dontReport) {
    int seq = getLifecycleSeq();
    if (DEBUG_ORDER) Slog.d(TAG, "pauseActivity " + ActivityThread.this
            + " operation received seq: " + seq);
    sendMessage(
            finished ? H.PAUSE_ACTIVITY_FINISHING : H.PAUSE_ACTIVITY,
            token,
            (userLeaving ? USER_LEAVING : 0) | (dontReport ? DONT_REPORT : 0),
            configChanges,
            seq);
}
```


## 总结
### 第一部分 -  上半场
>当启动Activity的过程中且在AMS处理之前的操作，称之为"上半场"。

Activity -> Instrumentation

### 第二部分 - 下半场
> AMS处理完成，并开始回调App 进程的过程到Activity启动大的操作，称之为"下半场"。

ActivityStarter -> ActivityStackSupervisor -> ActivityThread -> ActivityThread.java#H -> handleLaunchActivity -> performLaunchActivity 

# Context 的区别
Context 的关联类采用了装饰模式，主要有以下的优点：

-   使用者（比如 Service ）能够方便的使用 Context。
-   如果 ContextImpl 发生变化，它的装饰类 ContextWrapper 不需要做任何修改。
-   ContextImpl 的实现不会暴露给使用者，使用者也不必关心 ContextImpl 的实现。
-   通过组合而非继承的方式，拓展 ContextImpl 的功能，在运行时选择不同的装饰类，实现不同的功能。

![Context Summary](https://static001.geekbang.org/infoq/fc/fce9147eea433965e4becfb9bc2e067c.webp?x-oss-process=image%2Fresize%2Cp_80%2Fformat%2Cpng)

Context的数量等于Activity的个数 + Service的个数 +1，这个1为Application。

这还涉及 application  启动 activity 的时候，需要加 Intent.FLAG_ACTIVITY_NEW_TASK flag；


application 的 context 是在启动进程的时候创建的。



# Android 的内存管理/回收策略
> 对Android系统有较好理解，知道AMS与Kennel层LMK相结合的方式(Adj)去管理和查杀或回收进程；
以及LMK的内存统计策略，当系统内存不足时，AMS会遍历App进程并通知进程进行内存释放(onTrimMemoryLevel)，以及触发各个进程进行GC等。
需要有一些纵向层次的理解，涉及LMK，AMS，APP多层的协同机制，其中每个层面又都有一套自己的策略。

## AMS

AMS 在进程管理这块儿做两件事
- **进程LRU列表动态更新**：动态调整进程在mLruProcesses列表的位置
- 进程优先级动态调整：实际是调整进程oom_adj的值。

## LMK机制
**在Android中，即使当用户退出应用程序后，应用进程也还会存在内存中，方便下次可以快速进入应用而不需要重新创建进程**。 这样带来的直接影响就是**由于进程数量越来越多，系统内存会越来越少，这个时候就需要杀死一部分进程来缓解内存压力。** **至于哪些进程会被杀死，这个时候就需要用到Low Memory Killer机制来进行判定。**

**Android的Low Memory Killer基于Linux的OOM机制**， 在Linux中，内存是以页面为单位分配的，当申请页面分配时如果内存不足会通过以下流程选择 bad 进程来杀掉从而释放内存

**LMK驱动层在用户空间指定了一组内存临界值及与之一一对应的一组 oom_adj 值，** **当系统剩余内存位于内存临界值中的一个范围内时，如果一个进程的 oom_adj 值大于或等于这个临界值对应的oom_adj值就会被杀掉。**

# Binder 
- [可参考链接](http://gityuan.com/2015/11/28/binder-summary/)

1.  从IPC角度来说：Binder是Android中的一种跨进程通信方式，该通信方式在linux中没有，是Android独有；
2.  从 Android Driver 层：Binder 还可以理解为一种虚拟的物理设备，它的设备驱动是 /dev/binder；
3.  从 Android Native 层：Binder 是创建 ServiceManager 以及 BpBinder/BBinder 模型，搭建与binder 驱动的桥梁；
4.  从 Android Framework 层：Binder 是各种 Manager（ActivityManager、WindowManager等）和相应 xxxManagerService 的桥梁；
5.  从 Android APP 层：Binder 是客户端和服务端进行通信的媒介，当 bindService 的时候，服务端会返回一个包含了服务端业务调用的 Binder对象，通过这个Binder对象，客户端就可以获取服务端提供的服务或者数据，这里的服务包括普通服务和基于AIDL的服务。

Binder 是基于 C/S 架构设计的。由 Client、Server、ServiceManager 和 Binder 驱动组成。Client、Server负责自各的业务、ServiceManager 运行在**用户空间**负责代理业务，Binder 驱动运行在内核空间真正对接业务细节。其中 ServiceManager 和 Binder 驱动由系统提供，而 Client、Server 由应用程序来实现。

![binder 的framework架构图](http://gityuan.com/images/binder/java_binder/java_binder.jpg)

## 一次通信的过程

1.  Server 进程向 Binder 驱动发送 Binder 实体请求注册服务， Binder 驱动将请求转发到 ServiceManager 注册后把名字和 Binder 实体等信息填入查找表中。
2.  Client 进程在 Binder 驱动的帮助下通过名字从 ServiceManager 中获取到 Server Binder 实体的引用，至此，Client 与 Server 总算是勾搭上了。
3.  与此同时，Binder 驱动开始创建数据缓冲区并通过 ServiceManager 提供的 Server 进程信息地址使用 mmap 函数将 Server 进程用户空间地址映射到创建好的缓冲区上，为跨进程通信做好了准备。
4.  Client 进程调用 copy_from_user 函数将数据拷贝到数据缓冲区后 Binder 驱动再通知 Server 进程进行解包。
5.  Server 进程收到通知后进行解包和方法调用，并将结果写到自己的有映射的内存中。
6.  由于内核缓冲区与 Server进程用户空间地址存在映射关系的，这就相当于目标方法的结果直接传递到了内核数据缓冲区，Binder 驱动通知 Client 进程调用 copy_to_user 函数将缓冲区的目标结果拷贝到自己的用户空间中。

## 为什么要使用 Binder？

性能：移动设备中如果广泛的使用跨进程通信机制肯定会对通信机制提出严格的要求，而 Binder 相比较传统的进程通信方式更加的高效。
安全：由于传统进程通信方式没有对通信的双方和身方做出严格的验证，只有上层协议才会去架构，如 socket 连接的 IP 地址可以人为的伪造。而 Binder 身份校验也是 android 权限模式的基础。


## Binder 的线程管理
每个 Binder 的 Server 进程会创建很多线程来处理 Binder 请求，可以简单的理解为创建了一个 Binder 的线程池吧（虽然实际上并不完全是这样简单的线程管理方式），而真正管理这些线程并不是由这个 Server 端来管理的，而是由 Binder 驱动进行管理的。



## Binder 有什么优势？
性能方面
- 共享内存 0 次数据拷贝
- Binder 1 次数据拷贝
- Socket/管道/消息队列 2 次数据拷贝

稳定性方面
- Binder：基于 C/S 架构，客户端（Client）有什么需求就丢给服务端（Server）去完成，架构清晰、职责明确又相互独立，自然稳定性更好
- 共享内存：虽然无需拷贝，但是控制复杂，难以使用
- 从稳定性的角度讲，Binder 机制是优于内存共享的。

安全性方面
- 传统的 IPC 没有任何安全措施，安全依赖上层协议来确保。
- 传统的 IPC 方法无法获得对方可靠的进程用户 ID/进程 UI（UID/PID），从而无法鉴别对方身份。 binder 的 uid 是由 driver 保证；
- 传统的 IPC 只能由用户在数据包中填入 UID/PID，容易被恶意程序利用。
- 传统的 IPC 访问接入点是开放的，无法阻止恶意程序通过猜测接收方地址获得连接。
- Binder 既支持实名 Binder，又支持匿名 Binder，安全性高。


## Binder 是如何做到一次拷贝的？
主要是因为 Linux 是使用的虚拟内存寻址方式，它有如下特性：

-   用户空间的虚拟内存地址是映射到物理内存中的；
-   对虚拟内存的读写实际上是对物理内存的读写，这个过程就是内存映射；
-   这个内存映射过程是通过系统调用 mmap() 来实现的；

**Binder 借助了内存映射的方法，在内核空间和接收方用户空间的数据缓存区之间做了一层内存映射，就相当于直接拷贝到了接收方用户空间的数据缓存区，从而减少了一次数据拷贝**


## Binder 机制是如何跨进程的？
- Binder 驱动
	- 在内核空间创建一块接收缓存区；
	- 实现地址映射：将内核缓存区、接收进程用户空间映射到同一接收缓存区
- 发送进程通过系统调用（copy_from_user）将数据发送到内核缓存区。
由于内核缓存区和接收进程用户空间存在映射关系，故相当于也发送了接收进程的用户空间，实现了跨进程通信。


## 为什么 Intent 不能传递大数据？
Intent 携带信息的大小其实是受 Binder 限制。数据以 Parcel 对象的形式存放在 Binder 传递缓存中。如果数据或返回值比传递 buffer 大，则此次传递调用失败并抛出 TransactionTooLargeException 异常。

Binder 传递缓存有一个限定大小，通常是 1Mb。但同一个进程中所有的传输共享缓存空间。多个地方在进行传输时，即时它们各自传输的数据不超出大小限制，TransactionTooLargeException 异常也可能会被抛出。在使用 Intent 传递数据时，1Mb 并不是安全上限。因为 Binder 中可能正在处理其它的传输工作。不同的机型和系统版本，这个上限值也可能会不同。



## Binder IPC 实现原理

Binder IPC 正是基于内存映射（mmap）来实现的，但是 mmap() 通常是用在有物理介质的文件系统上的。

比如进程中的用户区域是不能直接和物理设备打交道的，如果想要把磁盘上的数据读取到进程的用户区域，需要两次拷贝（磁盘–>内核空间–>用户空间）；通常在这种场景下 mmap() 就能发挥作用，通过在物理介质和用户空间之间建立映射，减少数据的拷贝次数，用内存读写取代 I/O 读写，提高文件读取效率。

而 Binder 并不存在物理介质，因此 Binder 驱动使用 mmap() 并不是为了在物理介质和用户空间之间建立映射，而是用来在内核空间创建数据接收的缓存空间。

### 一次完整的 Binder IPC 通信过程通常是这样：

- 首先 Binder 驱动在内核空间创建一个数据接收缓存区；
- 接着在内核空间开辟一块内核缓存区，建立内核缓存区和内核中数据接收缓存区之间的映射关系，以及内核中数据接收缓存区和接收进程用户空间地址的映射关系；
- 发送方进程通过系统调用 copyfromuser() 将数据 copy 到内核中的内核缓存区，由于内核缓存区和接收进程的用户空间存在内存映射，因此也就相当于把数据发送到了接收进程的用户空间，这样便完成了一次进程间的通信。



# Oneway 关键词
非阻塞式调用。
Oneway 表示客户端不关服务端的返回值，甚至也不关系执行是否正常或者异常。只管通知一下。 代码层面也可以看出来，我们仔细看 initASVE 和 initASVEOneway 这两个方法在aidl内的实现。

oneway 关键词是用在 AIDL 中的，目的是实现异步调用， IPC 调用的时候，不用等待 server 的 reply 就返回。

Client端，判断如果是 ONEWAY，在会在transaction发送到binder驱动后（收到BR_TRANSACTION_COMPLETE），马上退出等待。


## 坑点
oneway 默认处理异常不返回。发现为oneway直接吃掉了异常，仅仅打印一行日志。 也没有任何机制可以给上层通知。

# Handler机制
![handler的示意图](https://github.com/guoxiaoxing/android-open-source-project-analysis/raw/master/art/native/process/android_handler_structure.png)

主要涉及的角色如下所示：
-  Message：消息。
- MessageQueue：消息队列，负责消息的存储与管理，负责管理由 Handler 发送过来的 Message。读取会自动删除消息，单链表维护，插入和删除上有优势。在其next()方法中会无限循环，不断判断是否有消息，有就返回这条消息并移除。
- Looper：消息循环器，负责关联线程以及消息的分发，在该线程下从 MessageQueue获取 Message，分发给Handler，Looper创建的时候会创建一个 MessageQueue，调用loop()方法的时候消息循环开始，其中会不断调用messageQueue的next()方法，当有消息就处理，否则阻塞在messageQueue的next()方法中。当Looper的quit()被调用的时候会调用messageQueue的quit()，此时next()会返回null，然后loop()方法也就跟着退出。
- Handler：消息处理器，负责发送并处理消息，面向开发者，提供 API，并隐藏背后实现的细节。

整个消息的循环流程还是比较清晰的，具体说来：

- 1、Handler通过sendMessage()发送消息Message到消息队列MessageQueue。
- 2、Looper通过loop()不断提取触发条件的Message，并将Message交给对应的target handler来处理。
- 3、target handler调用自身的handleMessage()方法来处理Message。

事实上，在整个消息循环的流程中，并不只有Java层参与，很多重要的工作都是在C++层来完成的。我们来看下这些类的调用关系。

![](https://github.com/guoxiaoxing/android-open-source-project-analysis/raw/master/art/native/process/android_handler_class.png)

注：虚线表示关联关系，实线表示调用关系。

在这些类中MessageQueue是Java层与C++层维系的桥梁，MessageQueue与Looper相关功能都通过MessageQueue的Native方法来完成，而其他虚线连接的类只有关联关系，并没有直接调用的关系，它们发生关联的桥梁是MessageQueue。

##### 总结
- Handler 发送的消息由 MessageQueue 存储管理，并由 Looper 负责回调消息到 handleMessage()。
- 线程的转换由 Looper 完成，handleMessage() 所在线程由 Looper.loop() 调用者所在线程决定。

## Handler 引起的内存泄露原因以及最佳解决方案
Handler 允许我们发送延时消息，如果在延时期间用户关闭了 Activity，那么该 Activity 会泄露。 这个泄露是因为 Message 会持有 Handler，而又因为 Java 的特性，内部类会持有外部类，使得 Activity 会被 Handler 持有，这样最终就导致 Activity 泄露。

解决：将 Handler 定义成静态的内部类，在内部持有 Activity 的弱引用，并在Acitivity的onDestroy()中调用handler.removeCallbacksAndMessages(null)及时移除所有消息。


## 为什么我们能在主线程直接使用 Handler，而不需要创建 Looper ？
通常我们认为 ActivityThread 就是主线程。事实上它并不是一个线程，而是主线程操作的管理者。在 ActivityThread.main() 方法中调用了 Looper.prepareMainLooper() 方法创建了 主线程的 Looper ,并且调用了 loop() 方法，所以我们就可以直接使用 Handler 了。

## Handler 里藏着的 Callback 能干什么？
Handler.Callback 有优先处理消息的权利 ，当一条消息被 Callback 处理并拦截（返回 true），那么 Handler 的 handleMessage(msg) 方法就不会被调用了；如果 Callback 处理了消息，但是并没有拦截，那么就意味着一个消息可以同时被 Callback 以及 Handler 处理。


## 创建 Message 实例的最佳方式

为了节省开销，Android 给 Message 设计了回收机制，所以我们在使用的时候尽量复用 Message ，减少内存消耗：

    通过 Message 的静态方法 Message.obtain()；
    通过 Handler 的公有方法 handler.obtainMessage()。


## 主线程的死循环一直运行是不是特别消耗CPU资源呢？
并不是，这里就涉及到Linux pipe/epoll机制，简单说就是在主线程的MessageQueue没有消息时，便阻塞在loop的queue.next()中的nativePollOnce()方法里，此时主线程会释放CPU资源进入休眠状态，直到下个消息到达或者有事务发生，通过往pipe管道写端写入数据来唤醒主线程工作。这里采用的epoll机制，是一种IO多路复用机制，可以同时监控多个描述符，当某个描述符就绪(读或写就绪)，则立刻通知相应程序进行读或写操作，本质是同步I/O，即读写是阻塞的。所以说，主线程大多数时候都是处于休眠状态，并不会消耗大量CPU资源。


## handler postDelay这个延迟是怎么实现的？
handler.postDela y并不是先等待一定的时间再放入到 MessageQueu e中，而是直接进入MessageQueue，以 MessageQueue 的时间顺序排列和唤醒的方式结合实现的。

next 方法中，有计算下次唤醒的时间间隔；


## 如何保证在msg.postDelay情况下保证消息次序？
入队的时候，会按照时间顺序进行排序。


# AMS 
- [参考链接1](https://cloud.tencent.com/developer/article/1466430)

## 关键的类
1. ActivityManagerServices，简称AMS，服务端对象，负责系统中所有Activity的生命周期。
2. ActivityThread，App 的真正入口。当开启 App 之后，调用 main() 开始运行，开启消息循环队列，这就是传说的 UI 线程或者叫主线程。与 ActivityManagerService 一起完成Activity的管理工作。
3. ApplicationThread，用来实现 ActivityManagerServie 与 ActivityThread 之间的交互。在ActivityManagerSevice 需要管理相关 Application 中的 Activity 的生命周期时，通过 ApplicationThread 的代理对象与 ActivityThread 通信。
4. ApplicationThreadProxy，是 ApplicationThread 在服务器端的代理，负责和客户端的 ApplicationThread 通信。AMS 就是通过该代理与 ActivityThread 进行通信的。
5. Instrumentation，每一个应用程序只有一个 Instrumetation 对象，每个 Activity 内都有一个对该对象的引用，Instrumentation 可以理解为应用进程的管家，ActivityThread 要创建或暂停某个 Activity 时，都需要通过 Instrumentation 来进行具体的操作。
6. ActivityStack，Activity 在 AMS 的栈管理，用来记录经启动的 Activity 的先后关系，状态信息等。通过 ActivtyStack 决定是否需要启动新的进程。
7. ActivityRecord，ActivityStack 的管理对象，每个 Acivity 在 AMS 对应一个 ActivityRecord，来记录 Activity 状态以及其他的管理信息。**其实就是服务器端的 Activity 对象的映像**。
8. TaskRecord，AMS 抽象出来的一个“任务”的概念，是记录 ActivityRecord 的栈，一个“Task”包含若干个ActivityRecord。AMS 用 TaskRecord 确保 Activity 启动和退出的顺序。如果你清楚 Activity 的4种launchMode，那么对这概念应该不陌生。

## 简述ActivityManagerService的作用，什么时候初始化？
ActivityManagerService 主要负责系统中四大组件的启动、切换、调度及应用进程的管理和调度等工作，其职责与操作系统中的进程管理和调度模块类似。

ActivityManagerService进行初始化的时机很明确，就是在SystemServer进程开启的时候，就会初始化ActivityManagerService， 可以在SystemServer类中找到相关的启动代码。

## 简述ActivityThread和ApplicationThread，以及关系和区别
**ActivityThread**

ActivityThread在Android中代表Android的主线程，但是并不是一个Thread类。ActivityThread类是Android 进程的初始类，它的main函数是这个App进程的入口。当创建完新进程之后，main函数被加载，然后执行一个loop的循环使当前线程进入消息循环。

**ApplicationThread**

ApplicationThread是ActivityThread的内部类， 是一个Binder对象。在此处它是作为IApplicationThread对象的server端等待client端的请求然后进行处理，最大的client就是AMS。

首先，我们看一下Activity的启动逻辑过程：Applicationthread的ScheduleActivity通过一个叫H的Handler发送了一个启动Activity信息。handleLaunchActivity接收了这个消息，然后做处理，处理的逻辑是让PreformLaunchActivity处理，并最终执行Activity的启动。


## Instrumentation是什么，和ActivityThread是什么关系
Instrumentation 是Android系统中一系列控制方法的集合(hook),这些方法可以在正常的生命周期之外控制Android控件的运行，也可以控制Andoroid如何加载应用程序。

事实上，AMS与ActivityThread之间诸如Activity的创建、暂停等的交互工作都是由Instrumentation操作的。并且每个Activity都持有一个Instrumentation对象的一个引用， 整个进程中是只有一个Instrumentation。当startActivityForResult()调用之后，实际上还是调用了mInstrumentation.execStartActivity()。

它们之间的关系如下： **AMS是大BOSS资本，负责指挥和调度的，ActivityThread是企业老板，虽然说企业的事自己说了算，但是需要听从AMS的指挥，而Instrumentation则是CTO，负责项目的大事小事，但是一般不抛头露面，听老板ActivityThread的安排。**

## ActivityManagerService和zygote进程通信是如何实现的
应用启动时，Launcher进程请求 AMS，AMS 发送创建应用进程请求，Zygote进程接受请求并fork应用进程。而AMS发送创建应用进程请求调用的是 ZygoteState.connect() 方法，ZygoteState 是 ZygoteProcess 的内部类。

Zygote 处理客户端请求：Zygote 服务端接收到参数之后调用 ZygoteConnection.processOneCommand() 处理参数，并 fork 进程。

最后通过 findStaticMain() 找到 ActivityThread 类的 main() 方法并执行，子进程就这样启动了。


**怎么用**
Context.getSystemService()方法获取AMS的实例，然后调用其提供的API。

# WMS
## WMS 的作用
Android中的window机制的出现就是为了统一管理屏幕上的显示View，让程序有条不紊的执行。

- WMS 管理的是窗口，负责窗口的添加、删除、顺序的调整；
- WMS 不负责洁面的合成和绘制，合成和绘制是由 SurfaceFlinger 完成；
- View 必须被添加到窗口中，才可以被绘制；
- View 有自己的 onDraw 回调，在这个回调里进行视图的绘制操作；
- View 每个视图都是一个图层，就是一个 surface，这块的内存是由 SurfaceFlinger 申请的；
	- 这块内存是通过内存共享的方式在 SurfaceFlinger 和 App 进程之间进行共享的；
	- 具体的感知方式：SurfaceFlinger 申请了 surface 的内存后，通过 binder 的方式将内存相关信息传递给 App 进程，App 向内存中写入绘制内容，绘制完毕，通知 SurfaceFlinger 进行图层的混排，再将数据渲染到屏幕上；
	- 为什么使用共享内存的方式，而不是 binder 呢？因为图像的数据太大了，binder 和 socket 效率都太低了；

## 关键的类
1. PhoneWindow：Window我们应该很熟悉，它是一个抽象类，具体的实现类为PhoneWindow，它对View进行管理。
2. WindowManagerImpl  ：WindowManager是一个接口类，继承自接口ViewManager, 它是用来管理Window的，它的实现类为WindowManagerImpl。
3. ViewManager中定义了三个方法：ViewManager中定义了三个方法，分别用来添加、更新和删除View 
4. DecorView：窗口内的真实视图；最高层的视图；
5. ViewRootImpl：View 的最高层级，实现 view 和 WindowManager 之间的一些协议；可以触发视图的绘制；可以理解成一个桥梁；内部持有 DecorView 实例的引用；
6. WindowSession：
7. windowstate：WindowState是系统进程层面管理的window对象，windowState通过InputChannel与IMS获取联系。
8. W，IWindow.Stub 的 binder 服务对象，主要的作用是接收 WMS 的 IPC 调用；
9. 想要对Window进行添加和删除就可以使用WindowManager，具体的工作都是由WMS来处理的，WindowManager和WMS通过Binder来进行跨进程通信，WMS作为系统服务有很多API是不会暴露给WindowManager的，这一点与ActivityManager和AMS的关系有些类似。

## 创建图层的流程
1.  首先APP端新建一个Surface图层的容器壳子，
2.  APP通过Binder通信将这个Surface的壳子传递给WMS，
3.  WMS为了填充Surface去向SurfaceFlinger申请真正的图层，
4.  SurfaceFlinger收到WMS请求为APP端的Surface分配真正图层
5.  将图层相关的关键信息Handle及Producer传递给WMS
6.  WMS利用Handle和Producer创建一个SurfaceControl对象，之后再利用其创建Surface
7.  至此，WMS端的Surface创建并填充完毕，然后返回给APP端，APP端就获得了直接和SurfaceFlinger通信的能力


## View 的绘制原理
- [参考链接1](https://www.youtube.com/watch?v=1iaHxmfZGGc)
- [参考链接2](https://www.bilibili.com/video/BV19V4y1V79X/?spm_id_from=333.337.search-card.all.click&vd_source=2383886846e4aa5e7cf6fd52f9d0a367)

### 屏幕的刷新机制：
1. 在一个典型的显示系统中，一般包括 CPU、GPU、Display 三个部分， CPU负责计算帧数据，把计算好的数据交给GPU，GPU会对图形数据进行渲染，渲染好后放到buffer(图像缓冲区)里存起来，然后Display（屏幕或显示器）负责把buffer里的数据呈现到屏幕上。
2. 画面撕裂：由于上述谈到的，缓冲生产出的视图数据和绘制数据的数据源，使用的是同一个 buffer，就可能出现在屏幕刷新的时候，数据不一致，进而导致不是一个完整的一帧画面的问题；
3. 为了解决画面撕裂的问题
	1. 双缓冲：一个绘制数据的缓冲；一个显示数据的缓冲；二者互不干扰；
	2. VSync：发出该信号时，就是两个缓冲区交换内存地址的时机；

### 两个概念
1. 刷新频率，refresh rate：屏幕在一秒内，可以刷新的次数；比如 60Hz；
2. 帧率，frame rate：GPU 每秒可以画的帧数；比如
3. 帧率比刷新频率快的时候，就会出现画面撕裂；
4. 屏幕的显示，是由上至下扫描展示的；当屏幕扫描完，到最后一行的时候，放松 vsync 信号，通知缓冲期交换地址指针；

### API 16 后新增的优化机制
VSync 的引入，还残留一个问题没解决：刷新依赖于 VSync 信号，没有 VSync 不能刷新，如果 VSync 来时，缓冲区的数据没有准备好，就会显示上一帧的内容，就会出现卡顿。

- 解决方案：新增一个缓冲期，变成 三缓冲区。能保证帧是连续的。
- 本质原因，是帧的生成时间太长导致的。

### 重绘
1. 调用 requestLayout 会导致该 view 已经其递归上去的父 view都会重新走一遍measure、layout、draw三大流程。 
2. 调用invalidate只会导致draw流程的重走。


## SurfaceView 与 TextureView
### SurfaceView
我们知道每个窗口在SurfaceFlinger服务中都对应有一个layer，用来描述它的绘制表面surface。**对于那些具有SurfaceView的窗口来说，每个SurfaceFlinger服务中还对应一个独立的Layer或者LayerBuffer**，用来单独描述它的绘制表面，以区别它的宿主窗口的绘制表面。
surfaceView在其宿主Activity窗口上挖了一个“洞”，这个“洞”实际上只不过是在其宿主Activity窗口上设置了一块透明区域

优缺点
-   **优点：**由于在系统中（WMS和SF）中，它与宿主窗口是分离的。因此：Surface的渲染可以放到单独线程去做，渲染时可以有自己的GL context。这对于一些游戏、视频等性能相关的应用非常有益，因为它不会影响主线程对事件的响应。
-   **缺点:** 因为这个Surface不在View hierachy中，它的显示也不受View的属性控制，所以不能进行平移，缩放等变换，也不能放在其它ViewGroup中，一些View中的特性也无法使用


### TextureView
在4.0(API level 14)中引入，与SurfaceView一样继承View，它可以将内容流直接投影到View中，它可以将内容流直接投影到View中，可以用于实现Live preview等功能。
1. 它不会在WMS中单独创建窗口，而是作为View hierachy中的一个普通View，因此可以和其它普通View一样进行移动，旋转，缩放，动画等变化。
2. 值得注意的是TextureView必须在硬件加速的窗口中。它显示的内容流数据可以来自App进程或是远端进程。



# 各种 token
## ActivityRecord 中的 token 是啥时候创建的？作用是什么？
- [可参考的链接1](https://www.cnblogs.com/mingfeng002/p/10951883.html) ，这个文章写得很好。

- 时机：在创建 ActivityRecord 对象的时候，就会创建 token。
- 作用：启动一个Activity的时候会为这个 Activity 生成一个 ActivityRecord 对象，该对象用于 AMS 管理跟踪，而 Token 就在这里诞生了。Token 类实现了 IApplicationToken.Stub，也就是作为 Binder 的服务端，那么它自然的接收客户端的请求，那它主要提供什么样的服务呢?

*android/view/IApplicationToken.aidl*
```java
interface IApplicationToken
{
    void windowsDrawn();
    void windowsVisible();
    void windowsGone();
    boolean keyDispatchingTimedOut(String reason);
    long getKeyDispatchingTimeout();
}
```

可以看出，大部分是用于接收 WindowManagerService 通知 ActivityManagerService 的关于 Window 的消息，也有 key 的相关消息。



## WMS 中的 token 是什么时候创建的？作用是什么？
- 是什么：WMS专为Activity实现了一个 WindowToken 的子类：AppWindowToken

- 时机：具体位置为 ActivityStack.startActivityLocked()，也就是启动 Activity 的时候，通过 WMS 根据  ActivityRecord 中的 token，一起创建了 AppWindowToken。
- 作用：WindowManagerService 中 AppWindowToken 保存着 ActivityManagerService Binder 对象，用来向AMS 传递 Window 和按键的一些信息.

 wms 中有与  ams 中一一对应的 stack 和 task；

## App 中的 token

ActivityClientRecord 是 activity 在 App 应用进程中的代表。其中保存了 AMS  这个 server 的 token 引用。

- 时机：scheduleLaunchActivity 方法中，进行了 ActivityClientRecord 的构建。这里 scheduleLaunchActivity 方法是通过 binder 进行 IPC 方法调用的。   
- 作用：这个 ActivityClientRecord 类是 Activity 在 ActivityThread 中一一对应的，一个APP有多个Activity, 也就是说有多个ActivityClientRecord， 那么当 AMS 要启动一个 Activity 的时候，怎么样找到 APP 中正确的那个Activity呢？答案就是通过Token,先通过token找到ActivityClientRecord,然后再通过ActivityClientRecord中的activity就找到了正确的Activity了。Activity中Token主要用于在请求AMS服务时用于定位到具体到AMS中正确的 ActivityRecord。

## WindowManager.LayoutParams里的token
- 时机：setView 的时候，构建 WindowManager.LayoutParams 的 token 时，用 Window 中的 mAppToken 也就是AMS中ActivityRecord中的Token。
- 作用：WindowManager.LayoutParams中的 token传递给WMS,另外它的大部分作用是一致性判断。






# badToken 问题案例
那为什么窗口显示的时候会出现BadTokenException？

很明显是WindowManagerService 的[mWindowMap](http://androidxref.com/8.0.0_r4/xref/frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java#mWindowMap) 已经找不到对应的token，那为什么会出现这种情况呢？从表面上看那肯定是addview的时候，activity或者toast生成的token，已经被清除了。

我们来分析上述的问题，BadTokenException能够产生的情形蛮多的，就当前的调用栈看，是在handleResumeActivity的时候出现问题。

1. 情况一
ActivityManagerService 里面成员变量 mHandler 的 Looper，不会被Client-Side（即APP进程的操作所影响）。它在调用 destroyActivityLocked 的时候会先给Handler发一个延迟的DESTROY_TIMEOUT_MSG，如果client端顺利执行完了destroy，则会通知ActivityStack移除这个MSG，否则它就会被执行。
系统在调用 Activity 的 destroy 操作的时候，出现超时导致系统 mHandler 执行 DESTROY_TIMEOUT_MSG，测试log：
那什么情况下，在没有调用这些生命周期的情况就直接调用了destory。

1）还未完成resume的情况下，主动调用clean task;
2）在manifest中注明了activity是noHistory=true，则会在切换activity的时候，当前activity stop 的时候会执行requestFinishActivityLocked：
3）Intent 启动参数，设置FLAG_ACTIVITY_CLEAR_TOP，跳转到之前到activity；


解决方案
**主线程里面不能有大量耗时操作，这个DESTROY_TIMEOUT为10s**
**1.避免在Application与Activity的onCreate及onActivityResult阶段做耗时操作。**
**2.尽量把noHistory去掉，stop不强制触发finish，不发送DESTROY_TIMEOUT，AMS不再删除WindowToken。**
**3.根据业务逻辑判断较早的基于FLAG_ACTIVITY_CLEAR_TOP启动Activity后当前Activity是否具有耗时操作，进一步优化。**


2. 情况二
延迟addView
目前最常见的堆栈，也是日常编码极其容易触发的BadTokenException就是多线程操作 Dialog，在执行窗口显示的时候Activity已经被销毁了，可以看下边的调用栈：

能够很清晰的看到Dialog.show的时候找不到token，这类问题一般有两种情形导致：

1.多线程操作，未准确判断生命周期直接进行窗口展示，当Activity已经destroy后，线程回调继续使用该context进行页面展示。
2.通过hanlder发送Runnable，而该runnable中有ui操作的流程, 在onPause之后没有及时清理掉这些未处理的消息导致destory后又触发了ui操作。


解决方案
1. 线程回调中执行需要判断context，currentActivity.isFinishing()，能够部分拦截；
2. 在show的时候catch (WindowManager.BadTokenException e)；




# 热修复
https://mp.weixin.qq.com/s?__biz=MzAwNDY1ODY2OQ==&mid=2649286384&idx=1&sn=f1aff31d6a567674759be476bcd12549&scene=4#wechat_redirect

## 类加载
### Android中ClassLoader的种类&特点
-   BootClassLoader（Java的BootStrap ClassLoader）： 用于加载Android Framework层class文件。
-   PathClassLoader（Java的App ClassLoader）： 用于加载已经安装到系统中的apk中的class文件。
-   DexClassLoader（Java的Custom ClassLoader）： 用于加载指定目录中的class文件。
-   BaseDexClassLoader： 是PathClassLoader和DexClassLoader的父类。

## 热修复的原理
在app重启时，通过classloader抢先加载补丁的类，由于app在运行时需要修复的类已经被加载，运行时无法对类卸载，因此必须需要重启才能生效。

### Dalvik下的类校验问题
当dex加载到内存时，如果不存在odex文件会做dexopt将其转换成odex，在dexopt的过程中会做dvmClassVerify，如果某个类及其直接引用的类都在同一个dex的情况下，该类会被打上CLASS_ISPREVERIFIED标志，在dvmResolveClass的时候会做校验，也就是以下的代码，当补丁类加载的时候，如果满足这三个条件，就会校验失败抛出异常

-   fromUnverifiedConstant=false，非const-class/instance-of指令引用补丁类
-   IS_CLASS_FLAG_SET(referrer, CLASS_ISPREVERIFIED)，补丁类被打上CLASS_ISPREVERIFIED
-   referrer->pDvmDex != resClassCheck->pDvmDex，补丁类和引用类不在同一个dex


## 方案分类
### 【native修复】Andfix
#### 原理
通过env->FromReflectMethod可以得到Method对象的ArtMethod起始地址，然后将其强转为ArtMethod指针，从而对方法结构进行修改。
#### 问题
- 不能对原有类进行方法和字段的增减，都会导致索引发生变化，从而破坏原有类的结构，访问时就无法索引到正确的方法和字段
- 依赖Android底层源码，如果厂商修改了虚拟机方法，替换机制就可能出问题。


### 【native 修复】Sophix
https://developer.aliyun.com/article/103527

#### 原理
Sophix在Andfix基础上提出了一种新的思路，通过计算地址偏移对ArtMethod整体做替换，从而解决了底层结构兼容性问题。从方法创建的源码上看，类的方法数组是通过开辟连续的空间，然后逐个new出来。也就是说ArtMethod的内存地址排列是连续的，因此可以通过两个方法的地址的差计算出。ArtMethod的大小，然后通过memcpy直接拷贝即可完成方法的替换。

#### 问题
- 依赖ArtMethod的内存排列结构；
- 不能对原有类进行方法和字段的增减；
- 非静态修复类不能被反射调用，由于反射的对象是运行时的，而修复方法的类新加载的，不一致导致校验失败；

### 【Instant Run方案】Robust
#### 原理
通过gradle的transform在编译期插桩，对每个类中插入一个变量changeQuickRedirect，并且在所有方法预置一段逻辑，如果changeQuickRedirect不为空，则重定向到补丁方法执行，否则执行原来的方法。

补丁下发后通过DexClassLoader加载patch.dex，反射拿到 PatchesInfoImpl 类，里面保存了混淆前后的名字，比如说上面的 *com.meituan.sample.State* 被混淆成 *com.meituan.sample.d*

通过反射拿到com.meituan.sample.d，将changeQuickRedirect字段赋值用patch.dex中加载出来的StatePatch对象。

####  问题
-   包体积问题，插桩会导致包体积变大
-   dex中method id不能超过65536限制，主要原因是小函数插桩函数变大，容易导致无法内联，解决方式通过配置黑名单过滤小函数
-   super调用问题，补丁类无法直接通过super调用父类方法

解决方法：修改原始类的调用指令，改成invoke-super，由于invoke-super在调用时会校验当前类，补丁类需要继承原始类的父类，举个例子解释一下


### 【类加载】QQ 空间

#### 原理
QZone的思路是绕过第二个条件，预先在apk里面增加一个辅助类并单独打成一个dex，通过插桩的方式给所有类增加对辅助类的引用，从而让所有类不被打上CLASS_ISPREVERIFIED标志

#### 问题
-   Dalvik下绕过dexopt导致在类加载的时候才做校验，影响加载性能
-   art下dex2oat时同dex的类存在内联优化可能，如果补丁原来的类被内联了，且补丁类新增方法，内联类调用补丁类时方法、字段索引都是原来的，从而导致内存地址错乱，解决办法是将补丁类的父类和调用类都打到补丁包，但这样会导致补丁包变得很大。


### 【类加载】QFIX
#### 原理
QFix的思路是绕过第一个条件，在加载补丁类的时候，手动在jni层调用dvmResolveClass，设置fromUnverifiedConstant=true，提前调用后会有缓存，下次执行就直接从缓存获取从而绕过校验。


#### 问题
-   不支持多态，由于时机在dexopt之后，optimize阶段会将invoke-virtual指令替换成invoke-virtual-quick，也就是将方法直接替换成vtable里面的索引，当新增virtual方法后会导致vtable索引错乱，从而导致方法调用异常
**多态是怎么实现的**

-   每个类在解析之后对所有的虚方法会生成会有一个vtable表，子类的vtable表会与父类的vtable表合并，通过比对方法签名，优先删去父类的保留子类的，否则添加到末尾，而invoke-virtual指令是通过遍历类的vtable表找到方法索引，然后通过索引在对象的vtable找到实际方法调用



### 【类加载】Tinker
#### 原理
Tinker的思路是绕过第三个条件，自研了一套DexDiff算法，通过差量的方式打出补丁包下发，然后将patch.dex和应用变更的classN.dex合并成完整的dex，然后对完整的dex进行替换。
**合并dex**
app启动后开启独立进程，自研了一套算法进行合并
-   Dalvik：对每个需要修复的dex合成新dex，然后做dexopt
-   Art：多个dex打包成zip包，由于动态加载的dex2oat默认是speed模式全量编译，为了避免耗时过长导致anr，通过命令先做quiken/interpret-only模式的dex2oat，通过解释模式先运行，然后后台再做全量编译

#### 问题

-   dex合并在vm heap上进行，容易oom，可以优化成在native进行，native内存不受vm限制


### 【类加载】JVMTI

JVMTI：JVM Tool Interface，Java虚拟机定义的开发和监控JVM使用的接口，通过该接口可以探查JVM内部的运行状态，以及控制JVM应用程序的执行。



## So修复
### 原理

**替换加载方式**

android提供了两个方法加载so

```
System.load(String filename);//可以加载自定义路径下的so
System.loadLibaray(String filename);//用来加载已经安装APK中的so
```

最简单的方式就是包装一下这两个方法，如果有补丁so则通过`System.load`加载，否则通过`System.loadLibaray`加载，不过该方式对代码有侵入性，无法修复三方库的so

**反射替换加载路径**

类似类加载的方式，so加载的时候实际也是从classloader中查找路径，可以把补丁so库的路径插入到nativeLibraryDirectories数组的最前面，优先加载补丁库达到修复的目的。




# 插件化
- [参考链接1](https://bytedance.feishu.cn/wiki/wikcnwJbYjeGOIEtWLlGGFBZzhe)

## 目的
插件一般也是以一个apk的文件形式存在，然后被宿主apk动态加载并运行。这么做的好处是：
1. 减小包大小。例如用户不常用的功能可以单独打包成一个插件apk，这样可以减小宿主apk的大小。
2. 插件apk可以独立更新。
3. 宿主和插件分开编译，提升开发效率。

## classloader 相关的内容

类加载原理和"父委托"查找机制，大致分为三步，简单作如下说明：
-   第一步：当前ClassLoader.findLoadedClass()，尝试从已经加载过的缓存中读取。有则直接返回，无则进入下一步；
-   第二步：如果存在parent，则委托parentClassLoader.loadClass()，自低向上依次委托，一致到BootClassLoader；
-   第三步：自顶向下依次findClass()。

## SO 文件的安装处理
- [APK安装流程详解4——安装中关于so库的那些事](https://cloud.tencent.com/developer/article/1199441)
### ClassLoader与SO的关系
-   加载：System.load(path)/System.loadLibrary(name)，能否成功，取决于目标so文件能否找到，或由当前ClassLoader.nativeElements[]内部找到，或由外部传入。
-   打开：能否成功，一个so只能被同一个ClassLoader打开，打开次数不限制。否则报错：already opened by ClassLoader xxx, can't open in ClassLoader xxx
-   native方法调用：如果是JNI调用，能否成功，取决于当前JNI接口类的classLoader是否是打开它的classLoader，否则报错：No implementation found for xxx。
-   dlopen/dlsym/dlclose：如果是dlopen形式的打开、方法调用、关闭，取决与加载依赖它的so的ClassLoader的能否找到被依赖的so，如果找不到，主动调用System.load()，然后在dlopen也行。但是这个so库只能被同一个ClassLoader load一次。


## 插件的整体的流程
1. 时机
	1. 主动：业务方主动调用加载方法；
	2. 被动：业务方调用了插件中的类；
2. 核心操作：hook Instrumentation，创建自己的 Instrumentation，代理原有ActivityThread.mInstrumentation；
3. 流程详解
	1. 创建 applicationinfo；
	2. 创建 loadapk；
	3. 创建 plugin 的 classloader；
	4. 替换资源；
	5. 注册广播；
	6. 安装 ContentProvider；
	7. 创建 application；
	8. 替换 packageManager；
	9. 回调 application 的 oncreate；


## Activity 组件化的处理方式
对于Activity的插件化来说主要是两步：
- 如何让未在AndroidMainfest.xml注册的Activity通过AMS的校验
- 如何让未注册的Activity正常启动

### 业内方案
#### Qigsaw
`Qigsaw`对于第一步来说它的处理是将插件和宿主编译的时候，将插件内`AndroidMainfest.xml`的`Activity`信息复制到宿主的`AndroidMainfest.xml`内，从而第一步通过校验。

对于第二步来说，`Qigsaw`插件化框架修改了`Classloader`的实现，添加了插件相关的`Path`，从而导致即使是插件内的类也可以加载,从而实现`Activity`插件化。

**优点：**
-   稳定且兼容性强
-   未使用`Hidden Api`
-   `Hook` 少
-   改动少

**缺点：**
-   宿主和插件要在同仓下编译
-   无法动态新增`Activity`（宿主的`AndroidMainfest.xml`是编译时修改，无法通过更新插件升级）


#### Replugin
`Replugin`对于第一步来说，它是编译期间将生成`Activity`的基类，然后强制插件内的`Activity`继承此`Activity`,然后重写了其`startActivity`的方法，改为使用`Replugin`内的实现。这样可以按照一般插件化的方案来处理第一步，优点是没有`Hook`系统类。

对于第二步来说，`Replugin Hook` 了`Classloader` ,所以其可以在`Activity`创建的时候，将第一步中存储的插件的`Activity`的类名和占桩的`Activity`的类名读取出来，然后返回真正的插件的`Activity`的类即可。

**优点：**

-   稳定且兼容性强
-   未使用`Hidden Api`
-   `Hook` 少
-   改动少

**缺点：**

-   需要生成代码和修改原代码


#### Mira

对于`Mira`来说，其第一步`Hook`了`Instrumentation`，当你执行了`Activity#startActivity`的时候，内部执行了`Instrumentation#execStartActivity`方法，所以目前`Hook`了这个方法，用于存储插件`Activity` 和 占桩`Activity`的关系，并将插件`Activity`替换成占桩`Activity`，完成第一步。

对于第二步来说，`Hook` 了`ActivityThread`的内部类`H`,它是个`Handle.Callback`，

这个地方需要注意，在`Android 7` 以上会触发`ActivityThread.H.EXECUTE_TRANSACTION` 消息，以下则会触发`ActivityThread.H.LAUNCH_ACTIVITY` 消息，版本不同消息类型不同，然后针对这俩消息中`Intent`存储的占桩`Activity`信息还原成插件`Activity`的信息。且此时也需要修改`Classloader`,用于加载插件`Activity`。

**优点：**

-   稳定性强

**缺点：**

-   兼容性一般（需要针对`Android7.0`前后版本做适配）
-   使用了`Hidden Api`
-   `Hook`稍多
-   改动稍多



## Resource 资源的插件化
资源插件化的过程可总结为三步：
1. 第一步，Application启动时hook替换Application及LoadedApk的resouces到我们自定义的 WrapperResources 上来；
	1. 反射调用AssetManager.addAssetPath(String path)方法，我们把插件APK的路径传递进去，就可实现访问插件的资源。App中有多少插件就调用多少次，把所有插件的资源都add进去，构造出一个"超级Resources"为全局共用，查找引用，把宿主与插件中需要使用资源的地方都替换为这个"超级Resources"即是。这是资源插件化的第一步。
2. 第二步，插件启动时把插件的资源add到全局资源中，分配不同的段位与resId；
3. 第三步，Activity在启动时hook替换resouces到MiraWrapperResources实例上。


在以下几个时机需要注意替换 resource 引用、AssetManager 引用
- application 启动
- activity configuration 变化
- 新建 activity 的时候；


### ID 冲突的解决方案
每个资源都有一个对应的 ID 值 0xPPTTEEEE，由于宿主和插件都是各自打包，开发插件 App 和开宿主App的流程基本是一样的。所以在合并插件资源到宿主时，某个资源 ID 值有可能与宿主资源的 ID 是相同的，即资源 ID 冲突，Mira 的解决方案是通过修改 AAPT 打包工具为插件资源 ID 分配新段位，和固定宿主 App 中需要共享给插件的资源 ID。


## service 的插件化
整体的流程和 activity 的插件化很像，分为上半场和下半场。
- 上半场一样；
- 区别在于下半场；
	- 因为 start 就是简单的启动一个 Service 组件，所以可以简单理解为只有一次交互消息，而bindService 执行时会有两次消息传递，第一次是 AMS 通知 ActivityThread 创建Service的消息，第二次是 AMS 传递 ServiceConnection 对象到 ActivityThread 的消息。
	- 因为 stop 是简单的停止一个 Service 组件，只有一次交互消息，且消息体中包含了Intent 信息；而 unbind 有两次交互消息，且 stopService 消息体中没有 Intent 信息，只有 IBinder token 信息。

### 具体步骤
-   上半场我们选择hook AMN，即创建IActivityManagerProxy作为AMN的代理对象。我们只要"欺骗AMS"要start/stop、bind/unBinder的Service已在AndroidManifest中存在就好。在经过AMN发送Intent信息到AMS之前，把该Intent上的targetService替换为一个在AndroidManifest中声明的StubService，把原来的targetService信息存放在Bundle的扩展字段位上即可。
- 下半场我们选择hook ActivityThread.H.mCallback，拦截AMS通知App启动或绑定StubService的消息。在即将启动或绑定StubService之前，把StubService换回原先保存在Bundle扩展位上的targetService。


## Receiver的插件化
静态广播需要在AndroidManifest中声明，应用安装和Android系统重启时，PMS都会解析App的Manifest，所以静态广播的注册信息位于PMS中。动态广播是在代码中调用Context.registerReceiver() --> AMN.getDefault().registerReceiver()手动注册的，所以注册信息存在于AMS中。二者区别仅在于注册方式不同，之后就都一样了。
我们就可以把静态广播都改为动态广播，避免在AndroidManifest中声明，也就避免了AMS检查。这就是Receiver的插件化思路。


动态广播不需要和AMS打交道，所以它就是一个普通类。我们只要确保宿主App能加载插件中的这个动态广播类即是。在ClassLoader相关章节中介绍过全局的类加载器，有它作保证，我们可以加载任一插件中的任何一个类。这样，插件中的动态广播就可以被宿主App正常调用了。


-   时机：Mira 在使用一个插件前，都必须要preload()预加载。其中包括很重要的一步是将插件中的静态广播转为动态广播注册在宿主中。即插件初始化调用之前，我们取得插件中的静态广播列表，调用相应插件的PluginClassLoader创建实例，以动态广播的形式注册进宿主中。

这种方案使得丧失了静态广播的特性，体现在不需要启动App就可以启动一个静态广播。当然这里我们不能浅显的认为，App的启动一定是打开某个Activity，通常情况下启动首页的MainActivity只是启动App的一种。Receiver作为四大组件之一，也支持外部调起，进而创建进程，启动应用，接收到广播。尤其推送Push场景下，当App进程已死后，其它伙伴应用可通过发送静态广播来唤醒App。当采用了这种静态转动态的方案后，插件中的所有静态广播特性丧失，即不在支持在App没启动和插件没就绪的情况下，接收到广播。后续我们会对此方案持续改进。
## ContentProvider的插件化
### 主进程插件化
比较简单。类似Receiver的插件化解决方案，把插件中声明在Manifest的静态广播转动态广播"注册"在宿主中。其实Provider也可以这么实现，这时不叫"注册"，叫"安装"。该方法位于 [ActivityThread.installContentProvider](http://aosp.opersys.com/xref/android-8.0.0_r1/xref/frameworks/base/core/java/android/app/ActivityThread.java#5838)方法中，我们可以反射调用该方法，把插件中的Providers作为第二个参数填充进去即可。

- 时机：越早越好。因为Provider更多场景是给第三方App用得，插件中的Provider必须要安装在宿主中才能生效，越早安装，第三方App等待的时间就越短。App安装自身Provider时机是ActivityThread.installContentProvider()方法执行，会在App进程启动时立刻执行，比Application.onCreate()早，略晚于Application.attchBaseContext()。Mira在使用一个插件前，都必须要preload()预加载。其中包括很重要的一步是将插件中的Provider安装在宿主中。即插件初始化调用之前，我们取得插件中的Provider列表，修改providerInfo.applicationInfo.packageName包名为宿主包名，反射调用ActivityThread.installContentProvider()方法安装即可。所以如果可能，尽可能早得preload()插件。
-   参照PMS通过PackageParser解析Apk，Mira中PluginPackangeManager用于管理插件安装包相关内容，包括解析插件Apk，解析Manifest，创建Package，提取四大组件等。在此调用getProviders()获取插件中的Provider列表。


### 子进程插件化
有跨进程IPC操作，工作在本进程，主要为其它进程App提供数据的Provider，这类Provider的插件化方案比较复杂。直接让外界App或其它进程直接调用本进程插件里定义的Provider，并不是一个理想的解决方案，我们定义了一个StubContentProvider作为中转，让外界App先访问到中转Provider，进而分发到插件里的Provider上，这种设计得益于Provider独有的URI authority作为唯一定位符机制，URI是资源统一定位符，按照既定协议唯一标识一个确定的资源，但它本身就是一个字符串，所以非常适合用于这种转发机制。

