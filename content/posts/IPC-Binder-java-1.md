---
title: IPC_Binder_java_1
date: 2017-01-03 21:30:55
tags: [IPC,Binder]
categories: [Mobile,Android]
---
## 背景
由于 Binder 很复杂,这个分多篇展开,目前先将零碎的知识整合,在后面几篇进行总结.

## 概述:
Binder 用于进程间通信，而 Handler 消息机制用于同进程的线程间通信。
Binder 的英文涵义是别针，回形针的意思。
在 Android 中 Binder 的存在是为了完成进程间的通信，将进程"别" 在一起。比如说：普通应用可以调用播放器提供的服务：播放、暂停、停止等功能。
Binder 是工作在 Linux 层面，属于一个驱动，只是这个驱动是不需要硬件的，或者说是基于操作系统的一小块内存。从线程的角度来讲，Binder 驱动的代码是运行在内核态的，客户端程序调用 Binder 是通过系统调用完完成。

<!-- more -->

### Binder 框架：一种架构

Binder 框架提供 服务端接口、Binder 驱动、客户端接口 三个模块。
![Binder架构](https://github.com/tinggengyan/tinggengyan.github.io/blob/source/imgur/Binder%E6%A1%86%E6%9E%B6%E5%9B%BE.png?raw=true)

1. 从服务端的角度来说，一个 Binder 服务端实际上就是一个 Binder 类的对象，该类一旦创建，内部就会启动一个隐藏线程。该线程接下来就用于接收 Binder 驱动发送来的消息，收到消息之后，会执行到 Binder 对象中的 onTransact 方法，在这个方法中，根据不同的参数，执行不同的服务代码。因此，要实现一个 Binder 服务，就必须重载 onTransact 方法。
在 onTransact 方法中，会获取传递进来的参数，将其转换成服务函数的参数。onTransact  参数的来源于 客户端的调用  transact 方法。所以，如果  transact 方法的参数有固定的格式输入，那么 onTransact 就会有相应的固定格式输出。

2. 从 Binder 驱动的角度来说。任何一个服务端的 Binder 对象被创建的时候，都同时会在 Binder 驱动中创建一个 mRemote 对象，这个对象也是 Binder 类。客户端想要访问远程服务的时候，都是通过这个 mRemote 对象。

3. 从客户端的角度来说。要想访问远程服务，必须先获取远程服务在 Binder 驱动中对应的 mRemote 引用，在获取该对象之后，就可以调用 transact 方法，而在 Binder 驱动中，mRemote  对象也重载了 transact 方法。
- 以线程间消息通信的模式，向服务端发送客户端传递过来的参数。
- 挂起当前的线程，当前线程正是客户端线程，并等待服务端线程执行完指定服务函数之后通知。
- 接收服务端线程的通知，然后继续执行客户端线程，并返回客户端代码区。

从以上的叙述中，可以看出，对应用开发者来说，客户端似乎是在直接调用了远程服务对应的 Binder ，而事实上，则是通过 Binder 驱动中的 Binder 对象，不同的是， Binder 驱动中的对象不会额外产生一个线程。

- 简言之:
客户端将消息发至 -> Binder 驱动 ，向服务端发送调用信息，驱动挂起当前线程 ，等待返回-> 服务端 ，处理完消息，返回给驱动-> 驱动接到完成的通知，继续客户端的线程 - >返回结果给客户端。
连接他们的是一个叫 mRemote 的引用，这个引用存在于 Binder 驱动当中，每个服务端的都需要向 Binder 驱动注册，生成这个 mRemote 引用。
客户端利用这个引用去发送消息给驱动，驱动利用这个引用去发送消息给服务端， 整个过程像客户端直接调用了服务端，事实上是通过 Binder 驱动中转了，存在两个 Binder 对象，一个是服务端的 Binder 对象， 一个是 驱动中的 Binder  对象，区别中，Binder 驱动中不会产生额外的线程，而服务端的 Binder 在创建之初就有一个隐含的线程。


### 设计 Server 端

设计 server 端只需要新建一个继承 Binder 的 service 即可，当启动这个 service 的时候，在 ddms 中的 thread 会发现多了一个 Binder thread 。
定义完  service ，接下来需要重载 onTrasact 方法，并从 data 变量中读取客户端传递进来的参数。 假如，这里有很多参数，那么怎么知道参数的顺序呢？所以，这个需要一个双方的约定。
方法的第一个参数 code 是用来标记不同的服务端函数的。
如果想要返回结果，则在 reply 中调用相关的函数写入即可。

### 设计 Binder  客户端

对于客户端要想使用服务端的服务函数，则必须先获取服务端在 Binder 驱动中对应的 mRemote 对象。在获取到该对象之后，就可以调用该变量的 transact 方法。
public final boolean transact(int code, Parcel data, Parcel reply,  int flags)
data 是传递给服务端的数据，远程服务函数的参数，都是从这个 data  中取的。
data  中能放的类型都是常用的原子类型，String，int ，long 等，当然也包括实现了 Parcelable接口的类。
这里向 data 写入的数据的顺序，必须和 onTransact  取参数的顺序保持一致，需要实现约定好。
当调用客户端调用远程方法，经由 mRemote 调用 transact 的时候，客户端线程进入 Binder 驱动， Binder 驱动就会挂起当前线程，并向远程服务发送一个消息，消息中包含了客户端传进来的包裹数据。
当服务端 service 执行 onTrasact  的时候，就可以对包裹 data 进行拆解，然后根据参数执行相应的 服务函数，执行完之后，会将执行的结果放入 reply 中。
当这一切都执行完之后，服务端会向 Binder 驱动发送一个 notify 的消息（客户端线程在调用 transact 的时候，客户端线程会被挂起），从使得客户端线程从 Binder 驱动代码区返回到客户端代码区。
对于最后一个参数 flags ，表示的是 IPC 调用的模式，分为：双向，用0 表示，含义是服务端执行完之后会返回一定的数据；还有一种是单向，用1 表示，含义是不返回任何数据。
同样的，返回到结果都是在 reply 中，客户端从这个 reply 中取的数据，这部分顺序也必须实现约定好。

### 使用 service
在编写 Binder 服务端和客户端的过程中，会伴随着两个问题。

  * 客户端如何获得服务端的 Binder 对象引用
  * 客户端和服务端约定关于顺序的顺序和服务函数的 int 标志。


使用 Binder 的原因是想提供一个全局的服务，所谓的全局，意思是系统中的任何程序都可以访问 。很明显，这个应该属于操作系统需要提供的基本功能之一，所以有个方法就是 service。

无论是否使用 service类，都需要解决上面的两个问题。

当然完全可以不使用 service 类，而是仅仅基于 Binder 类编写服务程序，然而这个只是一部分。具体来说，可以仅仅使用 Binder 类扩展系统服务，对于客户端服务则必须是基于 service 类来编写的。系统服务是指那些通过 getSystemService 方法获取的服务，而客户端服务是指应用程序提供的自定义服务。
也就是说，扩展系统服务的时候，可以完全只使用 Binder 类；而对于客户端的服务则必须基于 service。



#### 获取 Binder 对象

看下几个启动 service 相关的方法，这些方法在 android.app.ContextWrapper 类中。

```java
public ComponentName startService(Intent service) {
    return mBase.startService(service);
}
```

这个方法很熟悉，就是一个 启动服务的方法，然后启动之后，客户端并不能拿到服务端的 Binder 引用，因此并不能调用服务端的任何服务。

```java
 @Override
public boolean bindService(Intent service, ServiceConnection conn,   int flags) {
        return mBase.bindService(service, conn, flags);
}

public interface ServiceConnection {
    public void onServiceConnected(ComponentName name, IBinder service);
    public void onServiceDisconnected(ComponentName name);
}
```

bindService 方法第一个参数是启动 service 的intent ，第二个参数是一个接口，接口中有个方法叫 onServiceConnected 这个方法含有两个参数，第二个参数就是 Binder 。
具体的运行过程中，当客户端请求启动 service 的时候，请求就会通过 Ams 发出，若 service 正常去懂了，那么 Ams 就会远程调用 ActivityThread 类中的 ApplicationThread 对象，调用的参数就包含了 service 的 Binder  对象的引用，然后在 ApplicationThread  中回调 bindService 的第二个参数 ServiceConnection  的方法 onServiceConnected ，将 Binder 引用传递回客户端，这样客户端就拿到了远程服务的 Binder  对象引用，在实际操作中，常常可以这个 Binder 对象引用设置成一个全局变量，可以在客户端的任何地方都可以访问到。

![Binder客户端和服务端的调用过程](https://github.com/tinggengyan/tinggengyan.github.io/blob/source/imgur/Binder%E5%AE%A2%E6%88%B7%E7%AB%AF%E5%92%8C%E6%9C%8D%E5%8A%A1%E7%AB%AF%E7%9A%84%E8%B0%83%E7%94%A8%E8%BF%87%E7%A8%8B.png?raw=true)

### 保证参数顺序的工具-AIDL

在数据传递的过程中，需要实现约定好服务函数所对应的 code 的 int 值，需要约定好参数的写入顺序。在 Android 中的 AIDL 就是这么个工具。
AIDL 可以将一个 AIDL 文件转换成一个 Java  类文件，同时重载 transact 和 onTransact 方法。关于服务函数对应的 int 值和参数的读写书序，都统一做了处理。这样，开发者只需要专注于服务代码本身了。
可以看得出来，AIDL 并非是必须的，只是一个工具。

```java
interface ICompute {
    void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat,double aDouble, String aString);
    int add(int a, int b);
}
```

一般情况下，第一个字母是 I，这样是为了程序风格的统一，后面的 Compute 是服务的类名，AIDL 工具会以这个服务的名字生成 Java 类。( 当然这个是默认的，相应的也是可以修改的，具体的另行参照说明。 )
aidl 文件中可以引用其他的 Java 类，但是需要遵循以下要求：

-  Java 原子类型，int，long，String 等。
-  Binder 引用。
-  实现了 parcelable 接口的对象。

运行 AIDL 工具之后，生成的文件中，包含了 Java interface，proxy，stub类。
-  Java interface 是以 aidl 文件命名的。比如 ICompute 。产生的 interface就是 ICompute 。并且该类继承了 IInterface 接口，即，需要实现 asBinder 方法。
-  在 Stub 内部还有一个内部静态类 proxy ， 该类具体实现了 AIDL 生成的接口，按照约定的顺序写入参数，可以注意到这里的顺序和 Stub 中重载的 onTransact  中读取的顺序是一致的。proxy 类中持有了一个 IBinder mRemote 对象，这个对象就是远程服务端的引用，Proxy 该类作为客户端访问服务端的代理，该类的代理产生的原因：主要是为了解决约定写入参数的顺序。
-  内部有一个静态内部抽象类 Stub，这个类主要是由服务端使用 ，之所以是抽象类，因为具体的服务函数必须由程序员自己去实现。该类继承 Binder 类，并且实现 AIDL 生成的接口，但是没有具体的实现这个接口。该类也重载了 onTransact 方法，这个方法是去按照约定的顺序取参数中的值，因为是 ADIL 自己生成的，所以顺序，它自己很清楚；并且定义了服务函数对应的 int 值。asBinder 方法返回的就是 Stub 自身。它内部还有个非常重要的方法  asInterface ，这个方法根据参数 IBinder 对象是否是自身进程中的对象，返回不同的对象。因为我们知道，服务端的服务函数，不仅仅是别的进程可以使用，与 服务端在一个进程内部也可以调用，这种场景下，显然是不需要 IPC 的，而是直接调用。反之，则返回一个 proxy，交由跨进程的客户端引用。在 Binder 内部提供了 queryLocalInterface 方法根据描述符判断当前的 Binder 对象时不是本地的 Binder 引用。因为每当新创建一个 Binder 对象的时候，服务端进程内部会创建一个 Binder 对象，同时在 Binder 驱动中也会创建一个 Binder 对象。如若是跨进程调用，远程访问的时候，返回的 Binder 就会是 Binder 驱动中的 Binder 对象，如若是进程内部获取 Binder 对象，则会是服务端本身的 Binder 对象。所以，asInterface  是对外提供了一个 统一的接口，保证无论进程内还是进程外都能访问，返回的对象就两种，一个是 Proxy 类对象，一个就直接使用 Stub 本身，强制类型转换成 接口类型。

![Binder架构](https://github.com/tinggengyan/tinggengyan.github.io/blob/source/imgur/Binder_3.png?raw=true)

## 总结
本篇最后,放一张图进行总结.
![Binder绑定的流程](https://github.com/tinggengyan/tinggengyan.github.io/blob/source/imgur/Binder%E7%BB%91%E5%AE%9A%E7%9A%84%E6%B5%81%E7%A8%8B.png?raw=true)


## 感激,非常感激，万分的感激！
感谢以下的文章以及其作者和翻译的开发者们,排名不分先后
* [柯元旦](https://book.douban.com/subject/6811238/)
* [gityuan](http://gityuan.com/2015/12/26/handler-message-framework/)


