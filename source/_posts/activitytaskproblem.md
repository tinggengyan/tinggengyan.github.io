---
title: 记录一个进程重启的问题
date: 2017-02-25 21:30:21
tags: [Activity,Task]
categories: [Mobile,Android]
---

# why

在项目中，比如推送了一个具体的产品，肯定是打算显示具体的产品详情页，这时候点击返回，也自然打算让用户返回首页,还是让用户留在 APP 内。
最近项目在接入华为推送的时候，遇到一个问题,我们的默认启动 Activity 是一个广告页,在从详情页返回到主页的时候,在主页再返回,出现了广告页.

类似于如图.
![效果图](http://o9mhbhxlj.bkt.clouddn.com/task_desc.png)

<!-- more -->


# How
看到这个现象猜测,是不是当栈中有 Activity 实例的时候,进程是会自动重启.为此做了一个实验，A->B，在 B 中关闭虚拟机，这时候，虚拟机自动重启。打开了 A 页面。

A->B->C，在 C 中关闭虚拟机。

这时候,栈中的信息如下.
![正常栈信息](http://o9mhbhxlj.bkt.clouddn.com/task_abc.png)


此时在 C 中执行关闭虚拟机的操作,然后进程重启.下图是虚拟机重启之后的栈信息。发现,除了之前栈顶的 C 销往了,栈下的 B,C 都还在.并且 B 的状态信息还是重启之前的,除了 pid 不一样了.
![进程重启栈信息](http://o9mhbhxlj.bkt.clouddn.com/task_ab.png)


这时候按下返回,栈信息如下.
![重启按下返回键栈信息](http://o9mhbhxlj.bkt.clouddn.com/task_a.png)


# Conclusion

1. 为何会出现欢迎页
目前发现,华为推送在点击通知的时候,会自动启动 APP 的默认 Activity.
1. 关于为何进程会重启
目前发现的现象是,当栈底还有 Activity 的时候,就会重启. Google 了一些博客,还是未发现有合理的解释.
2. 为何重启后 task 中原有栈顶的 Activity 的信息和重启之前一模一样?
原因是什么?目前不知道怎么下手分析,仅仅是猜测,应该是和 ActivityRecord 有关,系统在重启的时候直接复用了.

