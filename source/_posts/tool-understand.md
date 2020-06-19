---
title: Mac下 Understand 的初步配置
tags:
  - Understand
  - Tool
categories:
  - Tool
date: 2020-06-19 22:57:14
---
# 概述
之前一直寻找一款类似于windows上的sourceinsight的软件,后来无意发现 Understand,感觉挺好,熟悉一下,可以用来看代码.体验不错.

<!-- more -->

## 导入流程
和sourceinsight一样,都是新建一个project,在此基础上进行代码的阅读和修改;
![e5d3267c.png](/img/efficiency_tool_understand/e5d3267c.png)
![e34dd270.png](/img/efficiency_tool_understand/e34dd270.png)
![4ddef3ef.png](/img/efficiency_tool_understand/4ddef3ef.png)
1. new project 
2. import project files

## 部分实用快捷键

1. command + F:

- 在侧边的文件栏可以按照文件名进行搜索;
- 在打开的文件内可以搜索匹配的关键词;

2. command + G:

在搜索的基础上可以查找匹配的结果的下一项;

3. command +shift + G:

在搜索的基础上可以查找匹配的结果的上一项,即反向查找;

4. command + F3

搜索选中的内容

5. command + option + p/n
返回前一个/下一个修改的地方


## 部分实用的操作

1. 绘图能力
![a520cff1.png](/img/efficiency_tool_understand/a520cff1.png)

- uml 类图
在类名上右击,`Graphical Views` -> `UML Class Diagram`
- 查看选中类调用其他类的关系图(单向的调用)
在类名上右击,`Graphical Views` -> `Cluster call`
- 查看选中类和其他之间关系图(单向和双向的调用都会列出)
在类名上右击,`Graphical Views` -> `Cluster callby Butterfly`
- 查看选中类内部的调用关系
在类名上右击,`Graphical Views` -> `Cluster callby Internal`
- 查看选中类被哪些其他的类调用(单向的被调用)
在类名上右击,`Graphical Views` -> `Cluster callby`


2. 预览能力,非常好用的功能
![8e21f7d4.png](/img/efficiency_tool_understand/8e21f7d4.png)

在类名上,右击 `View Information` -> `Reference by Flat List`: 查看类被引用的列表. 
注: 如果选中的是方法名的话,这里展示的就是方法被引用的列表了.

3. 文件搜索能力

![aed73fba.png](/img/efficiency_tool_understand/aed73fba.png)
除了直接使用搜索以外,可以在 `Entity Filter` 里进行过滤文件.

4. 收藏夹
![993c3f30.png](/img/efficiency_tool_understand/993c3f30.png)

用以将需要分析的文件分组

5. 查看调用链
![e2e50df5.png](/img/efficiency_tool_understand/e2e50df5.png)

通过选中方法右击 `explore` -> `explore called by/Calls`,可以看不到方法被谁调用,自身又调用了谁,非常非常实用.


## 缺陷
1. 文件的检索,快捷键
目前试用,有几个点比较不舒适,文件的检索能力,没有找到对应的快捷键,能迅速的搜索文件.
退而求其次,采用为`Entity Filter` 自定义快捷键的方式,来达到此目的.

![5e5424bf.png](/img/efficiency_tool_understand/5e5424bf.png)

当前我采用的是 `control + e`,目前看能满足需求.

2. 在 `information` 里搜索关键词超级慢;
3. 没有不可编辑的选项
阅读代码的时候,有时不小心误触什么键,可能导致代码变更,需要设置全部文件均不可修改. 不过这点可以通过权限控制,或者干脆不设置,小问题.
4. 没办法快捷键检索类的属性和方法
目前可以查看属性只能通过`Information`. 方法可以通过菜单栏处的`scope list` 查看.没有快捷键进行关键词搜索,只能鼠标,这点太大的缺陷.关于这点,还是IDE或者VSC 比较方便.目前这个只能通过选中文件,在光标处于选中文件编辑区的情况下, `commnand + f` 进行检索,检索的顺序还不能模糊匹配.









