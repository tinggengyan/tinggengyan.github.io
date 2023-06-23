# AndroidStudio引入AAR依赖

# 概述

本文介绍 AndroidStudio 项目如何如何进行 aar 包依赖.

<!-- more -->

# AndroidStudio如何引用aar依赖

## 未成功方案

google 了一圈，网上的方法基本都是以下这种，在 *module/build.gradle* 文件中添加如下代码.同时将 aar 文件 copy 到 libs 文件夹下.

```groovy

repositories {
    flatDir { dirs 'libs' }
}
compile(name:'aarName', ext:'aar')

```

我尝试了很多次,没有成功.

## 亲测有效方案

采用了以下方法成功了,和上面的内容一致,只是位置不一样.

1. **project** 目录下新建一个目录 aars(名字应该随意),新建的 aars 文件夹是用来存放需要 aar 包的.

2. 在 **project** 下的 build.gradle 中添加代码.

```groovy
allprojects {
    repositories {
        jcenter()
        //为了添加aar依赖
        flatDir {
            dirs '../aars'
        }
    }
}
```
注意: 是在根目录下的 build.gradle 文件中修改,添加的节点是在 **allprojects** 的 **repositories** 下.

3. 在需要引用的地方添加引用,格式如下.

```groovy

compile(name:'aarNameWithoutExtention', ext:'aar')

```
添加依赖,依赖的格式是 aar 包文件的名字(不带后缀),ext 注明后缀即可.


采取如上步骤之后,即可成功添加依赖.



