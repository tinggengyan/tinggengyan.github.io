---
title: 一个例子说明如何使用  RxJava 进行线程切换
date: 2016-06-14 16:07:10
tags: [RxJava,Source,Thread]
categories: [Mobile,Android]
---
# 概述
本文仅作记录如何在上层使用代码进行 RxJava 的线程切换.

<!-- more -->

# RxJava 线程管理

RxJava 中通过两个关键的方法 subscribeOn 和 observeOn 实现线程的切换，都说 RxJava 是可以任性的随意切换线程，到底可以多任性呢，在哪任性呢，代码上怎么体现呢？下面通过一个非常简单的例子
演示一下如何使用，源码讨论请移步另一篇文章。

##　测试代码

```java

//验证多线程切换的情况
    private void multiThreadSwitch() {
        Observable.create(new Observable.OnSubscribe<String>() {
            @Override
            public void call(Subscriber<? super String> subscriber) {
                //受subscribeOn的影响，另起一个线程
                Log.e(TAG, "create " + Thread.currentThread().toString());
                subscriber.onNext("hello world");
            }
        })
            .subscribeOn(Schedulers.newThread())

            .doOnSubscribe(new Action0() {
                @Override
                public void call() {
                    //和上一个doOnSubscribe运行在同一个线程中,因为中间并未切换线程
                    Log.e(TAG, "doOnSubscribe4 " + Thread.currentThread().toString());
                }
            })
            .doOnSubscribe(new Action0() {
                @Override
                public void call() {
                    //受subscribeOn的影响，另起一个线程
                    Log.e(TAG, "doOnSubscribe3 " + Thread.currentThread().toString());
                }
            })
            .subscribeOn(Schedulers.newThread())

            .map(new Func1<String, String>() {
                @Override
                public String call(String s) {
                    //和create在同一个线程执行
                    Log.e(TAG, "map1 " + Thread.currentThread().toString());
                    return s;
                }
            })
            .observeOn(Schedulers.newThread())

            .doOnSubscribe(new Action0() {
                @Override
                public void call() {
                    //受subscribeOn的影响，另起一个线程
                    Log.e(TAG, "doOnSubscribe2 " + Thread.currentThread().toString());
                }
            })
            .map(new Func1<String, String>() {
                @Override
                public String call(String s) {
                    //受observeOn的影响，另起一个线程
                    Log.e(TAG, "map2 " + Thread.currentThread().toString());
                    return s;
                }
            })
            .subscribeOn(Schedulers.newThread())

            .doOnSubscribe(new Action0() {
                @Override
                public void call() {
                    //作为消息发送发送的第一站，没有通过subscribeOn指定发送消息的线程，故而这个会在调用subscribe方法的线程上执行,这里是主线程
                    Log.e(TAG, "doOnSubscribe1 " + Thread.currentThread().toString());
                }
            })

            .observeOn(Schedulers.newThread())
            .subscribe(new Action1<String>() {
                @Override
                public void call(String s) {
                    // 受observeOn的影响，另起一个线程
                    // 假如没有observeOn，则运行在离这 最近 的observeOn，或者 最远(物理位置最远，按照消息自下往上的顺序，其实也是最近) 的subscribeOn线程上
                    Log.e(TAG, "subscribe " + Thread.currentThread().toString());
                }
            });
    }

```



## 日志输出

对于直接运行测试代码，产生的log日志是如下的。

```java

06-14 16:36:09.315 5471-5471/ cn.steve.study E/RXJavaActivity: doOnSubscribe1 Thread[main,5,main]
06-14 16:36:09.319 5471-12993/cn.steve.study E/RXJavaActivity: doOnSubscribe2 Thread[RxNewThreadScheduler-7,5,main]
06-14 16:36:09.323 5471-12995/cn.steve.study E/RXJavaActivity: doOnSubscribe3 Thread[RxNewThreadScheduler-9,5,main]
06-14 16:36:09.323 5471-12995/cn.steve.study E/RXJavaActivity: doOnSubscribe4 Thread[RxNewThreadScheduler-9,5,main]
06-14 16:36:09.327 5471-12996/cn.steve.study E/RXJavaActivity: create Thread[RxNewThreadScheduler-10,5,main]
06-14 16:36:09.327 5471-12996/cn.steve.study E/RXJavaActivity: map1 Thread[RxNewThreadScheduler-10,5,main]
06-14 16:36:09.329 5471-12994/cn.steve.study E/RXJavaActivity: map2 Thread[RxNewThreadScheduler-8,5,main]
06-14 16:36:09.331 5471-12992/cn.steve.study E/RXJavaActivity: subscribe Thread[RxNewThreadScheduler-6,5,main]

```

对于去掉两个observeOn，产生的log日志是如下的。

```java

06-14 16:43:42.001 18160-18160/cn.steve.study E/RXJavaActivity: doOnSubscribe1 Thread[main,5,main]
06-14 16:43:42.002 18160-19180/cn.steve.study E/RXJavaActivity: doOnSubscribe2 Thread[RxNewThreadScheduler-4,5,main]
06-14 16:43:42.006 18160-19181/cn.steve.study E/RXJavaActivity: doOnSubscribe3 Thread[RxNewThreadScheduler-5,5,main]
06-14 16:43:42.006 18160-19181/cn.steve.study E/RXJavaActivity: doOnSubscribe4 Thread[RxNewThreadScheduler-5,5,main]
06-14 16:43:42.023 18160-19182/cn.steve.study E/RXJavaActivity: create Thread[RxNewThreadScheduler-6,5,main]
06-14 16:43:42.024 18160-19182/cn.steve.study E/RXJavaActivity: map1 Thread[RxNewThreadScheduler-6,5,main]
06-14 16:43:42.024 18160-19182/cn.steve.study E/RXJavaActivity: map2 Thread[RxNewThreadScheduler-6,5,main]
06-14 16:43:42.024 18160-19182/cn.steve.study E/RXJavaActivity: subscribe Thread[RxNewThreadScheduler-6,5,main]

```


## 结论

### 说明
这里约定一下描述的规则，我们接下来讲的远近，上下指的是代码物理位置上。
响应式编程有个消息的概念，这里消息的产生是从下往上的，当调用了subscribe 的时候，就会产生，接着往上，我们可以通过代码和log可以看出，依次执行了 doOnSubscribe1 -  doOnSubscribe4，最后到达create处。
对于数据流，则是从上往下的，经过每个继承 lift 产生的操作符，例如map, reduce,filter等。

### 所以

- 要想指定create所在的线程，需要在create的下方调用 subscribeOn 方法，他受他下方遇到的第一个 subscribeOn 的影响，反正也可以说subscribeOn影响的是在他上方的消息传递的线程，直到遇到下一个subscribeOn为止。假如全程没有一处调用subscribeOn，则消息的传递是在调用subscribe所在的线程。

- 要想指定 map 等lift操作符和Subscriber中的执行线程，则需要在它上方调用observeOn方法；反之observeOn影响的是他下方的lift操作符直到遇到下一个observeOn位置。假如整个代码中未指明observeOn方法，则运行在整个代码中第一个subscribeOn指定的线程，也可以理解成运行在create所在的线程。

- 至于二者均未指定，则可以推导出运行在调用subscribe所在的线程。

## 结尾

至于源码解释参见另外一篇。
