---
title: Activity启动流程概述
tags:
  - Activity
categories:
  - Android
date: 2020-07-23 17:17:42
---
## 概述
花了点时间,debug了一下系统,跟踪了一下Activity的启动流程.画了一张图,作为综述.
分析的 `compileSdkVersion` 为 **28**.
用的是 draw.io 画的,[源文件](https://github.com/tinggengyan/tinggengyan.github.io/blob/source/source/img/activity_process/AndroidActivitySequenceDiagram.drawio).

![概述图](/img/activity_process/AndroidActivitySequenceDiagram.png)


