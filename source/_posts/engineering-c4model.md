---
title: 工程能力之C4模型
date: 2019-04-13 16:08:36
tags: [Engineering,C4]
categories: [Engineering]
---
# 概述

刚在InfoQ上看到一篇介绍C4Model的文章,觉得这个模型设计的很赞,很有指导意义,做个简单的记录.

<!-- more -->

# Why,为什么需要架构图?
`ThoughtWorks中国` 文章中有几句话我觉得很有道理,这里直接摘抄.
> “纸上的不是架构，每个人脑子里的才是” ; “那些精妙的方案之所以落不了地，是因为没有在设计上兼容人类的愚蠢”。

我觉得,软件工程,或者软件中的术语发明的原因就是为了减少沟通的障碍,让大家在一个 **平台** 上对话.

而架构图可以起到如下作用;
1. 一方面: 让软件的开发人员自己,以及和软件开发相关的用户,PM等人员都能快速了解一个系统的业务模型;
2. 另一方面: 利于开发人员相互之间协作,定下方案,因为自然语言是有模糊地带的,难以无歧义的传达;
3. 利于软件系统的维护,一图胜千言.

# What,C4 是什么呢?
详细的讲解,可以参考InfoQ的文章,这里做个总结.
> *C4* 4个单词的首字母为C的单词的代表, 分别为: 上下文(Context),容器(Container),组件(Component)和代码(Code);

依据不同的受众,分别抽象出了这四个级别.其中容器（应用程序、数据存储、微服务等,组件和代码来描述一个软件系统的静态结构.

## 第 1 层：系统上下文

显示了正在构建的软件系统，以及构建的系统与用户及其他软件系统之间的关系。
这个层级的图,关注的是用户层面看到的关系,注重的是和准备开发的系统与外部系统和交互人之间的关系.

*将用户,你的代建系统,已有的其他系统用不同的颜色进行区分;*

## 第 2 层：容器

将软件系统放大，显示组成该软件系统的容器（应用程序、数据存储、微服务等）。

在这个层级,已经关注系统本身了,开始关注这个系统有哪些部分组成,不过粒度非常粗.

## 第 3 层：组件

将单个容器放大，以显示其中的组件。这些组件映射到代码库中的真实抽象（例如一组代码）。

在这个层级,关注的已经是系统中的模块具体的功能了,这部分可能对应了具体的功能模块.

## 第 4 层：代码

如若必要,可以放大个别组件，以显示该组件的实现方式。 一般以UML图的形式展示;

这个层级,是具体的开发人员关注的实现细节了,用于具体的功能逻辑的分析和展示.

# How,怎能画图呢?
在C4官网,下有个**Tooling**节点,讲述了目前已有的几个画图工具.

# 参考
1. [用于软件架构的C4模型](https://www.infoq.cn/article/C4-architecture-model)
2. [可视化架构设计——C4介绍](https://zhuanlan.zhihu.com/p/55185723)
3. [C4官网](https://c4model.com/)

