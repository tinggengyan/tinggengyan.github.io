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

## 生成 android.ipr 文件


```shell
// 1. 编译 idegen模块
mmm development/tools/idegen/

// 2. 生成
./development/tools/idegen.sh

```

这个文件就代表了AS里的一个project.


## 修改 android.iml 文件

同时还会生成一个 iml文件,代表了project的配置情况,可以用于配置加载哪些配置.
AOSP巨大,可以只加载需要关注的模块,如 framework 和 Package 部分.
所以需要修改 android.iml 文件,将不需要的文件进行exclude.


## AS 打开 ipr 文件

### 可能遇到问题

导入可能遇到问题 `External file changes sync may be slow: The current inotify(7) watch limit is too low.`

```shell
sudo vim fs.inotify.max_user_watches = 524288

sudo sysctl -p --system

```
重启即可;


## 代码索引跳转
为了跳转到aosp的Java文件,而不是android.jar的class文件,需要调整 project struct.

1. 新建一个 jdk,此处为 jdk_none, 删除所有的path;

![create_jdk.png](/img/as_debug_framework/create_jdk.png)

2. 新建一个 android sdk,依赖 jdk_none;
![create_sdk_with_jdk.png](/img/as_debug_framework/create_sdk_with_jdk.png)

3. project 依赖的sdk切换成第2步新建的SDK即可;


## 让模拟器使用自定义的ROM
1. source ./build/envsetup.sh
2. lunch ,选择对应的 模拟器需要的API
3. emulator

> 自己编译编译出的ROM位置" ....../aosp/out/target/product/generic_x86_64 

### 附,我还没试过
```shell
$ emulator -avd Nexus5-API22 -verbose -no-boot-anim -system (the path of system.img)
```

# 方案对比
1. 第一种自然是方便的,要求比较低,对机器的性能要求也不高.有个劣势: 对于AIDL编译生成的Java文件,无法进行索引和导航.但是,可以借助官方的代码搜索网站进行弥补,搜索网站可以索引soong编译期间生成的Java代码: https://cs.android.com/android/platform/superproject 
2. AOSP的方式是灵活性更大,中间代码索引也方便. 就是性能要求比较高.


## Ref
1. [导入AOSP](https://www.jianshu.com/p/a19dcb06cd53)



