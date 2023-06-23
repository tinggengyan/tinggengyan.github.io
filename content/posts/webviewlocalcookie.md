---
title: 记录WebView的file协议的cookie跨域问题
date: 2017-03-19 17:24:17
tags: [Webview,cookie]
categories: [Mobile,Android]
---
# why

在做 hybrid 框架的时候,发现以 file 协议打开的文件,存在 cookie 跨域的问题.以做记录.


# what 
以 file 协议打开的文件,页面内存在新的跳转,跳转数据通过 cookie 传递,发现cookie 不能正常传递.

# how
CookieManager 存在一个静态方法,可以打开 file 协议的 cookie.
```java
boolean allowFileSchemeCookies ()
```
这是个静态方法,打开之后,整个 APP 的 webview 就都打开了.


# 参考文献
* [官方文档](https://developer.android.com/reference/android/webkit/CookieManager.html#allowFileSchemeCookies()



