title: 初探NDK(简单demo)
date: 2016-07-11 22:51:20
tags: [ndk]
categories: [Mobile,Android]
---

# 初探NDK

本文是 eclipse 版本.

<!-- more -->

## 操作步骤

1. 下载NDK;
![ndk下载](http://o9mhbhxlj.bkt.clouddn.com/ndkdownloadndk.png)

2. 解压到自定义的目录下。
![解压文件](http://o9mhbhxlj.bkt.clouddn.com/ndknewjnifloder.png)

3. 配置环境变量，因为需要ndk-build这个命令来构建。
4. 定义一个native方法
![定义native方法](http://o9mhbhxlj.bkt.clouddn.com/ndknativempthod.png)
5. 新建jni文件夹
![新建文件夹](http://o9mhbhxlj.bkt.clouddn.com/ndknewjnifloder.png)
6. 生成jni头文件。
命令行下切换到项目的根目录，执行javah命令。
![javah命令](http://o9mhbhxlj.bkt.clouddn.com/ndkjavah.png)
命令的文本是：E:\WorkSpace\eclipse-android\MyNDKDemo>javah -classpath bin/classes;**D:\IDE\AndroidSdk\platforms\android-22\android.jar;D:\IDE\AndroidSdk\extras\android\support\
v7\appcompat\libs\android-support-v4.jar;D:\IDE\AndroidSdk\extras\android\suppor
t\v7\appcompat\libs\android-support-v7-appcompat.jar** -d jni `com.example.myndkdem
o.MainActivity`
注：加粗部分为sdk中的Android包，按自己的情况指定即可。阴影部分包名加类名。

![生成头文件](http://o9mhbhxlj.bkt.clouddn.com/ndkautohfile.png)
执行成功之后，会生成头文件。
从NDK的sample中任意一个项目jni目录下拷贝一个android.mk文件到自己项目的jni中。
![ndk下载](http://o9mhbhxlj.bkt.clouddn.com/ndkandroidmk.png)


7. 编写C文件
![编写C](http://o9mhbhxlj.bkt.clouddn.com/ndkcfile.png)


在C文件中实现头文件的函数。
![实现头文件](http://o9mhbhxlj.bkt.clouddn.com/ndkincludeheadfile.png)

先将头文件include。
8. 在java文件中调用C函数。
![java调用C函数](http://o9mhbhxlj.bkt.clouddn.com/ndkincludeheadfile.png)

需要将加载lib的方法放在static代码块中，library的名字就是在android.mk文件中指定的名字。
9. 使用ndk-build命令编译
![build命令](http://o9mhbhxlj.bkt.clouddn.com/ndkbuild.png)
![自动生成so文件](http://o9mhbhxlj.bkt.clouddn.com/ndkgenerateso.png)

成功后会在libs下面生成对应的so文件。
操作顺序就是这样。


## 配置eclipse
 因为以上很多的操作都需要命令，所以可以再eclipse中进行配置，省得每次都要执行命令行。
 Run->external tools-> external tools configuration

![ndk配置头文件生成命令](http://o9mhbhxlj.bkt.clouddn.com/ndkeclipse1.png)

![ndk配置javah命令](http://o9mhbhxlj.bkt.clouddn.com/ndkeclipse2.png)

添加C++的代码提示
右击给project add native library,然后右击项目的属性，在C++选项出添加NDK目录下，android-ndk-r9d\platforms\android-19\arch-arm\usr\include,将include包含进去。
![ndk配置C代码提示](http://o9mhbhxlj.bkt.clouddn.com/ndkeclipse3.png)


