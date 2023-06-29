# DDMS_Threads的简单使用

## 概述
本文记录在 DDMS 如何查看线程的状态,以及状态表达的含义.

<!-- more -->

## 使用 DDMS 查看进程中的线程状态

### 简介

DDMS(Dalvik Debug Monitor Service),是 Android 开发的调试工具。

### 如何工作

在 Android 系统中每个应用都是在单独的一个进程中运行，DDMS 可以将一个进程通过 adb 和 IDE 连接，进行调试。

### 面板讲解

![DDMS面板](https://github.com/tinggengyan/tinggengyan.github.io/blob/source/imgur/ddms_panel.png?raw=true)

#### Threads
在左侧选中想要监控的进程，点击上方左起第五个图标(Update Threads) ,在对应的右侧打开 Threads 面板，就可以看到当前进程中的 线程状态。

##### 字段讲解

>* ID: 线程ID，是当前进程分配的唯一的线程ID.在 Dalvik 虚拟机中，这些值是从奇数3开始计数。
>* Tid: Linux 线程 ID， 对于一个进程的主线程而言，这个 ID 对应了进程 ID 。
>* Status: 该线程在进程中的状态，守护进程(Daemon thread)前面被标记了一个星号 ( * ) 。状态可取的值:
   - running: 正在运行的线程。
   - sleeping: 休眠的，等待被唤醒的线程。
   - monitor: 监视，正在等待获取一个监控锁。
   - wait: 执行了wait方法，释放了对象锁。
   - native: 正在执行 native 代码。
   - vmwait: 正在等待虚拟机的资源。
   - zombie: 僵尸线程，即将销往的进程的线程。
   - init : 初始化中的线程(理论上不应该看得到)
   - starting : 即将启动的线程(理论上不应该看得到)

>* utime: 花费在用户代码所花的累计时间，一小会儿(通常是10ms)。只有在linux环境下，才能看到。PS:windows 下。 DDMS 看得到，不知道他这里有啥特别的含义。
>* stime : 花费在系统代码上的累计时间，一小会儿(通常是10ms)。
>* Name: 线程名。

"ID" 和 "Name"  是在线程启动的时候被设置的。其他的字段是每过一段时间就更新一下(默认是4秒)

------

#### VM Heap

展示一些堆的统计数据，在 gc 的过程中会进程更新。如果选中一个 进程的时候，堆信息视图提示堆更新不可用，点击工具栏左上角的 "Show heap updates" 按钮，再回到 VM 堆视图，点击 "Cause GC" 进行垃圾回收，更新堆统计信息。



### 参考文献
[Using Dalvik Debug Monitor Service (DDMS)](http://www.linuxtopia.org/online_books/android/devguide/guide/developing/tools/ddms.html)



