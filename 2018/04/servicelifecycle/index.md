# Service 的生命周期

# 概述
Service 有两种启动方式,一种是 startService ;一种是 bindService.这两种的启动的 Service 的生命周期有些许差异,并且当二者混用的时候,有一些需要注意的地方.这里用打印 log 日志的方式记录下生命周期的差异.

<!-- more -->

# 仅 startService

## 触发 startService 方法

```console
D/ServiceLifeService: onCreate() called
D/ServiceLifeService: onStartCommand() called with: intent = [Intent { cmp=cn.steve.study/cn.steve.servicelifecycle.ServiceLifeService }], flags = [0], startId = [1]
```


## 退出APP
无回调

## 再次进入APP，触发startService 方法

```console
D/ServiceLifeService: onStartCommand() called with: intent = [Intent { cmp=cn.steve.study/cn.steve.servicelifecycle.ServiceLifeService }], flags = [0], startId = [2]
```

## 杀死 App 进程
没有回调，直接被系统杀死。只有一些AMS系统回调

```console
W/ActivityManager: Scheduling restart of crashed service cn.steve.study/cn.steve.servicelifecycle.ServiceLifeService in 1000ms
I/ActivityManager: Force stopping service ServiceRecord{38ffaacf u0 cn.steve.study/cn.steve.servicelifecycle.ServiceLifeService}
```


## 调用 stop 方法

```console
D/ServiceLifeService: onDestroy() called
```

## 小结
对于 startService 而言
- 当要启动的 Service 没有创建时，就会创建，而后回调 onCreate  和 onStartCommand  方法；
- 当要启动的 Service 已经存在，则只会回调  onStartCommand 方法；
- 退出 App，Service 依旧存活
- 调用 stop 方法，回调 onDestroy


# 仅 bindService


## 触发 bindService 方法

```console
D/ServiceLifeService: onCreate() called
D/ServiceLifeService: onBind() called with: intent = [Intent { cmp=cn.steve.study/cn.steve.servicelifecycle.ServiceLifeService }]
D/ServiceLifeService: onServiceConnected() called with: name = [ComponentInfo{cn.steve.study/cn.steve.servicelifecycle.ServiceLifeService}], service = [ServiceBinder:cn.steve.servicelifecycle.ServiceLifeService$ServiceBinder@4bb38ec]
```

## 多次触发 bindService 方法

对于同一个 ServiceConnection 只会进行一次 ServiceConnection 中的方法回调。
若不同的 ServiceConnection ，则会进行 ServiceConnection 中的 方法回调

## 退出APP

```console
D/ServiceLifeService: onUnbind() called with: intent = [Intent { cmp=cn.steve.study/cn.steve.servicelifecycle.ServiceLifeService }]
D/ServiceLifeService: onDestroy() called
```

## 退出前，主动调用 unbind 方法 

```console
D/ServiceLifeService: onUnbind() called with: intent = [Intent { cmp=cn.steve.study/cn.steve.servicelifecycle.ServiceLifeService }]
D/ServiceLifeService: onDestroy() called
```

## 小结
对于 bindService 而言
- 当要启动的 Service 没有创建时，就会创建，而后回调 onCreate  和 onBind  方法；
- 当要启动的 Service 已经存在，则只会回调  onBind 方法；
- 退出 App，Service 将自动解绑，并回调 onDestroy 方法
- 退出 App 和主动 unbind 的回调一致


# 先 startService ，再 bindService

## 触发 startService 方法，再触发  bindService 方法

```console
D/ServiceLifeService: onCreate() called
D/ServiceLifeService: onStartCommand() called with: intent = [Intent { cmp=cn.steve.study/cn.steve.servicelifecycle.ServiceLifeService }], flags = [0], startId = [1]
D/ServiceLifeService: onBind() called with: intent = [Intent { cmp=cn.steve.study/cn.steve.servicelifecycle.ServiceLifeService }]
D/ServiceLifeService: onServiceConnected() called with: name = [ComponentInfo{cn.steve.study/cn.steve.servicelifecycle.ServiceLifeService}], service =[ServiceBinder:cn.steve.servicelifecycle.ServiceLifeService$ServiceBinder@598f7d8]
```

## 退出APP

```console
D/ServiceLifeService: onUnbind() called with: intent = [Intent { cmp=cn.steve.study/cn.steve.servicelifecycle.ServiceLifeService }]
```

## 调用  unbind 方法

```console
D/ServiceLifeService: onUnbind() called with: intent = [Intent { cmp=cn.steve.study/cn.steve.servicelifecycle.ServiceLifeService }]
```


## 在 unbind 之前，调用 stop 方法

无回调

## 调用 stop 方法，再调 unbind

```console
D/ServiceLifeService: onUnbind() called with: intent = [Intent { cmp=cn.steve.study/cn.steve.servicelifecycle.ServiceLifeService }]
D/ServiceLifeService: onDestroy() called
```


## 调用 unbind，再调用 stop 方法

```console
D/ServiceLifeService: onUnbind() called with: intent = [Intent { cmp=cn.steve.study/cn.steve.servicelifecycle.ServiceLifeService }]
D/ServiceLifeService: onDestroy() called
```

## 小结
和前面的方式类似，有以下几点需要注意
- bindService 只会导致 Service 回调 onBind 方法，因为 Service本身已经存在，所以，不会再回调 onCreate 方法，也说明 Service 是同一个 Service
- 退出 App 后只会让 Service 回调 onUnbind 方法，受 startService 方法的影响， Service 依旧存活，故而不会回调 onDestroy。


# 先 bindService ，再 startService

## 触发 bindService 方法，再触发  startService 方法

```console
D/ServiceLifeService: onCreate() called
D/ServiceLifeService: onBind() called with: intent = [Intent { cmp=cn.steve.study/cn.steve.servicelifecycle.ServiceLifeService }]
D/ServiceLifeService: onServiceConnected() called with: name = [ComponentInfo{cn.steve.study/cn.steve.servicelifecycle.ServiceLifeService}], service = [ServiceBinder:cn.steve.servicelifecycle.ServiceLifeService$ServiceBinder@4bb38ec]
D/ServiceLifeService: onStartCommand() called with: intent = [Intent { cmp=cn.steve.study/cn.steve.servicelifecycle.ServiceLifeService }], flags = [0], startId = [1]
```


## 退出APP

```console
D/ServiceLifeService: onUnbind() called with: intent = [Intent { cmp=cn.steve.study/cn.steve.servicelifecycle.ServiceLifeService }]
```


## 调用  unbind 方法

```console
D/ServiceLifeService: onUnbind() called with: intent = [Intent { cmp=cn.steve.study/cn.steve.servicelifecycle.ServiceLifeService }]
```

## 在 unbind 之前，调用 stop 方法

无回调

## 调用 stop 方法，再调 unbind

```console
D/ServiceLifeService: onUnbind() called with: intent = [Intent { cmp=cn.steve.study/cn.steve.servicelifecycle.ServiceLifeService }]
D/ServiceLifeService: onDestroy() called
```


## 调用 unbind，再调用 stop 方法

```console
D/ServiceLifeService: onUnbind() called with: intent = [Intent { cmp=cn.steve.study/cn.steve.servicelifecycle.ServiceLifeService }]
D/ServiceLifeService: onDestroy() called
```

## 小结
和前面的方式类似，有以下几点需要注意
- bindService 调用产生的回调和单独 调用 bindService 一样；
- startService 调用时，因为 Service 对象已经存在，所以只回调 onStartCommand 方法
- 退出 App 后只会让 Service 回调 onUnbind 方法，受 startService 方法的影响， Service 依旧存活，故而不会回调 onDestroy。



# 总结

- 已经创建成功的 Service，在运行期间，无论是 start 还是 bind，都不会调用 onCreate；
- Service 在运行期间，每次调用 start 都会触发 onStartCommand 方法；而多次用同一个 ServiceConnection 进行bind 调用时，只会触发一次 onBind ；对于用不同的 ServiceConnection 进行 bind 时，都会触发 ServiceConnection 中的 onServiceConnected 方法，但并不会触发 onBind 方法，所以，相遇于多次触发 onStartCommand 触发，对于 bind 操作来说，onBind 只会触发一次。
- 当 start 和 bind 混合调用时，要想停止，必须要调用 unbind 才会回调 onDestory 方法。若先调用 stop，此时没有任何回调，再调用unbind时，会回调onUnbind，同时进行 onDestroy 回调。反之，若先调用 unbind，再调用 stop，回调顺序一致。

