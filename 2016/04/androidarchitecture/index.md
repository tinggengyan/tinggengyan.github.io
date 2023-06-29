# androidarchitecture


## 概述
本文记录 Android 官方关于项目架构的文章.

<!-- more -->

## [Android Architecture Blueprints [beta]](https://github.com/googlesamples/android-architecture)

当我们开始明确如何组织和架构一个AndroidApp的时候，Android Framework层提供了很强大的可伸缩性。
这份自由虽然很有价值，但是同时也导致一个APP内存在如过重的类，命名体系不一样，架构导致难以测试，维护和扩展苦难等问题。


Android架构蓝图打算演示解决这类通用问题可能的方法。
在这个项目中，我们会提供一个机遇不同架构概念和不同工具的同一个项目实现。


你可以使用这些例子作为你创建自己的APP的一个参考，或者直接作为一个基础。本篇集中于代码结构，架构，测试和维护性。
然而，铭记于心的是，利用这些架构和工具，创建一个APP有很多方式，取决于你的侧重点。所以，这些架构不应该被当做是经典案例。
本篇例子中UI刻意保持了简单。

## beat版的意义
我们一直在做一些可能会影响我们所有例子的决定。所以我们会在发布正式版之前，一直保持初始化的版本号。

## 例子
所有的例子都发布在他们对应的分支上。查看对应项目上看的README 以获取详细信息。

-todo-mvp/- 基本的MVP架构
-todo-mvp-loaders/- 基于todo-mvp，使用loaders来获取数据
-todo-mvp-databinding/-基于odo-mvp，使用databinding库

###　还在进行中的
- todo-mvp-contentproviders - 基于todo-mvp-loaders, 使用Content Providers
- todo-mvp-clean - 基于todo-mvp, 使用Clean Architecture的概念
- todo-mvp-dagger - 基于todo-mvp, 使用Dagger2来进行依赖注入

此外，还有很多计划中的例子。

## 为何是一个待做的项目
这个APP的目的是能够简单快速的理解，而不是增加演示这个复杂的设计和测试方案的复杂性。
可以查看这个APP的规格。

此外，还有一个类似的JavaScript的项目框架，叫TodoMVC。

## 我应该为我的APP选择哪个例子

每个例子，都有一个选择的尺度，和比较客观的评估，你可以根据你的实际情况选择。

你可能要考虑到你APP的大小，你整个团队的经验，你预估的维护成本，考虑你是否需要为平板，多平台适配，以及自己的框架偏好。



## [TODO-MVP](https://github.com/googlesamples/android-architecture/tree/todo-mvp/)

### 总概

这个例子是其他变种版本的基础。这个例子展示MVP模式的一个简单的实现，没有参杂其他的架构框架。
这个例子，使用手动注入依赖的方式来提供本地和远端的数据。异步的任务是通过callback实现的。

![示例图](https://github.com/googlesamples/android-architecture/wiki/images/mvp.png)

注意：在MVP的上下文中，属于view是被重新重载了。
android.view.View被称作"Android View",在MVP中接受presenter发送命令的view被简单的称为"view".

### Fragments

使用fragment有两个理由：
activity和fragment进行隔离，非常适合用来实现MVP。
- activity作为一个控制器，用来创建和控制view和presenter。
- 可以充分利用fragment框架进行平板和多屏幕适配。

### 关键概念

在这个APP中有四个特点：
任务
任务详细
添加编辑任务
数据统计

每个特点有:

- 约定view和presenter的定义
- activity负责产生fragment和presenter
- fragment实现view中的接口
- presenter实现presenter定义的接口

总之，业务逻辑在presenter中，并且依赖于实现UI工作的view。
view层几乎是不包含业务逻辑的，只负责将presenter中的UI指令转换成UI表现，并且监听用户的UI操作，然后传递给presenter层。
通过接口来约定view和presenter之间的连接。


### 依赖
- 常用的Android官方support包(com.android.support.*)
- Android测试包(Espresso, AndroidJUnitRunner…)
- Mockito
- Guava (null checking)

### 特点
#### 复杂性 - 这个比较容易理解
##### frameworks/libraries/tools的框架使用

还没有

##### 概念复杂性
这个比较低，作为一个纯MVP实现。

#### 可测试性
##### Unit testing

高，presenter可以作为仓库和数据源进行单元测试。

##### UI testing

高, 注入一个假的的module，允许进行假数据进行测试。

#### Code metrics
和传统没有架构的项目相比，
这个例子简绍了额外的类和接口:presenter，仓库，接口等等，所以在MVP中无论是代码的行数还是类的数量都比较高。

#### 维护性
##### 易于修改和添加新特性
高。
##### 学习成本
低。项目特点明确，责任清晰明确。开发人员不需要了解项目中的外部依赖。

