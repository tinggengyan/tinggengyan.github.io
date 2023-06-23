---
title: JsBridge分析
date: 2016-10-13 22:38:20
tags: [JsBridge]
categories: [Mobile,Android]
---

# 序 

本篇分析大头鬼的 [JsBridge](https://github.com/lzyzsd/JsBridge) 库.

<!-- more -->

## js bridge 分析大头鬼的 JS 库

主要的任务都交给了 JS 库去执行，整个框架的主要过程分为三个过程，native 通过发送消息调用 JS 方法，在 JS 层的消息处理,将消息处理完成之后返回。

### Java调用JS 方法

对于 native 想要调用 JS 代码的时候，会调用自定义的 BridgeWebView 的 send 方法进行发送消息，对于消息，我们可以自定义处理器和回调处理方法，发出去的消息不是立即处理的，而是将其进行封装，成 Message 对象，如果该消息后续有回调，需要记录回调等待后续的处理，而后添加到队列中。进行排队，等待处理。

### Java 消息的分发

这个消息的分发是指的 native 层面的消息分发，当页面加载 finish 之后，会调用 dispatch 方法对刚刚进入队列中的 Message 对象进行分发。解析出 Message 携带的内容，按照约定，生成对应的 JS 脚本，交由 JS 库处理。

### 加载 JS 库

按照流程的事件顺序上讲，这个过程应该是第一个被执行的，在加载完 HTML 页面的同时需要加载 JS 库，但是 JS 库的加载和页面的加载谁先成功，这个可能出现先后差异，所以需要在 HTML 中判断 JS 库是否加载成功，如果加载成功则进行消息处理；如果加载还未成功，则监听加载 JS 库的事件，等到加载成功之后，再进行消息的处理。这些事件的处理需要在 HTML 中处理.


### JS 库对消息的处理

这里处理的其实是一个 URL ，在 JS 库层面会调用相应的 JS 方法。当 Java 层调用了消息的分发命令，会通过执行 JS 脚本的方式将消息交给 JS 层，JS 层面会将需要处理的消息，添加到队列中，等待处理，这个过程又有点像 Java 层的处理方式，_handleMessageFromNative ，会将消息存入 receiveMessageQueue 中，等待处理。

当 JS 库加载成功，需要 HTML 层面可以主动的调用JS层的消息处理的方法 _dispatchMessageFromNative，遍历 receiveMessageQueue，对消息进行处理。

对于消息的业务处理，都在 JS 层面完成，JS 处理完成之后，如果有需要进行为 Java 层提供返回值的，则进行重定向，通知 Java 层返回值队列中有数据，Java 层拦截 URL 进行主动拉去数据的 JS 脚本执行，JS 层方法将返回值放入 URL 中，进行重定向，Java 层会拦截 url，根据一定的规则，从 url 中截取出数据，数据中携带了唯一的请求 ID，根据这个 ID 查找出对应的回调方法。


### JS 向 native 发送消息

当 JS 想主动向 native 发送消息的时候，会主动调用 JS 库中的 send 方法，将消息放入 JS 中的 sendMessageQueue 队列中。然后像给 native 层提供返回值一样，进行 url 的重定向,然后依旧是 Java 层拦截 URL,然后主动去拉取数据。


### native 处理 JS 发来的消息

当 JS 向 native 发出消息,native 收到消息后，解析成 Message 对象，根据 callid 去判断是否有特定的 handler，是否有回调,这些 handler 和 回调都是刚刚封装 Message 对象时保存的。


## 怎么使用

对于一般的具体应用，都是 JS 层调用 Java 层提供的方法。这时候，对于平时普通的常规调用，都是需要提前约定的，可以定义在 DefaultHandler 中。也可以根据不同的业务需要定义不同的 BridgeHandler。匿名实现 BridgeHandler 的方式只适合 java 层调用 JS 方法，这种方式不方便维护。



## 总结

native 层提供一些基础的方法，类似于分享操作，登录等功能。具体的业务类执行还是交由 JS 层执行，需要 native 协助的功能，以重定向 URL 的方式进行。JS 不直接调用 Java 方法，也就是没有 @JavascriptInterface 注解的方法，所有的约定,包括返回的数据,都在 URL 中。
这里会有一个缺点，虽然 http1.1 中声明 url 是没有长度的限制的，但是一般而言，服务端或者浏览器都有有长度的限制，所以，将所有的数据都通过 URL 来传递还是有不妥之处。


