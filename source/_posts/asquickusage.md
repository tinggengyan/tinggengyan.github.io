title: AS的使用技巧
date: 2016-04-01 23:34:11
tags: [AndroidStudio]
categories: [Tool,IDE]
---
# 概述
本文记录如何使用 AS 的模板功能,加快开发.

<!-- more -->

# [自定义一个代码模板，Live Templates in Android Studio](https://www.youtube.com/watch?v=4rI4tTd7-J8&list=TL-ecRgcxcKDAyMzAzMjAxNg&index=2)

对于一些特别容易，又固定的代码块，可以设置成一个代码模块，这样不需要每次自己手动的去敲。
节约大量的时间。

例如下面这块代码：

```java
 SharedPreferences sharedPreferences = getPreferences(Context.MODE_PRIVATE);
 SharedPreferences.Editor editor = sharedPreferences.edit();
 editor.putBoolean(getString(R.string.text_welcome), true);
 editor.apply();
```
上面这段代码中，除了字符串key和后面对应的value是true，其他都是模板代码。参考本目录下的LiveTemplates1-LiveTemplates3截图。

![第一步](https://github.com/tinggengyan/Study/blob/master/notes/AS%E5%BF%AB%E6%8D%B7%E9%94%AE/LiveTemplates1.png)
![第二步](https://github.com/tinggengyan/Study/blob/master/notes%2FAS%E5%BF%AB%E6%8D%B7%E9%94%AE%2FLive%20Templates2.png)
![第三步](https://github.com/tinggengyan/Study/blob/master/notes%2FAS%E5%BF%AB%E6%8D%B7%E9%94%AE%2FLive%20Templates3.png)
