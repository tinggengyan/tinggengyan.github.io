---
title: 加快 AndroidStudio 编译速度之 build cache
date: 2017-02-07 23:04:06
tags: [AndroidStudio]
categories: [Tool,Gradle]
---
## Why
AndroidStudio 编译速度慢,已经是人神共愤的事情了.本文是一篇译文，讲述如果利用 build cache 技术加快编译速度。分成两部分,一部分是第三方博文,另外一部分是官方文档.援引文章在结尾给出.

<!-- more -->

## Using build cache in Android Studio makes Gradle build faster
###  为何关心 build cache?
因为 build cache 可以加快 clean 和 build 的速度。当你执行 'gradle clean build' 或者类似的命令的时候。

### How does it make the build faster?

通过缓存已经分包的 libraries，这个过程是不在 Gradle 的缓存管理范围内的。无论是通过 Android studio 或者 命令行的方式执行 clean 操作，build-cache 内的包都会被保留，等到下次 build apk 的时候，被复用。可以在 build-cache 目录下查看缓存的结构。

![缓存文件夹](https://zeroturnaround.com/wp-content/uploads/2016/12/android-studio-android-build-cache-dir.png)

这是文件夹下列出的是一系列命名比较奇怪的文件和文件夹。文件大小是 0 字节的文件是用来锁定文件使用的。这个是非常必要的，因为同一个缓存文件可以被不同的项目使用。锁文件，可以防止两个项目同时对一个缓存文件进行读写操作。

### Exploded aar caches
aar 缓存以文件夹的形式展现。有两种类型的缓存，一种是 dex 缓存，一种是解压完的 aar 形式的缓存。解压完的 aar 将直接保存在对应的 output 文件夹下。比如 *220674f5fc7186b424e032744f0eeb413d469b54* 文件夹的  *input 文件* 包含以下内容：

```xml
COMMAND=PREPARE_LIBRARY
MAVEN_COORDINATES=com.google.maps.android:android-maps-utils:aar:0.3.4
```
文件夹的名字是 *input file* 的 *sha1sum* 值。在这个例子里，就是 *android-maps-utils* 库。解压完的 aar 在依赖的分析过程中（若未被缓存）会被缓存。

### Dexed caches
对于分包缓存，有着和 aar 缓存相似的结构。

```xml
COMMAND=PREDEX_LIBRARY
FILE_PATH=/Users/Sten/.android/build-cache/220674f5fc7186b424e032744f0eeb413d469b54/output/jars/classes.jar
FILE_HASH=cf251baf39f5c5138224b67b4106eb6331abbd13
BUILD_TOOLS_REVISION=25.0.0
JUMBO_MODE=false
OPTIMIZE=true
MULTI_DEX=false
```
文件中的 *FILE_PATH* 指向的就是我们上面所说的文件夹。文件中包含了 *android-maps-utils* 库的分包版本。*input file*  中的键值对定义了每个缓存实体。举个例子，*build tools revision* 是 25.0.0 和 25.0.1 将会有不同的分包缓存，因为 BUILD_TOOLS_REVISION 值不同。但是对于 aar 缓存而言，则会是同一个，因为对于 aar 缓存的 *input file* 而言，command 未变，maven 地址也没有变，输入文件未变。

这里的输出是一个文件，而不是一个文件夹。解压之后的文件结构如下：

```shell
73  12-06-16 16:07   META-INF/MANIFEST.MF
       0  12-06-16 16:07   META-INF/
   87000  12-06-16 16:07   classes.dex
       0  12-06-16 16:07   com/
       0  12-06-16 16:07   com/google/
       0  12-06-16 16:07   com/google/maps/
       0  12-06-16 16:07   com/google/maps/android/
       0  12-06-16 16:07   com/google/maps/android/clustering/
       0  12-06-16 16:07   com/google/maps/android/clustering/algo/
       0  12-06-16 16:07   com/google/maps/android/clustering/view/
       0  12-06-16 16:07   com/google/maps/android/geometry/
       0  12-06-16 16:07   com/google/maps/android/heatmaps/
       0  12-06-16 16:07   com/google/maps/android/projection/
       0  12-06-16 16:07   com/google/maps/android/quadtree/
       0  12-06-16 16:07   com/google/maps/android/ui/
```
如你所见，这个只是文件夹结构和 classes.dex 文件。

### Multidex and API level 21
根据 multidex  和 target API 是否高于 21 的不同组合，分包缓存的使用方式也不一样。

第一种情况是，不使用分包。在这种情况下，API 级别无论是否高于 21 都无关。将会使用分包缓存，也会进行 dex 文件的 merge 操作。在 apk 文件中，我们将会看到只有一个 classes.dex 文件，这个 classes.dex 包含了所有的 application 类和 libraries。

第二种情况是，minSdkVersion 低于 21 并且 multidex 无法使用 build cache 下的 predex libraries .这是因为兼容包里的 multidex 并不支持 predex.Gradle 插件总是将所有的 application 和 library classes 都放到一个 dex 包里.

最后一种情况是使用了 multidex 并且 API 级别高于 21.在这种情况下,build-cache文件夹下的分包文件将会被直接打包进 apk 文件中.每个库都将分别拥有一个将被打包进 apk 中的 classes.dex 文件.这也是为什么 API 21 是[编译时期最佳的选择](https://developer.android.com/studio/build/multidex.html#dev-build) .

### Performance measurements
针对2015年的 iosched app 在没有 multidex 和 API 最低版本 21 下分别进行测试.打开 Gradle 守护进程,启用和禁用 build cache,分别在命令行下运行 5 次 clean,build 操作.以下是五次运行结果的中位数报告.

![Clean build without build cache](https://zeroturnaround.com/wp-content/uploads/2016/12/android-build-profile-2.png)

![Clean build with build cache](https://zeroturnaround.com/wp-content/uploads/2016/12/android-build-profile-1.png)

从上图可以看到,编译时间很明显的从 18.7 降到了 6.5秒.从图上也可以很清晰的看到 *android:transformClassesWithDexForDebug task* 所花的时间,从 12.1 降到了 1.7 秒.节省的时间取决于项目中使用的依赖包数.

如果还没尝试 Android studio 2.3 ,建议现在尝试.你将会很明显的看到节省的时间.如果你对非正式版的没有兴趣,也可以在 Android studio 2.2 和 Android Gradle plugin 2.2 上实验,通过向项目根目录下的 gradle.properties  文件中添加 *android.enableBuildCache=true* .

<!-- more -->

------

下面是官方的译文.

##  Build Cache
### Introduction
在 *Android Studio 2.2 Beta3* 中介绍了一种可以减少编译时间的新 *build cache*  缓存特性，这个新特性可以加快包括全量编译，增量编译和 instant run 的编译时间，通过保存和复用前一次由同一个项目或者其他项目 build 产生的文件或者文件夹。
*build cache* 目的是为了在所有的 Android 项目中共用。开发者可以通过修改 *gradle.properties* 文件，实现是否启用 *build cache* 和指定缓存的位置。当前 *build cache*  只包含 *pre-dexed* 库，未来，*Android studio* 团队会支持其他类型的文件。

**注意**：*build cache*  的实现是和  *gradle cache*  管理（例如,reporting up-to-date statuses）是相互独立的。当执行一个 task 的时候，无论是否使用 *build cache*  对于 Gradle 而言都是未知的（即：即使命中了缓存，Gradle 也不会认为是 up-to-date）。然而，当使用 *build cache* 的时候，还是希望加快编译速度的。
即使目前还未发现有任何问题，我们希望给社区更多的时间以提供更多的反馈。目前这个特性仍旧作为实验性的特性，目前默认还是禁用的。（Android Studio 2.3 Canary 1 开始默认启用）。根据未来的反馈情况，当我们觉得这个特性稳定了，将会在 Android Studio 2.3 或者 2.4 中默认启动。

### How to use the Build Cache
#### Step 0
确保 [android.dexOptions.preDexLibraries](http://google.github.io/android-gradle-dsl/current/com.android.build.gradle.internal.dsl.DexOptions.html#com.android.build.gradle.internal.dsl.DexOptions:preDexLibraries)已经设置为 **true**。否则 *libraries* 不会被 *pre-dexed*，因而 *build cache* 并不会被使用。

#### Step 1
在 Android 项目中打开 *gradle.properties*，添加以下两个参数
```groovy
android.enableBuildCache=true # true:启用 build cache，反之禁用。如果这个参数未设置，默认是禁用 build cache.

android.buildCacheDir=<path-to-build-cache-directory> # 这个是个可选项，用来指定 build cache 目录的绝对路径。如果设置成项目路径，那么是项目于项目的根目录而言的。如果这个参数未被设置，那么默认的目录是 <user-home-directory>/.android/build-cache。如果使用相同的缓存目录，那么多个项目可以共用相同的缓存，所以，推荐使用默认的路径或者使用一个项目外的绝对路径。任何情况下，build cache 的路径都不应该放在 "build" 文件夹下，除非每次运行 clean 之后，都能删除 build cache 。如果 android.enableBuildCache 被设置成 false，则这个参数将会被忽略。

```

#### Step 2
build 项目，或者在命令行下执行 *./gradlew assemble*,检查以下位置，查看 build cache 是否起作用。
- 缓存的文件被存储在了上述  android.buildCacheDir 指定的文件夹下。默认情况下，是在 *<user-home-directory>/.android/build-cache.*
- 最终的 pre-dexed 文件被存储在了 *<project-dir/module-dir>/build/intermediates/pre-dexed/debug* 和 *<project-dir/module-dir>/build/intermediates/pre-dexed/release.*。可以在命令行下运行指令查看  “pre-dexed” 文件夹。如果点击的是 Android Studio 面板上的 “Run”  按钮，时无法看到这个文件夹的，因为这个文件夹背会被删除。

**注意**:
如果使用 Multi-dex 并且 minSdk >= 21 ，那么 dexed files 将会被直接保存在 *<project-dir/module-dir>/build/intermediates/transforms/dex* 目录下， 而不是在 *<project-dir/module-dir>/build/intermediates/pre-dexed*.

** Cleaning the Build Cache
如果想要清除 *build cache*， 可以直接删除 *build cache* 文件夹内的内容。
*build cache* 文件夹在 *android.buildCacheDir* 指定的目录下,或者在默认的 *<user-home-directory>/.android/build-cache* 文件夹下.

从 Android Studio 2.3 Canary 1 开始，Gradle task 中新增了一个叫做 *cleanBuildCache* 的任务，可以更加便利的删除 *build cache* 。
```shell
./gradlew cleanBuildCache
```

## 感激,非常感激，万分的感激！
感谢以下的文章以及其作者和翻译的开发者们,排名不分先后

*  [Build Cache](http://tools.android.com/tech-docs/build-cache)
*  [Using build cache in Android Studio makes Gradle build faster](https://zeroturnaround.com/rebellabs/using-build-cache-in-android-studio-makes-gradle-build-faster/)


