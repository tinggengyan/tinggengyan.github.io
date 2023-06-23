# 

title: DataBinding 入门简介
date: 2016-02-20 22:53:36
tags: [DataBinding]
categories: [Mobile,Android]
---

# 概述
本文介绍 DataBinding 的基本概念和接入流程

<!-- more -->

# DataBinding出现的背景
作为一种 MVVM 的实现方式出现.

# 概念

将数据的 provider 和 consumer 进行绑定，而后进行二者之间同步的一种技术。实现逻辑层和表现层的绑定。

------

### 注意事项：

* 使用的编译工具必须是 gradle，并且使用的 Android gradle 插件版本依旧官方所说，不能低于 Android Plugin for Gradle 1.3.0-beta4；
* 在使用的 module 的 gradle 文件中添加 apply plugin: 'com.android.databinding'；

------
以上部分是 beta 1.3 版本的环境搭建以下是 1.5 的

----

* 使用gradle for android 1.5
* 在 APP module 的 gradle 中添加代码段 
 
 ```xml
    dataBinding {
        enabled = true
    }
```

-----


# 如何使用

## 工作的流程原理

1. 在编译的时候，dataBinding 回去布局文件中进行文件的解析，然后获取关于 dataBinding 的设置，然后为对应的 view 设置 tag，
然后删除关于 dataBinding 的所有内容。
2. 对于属性中引用 java 变量的值的地方，原理都是调用的对应的 java 的 set 方法进行设置，比如 TextView 的属性 text 对应了 setText();对于 ImageView 的 src 属性
通过一些注解，让其对应 setImageResource
 ```java
@BindingMethod(
type = android.widget.ImageView.class,
attribute = "android:src",
method = "setImageResource")
```

3. dataBinding 的 BaseObservable，继承的类，通过注解 @Bindable 注解对应属性的 get 方法可以在属性变化的时候及时的通知布局中更新 UI。

4. BindingAdapter 方法，用在 adapter 中的

```java
@BindingAdapter("android:src")
public static void setImageUrl(ImageView view, String url) {
    Picasso.with(view.getContext()).load(url).into(view);
}
```

1.额外的属性,同样是 adapter 中
 ```xml
   <ImageView …
      android:src="@{contact.largeImageUrl}"
      app:placeHolder="@{R.drawable.contact_placeholder}"/>
 ```

```java
@BindingAdapter(value = {"android:src", "placeHolder"},requireAll = false)
public static void setImageUrl(ImageView view, String url,int placeHolder) {
    RequestCreator requestCreator =Picasso.with(view.getContext()).load(url);
    if (placeHolder != 0) {
        requestCreator.placeholder(placeHolder);
    }
    requestCreator.into(view);
}
```


> * 参考官方指导：
    https://developer.android.com/tools/data-binding/guide.html
> * 同时发现，敲完代码发现这个demo写的很详细：
    https://github.com/LyndonChin/MasteringAndroidDataBinding

