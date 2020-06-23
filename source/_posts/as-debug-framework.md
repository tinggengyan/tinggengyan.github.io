---
title: AndroidStudio调试framework源码
tags:
  - Android
  - framework
categories:
  - Tool
date: 2020-06-23 10:42:04
---
# 概述
debug 是学习流程最快的方式,也是验证想法最好的方法.记录 Androidstudio 如何debug Android framework的代码.
<!-- more -->
## 使用无 AOSP 的代码(Java层)
这是最简单方便的方式了.

1. 下载某个版本的 Android Source code

![45b1b25e.png](/img/as_debug_framework/45b1b25e.png)

确认 *Source code* 正确下载了.

![c974a437.png](/img/as_debug_framework/c974a437.png)

2. 新建项目,所用的 `compile SDK ` 版本为需要调试的代码版本

```groovy
android {
    // 设置成需要需要分析的,且已下载源码的版本
    compileSdkVersion 29
    ......
}

```

3. 新建并启动对应版本的模拟器.

![d0be67cc.png](/img/as_debug_framework/d0be67cc.png)

4. 打断点;

这里以系统的 **ActivityManagerService** 为例.
因为**ActivityManagerService** 并未导出到**Android.jar**,所以无法直接搜索定位到 **.java**文件,所以采用双击**shift**的方式,检索文件.
![ee514167.png](/img/as_debug_framework/ee514167.png)


5. attach 到对应的进程,运行,查看断点.

**ActivityManagerService**  这个类是在系统 **system_process** 进程中的,所以,需要对**system_process** 进程进行 **attach** 操作.
![9e039bde.png](/img/as_debug_framework/9e039bde.png)


## 小结

至此,经过如上操作,就可以对某个类进行debug操作了.对于分析framework代码也是方便的很.


# 使用 AOSP 的源码进行调试
上述的方法基本能满足常见的debug需求了.但是有个前提是,debug的设备基本只能是模拟器或者装了官方release镜像的亲儿子.
对于有修改ROM需求的情况下,debug 则需要导入 aosp 中framework 的代码. 对应的运行设备得是运行了自定义ROM的设备.

