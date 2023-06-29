# cronet_build

## 概述
记录编译Cronet for Android 的过程和步骤.

<!-- more -->

## 准备工具
1. install  depot_tools

```shell
## 下载
$ git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
```

2. 添加进path,或者 .bashrc/.zshrc

```shell
## 将 /path/to/depot_tools 天换成自己安装的目录即可
$ export PATH="$PATH:/path/to/depot_tools"
```

如果安装的位置是home目录下,上述命令切勿使用 *~*,使用绝对路径或者 HOME 替代.

```shell
$ export PATH="$PATH:${HOME}/depot_tools"
```

我的安装路径是  ~/ide/depot_tools

所以,我执行的命令是

```shell
$ export PATH="$PATH:${HOME}/ide/depot_tools"
```

## 下载代码

1. 找个目录,clone代码,我选择的是   ~/workspace/chromium
1. 拉取代码,因为我不想要history,如果想要history,去掉 --no-history 即可.

```shell
## 这个命令是第一次拉取代码使用
$ fetch --nohooks --no-history chromium
```

根据网速,快的话办小时,慢的话,数小时之后完成. 20G的东西，我的网速很慢，过了一夜吧，也没具体看多久，这个工具有个问题，没有进度条。。。
当命令结束之后,目录下就会出现隐藏文件.gclient 和 文件夹 src.
​
假如中间中断过，或者直接拷贝了一份已有的源码，非第一次拉取代码,可能会提示如下内容。
​
![sync warning](/img/cronet_build/sync_warning.png)

### 非初次同步,则执行同步代码的命令

```shell
$ gclient sync
```

进入漫长的等待...

![sync_proceed](/img/cronet_build/sync_proceed.png)

同步完成,自动执行 gclient runhooks 命令.

![sync_success](/img/cronet_build/sync_success.png)

## 切换到src目录下

```shell
$cd src
```

## 安装额外依赖

```shell
$ ./build/install-build-deps.sh
```

依赖比较多,安装需要点时间,约1G的空间大小.

我安装的时候,还遇到一个问题

![sync_error_font](/img/cronet_build/sync_error_font.png)

试了很多办法,还是不行,就按照提示的,跳过这个字体库的安装.
​
```shell
$ ./build/install-build-deps.sh --no-chromeos-fonts

```

## Run the hooks

```shell
$ gclient runhooks
```

## build

### 需求一: Building Cronet for development and debugging

#### 第一步: 设置out_dir,生成ninja文件

```shell
$ ./components/cronet/tools/cr_cronet.py gn --out_dir=out/Cronet
```

在linux进行编译,则自动生成Android 库,在Mac上,则会生成iOS库.

这个命令执行完成之后,会影响之前编译在out/Cronet目录中的内容.

如果 --out_dir 参数省略的话,就输出目录就会默认变成 out/Debug 和 out/Release,分别存放debug和release的输出内容.

![build_gn_ninja](/img/cronet_build/build_gn_ninja.png)

#### 第二步: Running the ninja files

```shell
$ ninja -C out/Cronet cronet_package
```

![build_success](/img/cronet_build/build_success.png)


#### 生成物解释

编译完,用作Android开发的库都在 chromium/src/out/Cronet/cronet 目录下.

![build_gn_dir](/img/cronet_build/build_gn_dir.png)
​

1. Android的jar包: 该目录下的所有jar文件,就是需要的jar包;
2. Android的动态库: libs目录下有对应的so文件;
3. 符号表: 对应的符号信息在symbols目录下,用于线上crash或其他栈信息的mapping;
4. 头文件: include目录下有对应的头文件.
5. 反混淆文件: 也在该目录下.


### 需求二: build mobile release

```shell
$ ./components/cronet/tools/cr_cronet.py gn --release
$ ninja -C out/Release cronet_package
```

![mobile_release](/img/cronet_build/mobile_release.png)


### 需求三: 其他abi
默认不指定参数的情况下,生成的是 ARMv7 32位的库,如果需要其他版本的库,可以通过添加如下参数,进行生成.


#### 方案一是,修改 [cr_cronet.py](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/components/cronet/tools/cr_cronet.py) 文件的 gn_args 变量,按照需求修改成如下的值.
​
- For ARMv8 64-bit: target_cpu="arm64"
- For x86 32-bit: target_cpu="x86"
- For x86 64-bit: target_cpu="x64"


![cr_cronet.py](/img/cronet_build/cr_cronet.py.png)


#### 方案二: 交互式,不需要修改文件

```shell
## 交互修改参数
$ gn args out/Cronet

```

会弹出输入界面,可以输入需要的参数,比如(这些参数我是参考的默认debug包的参数,只是添加了开头有的target_cpu部分)
​

```markdown
target_cpu="arm64"
target_cpu="arm"
target_cpu="x86"
target_os = "android"
enable_websockets = false
disable_file_support = true
disable_ftp_support = true
disable_brotli_filter = false
is_component_build = false
use_crash_key_stubs = true
ignore_elf32_limitations = true
use_partition_alloc = false
include_transport_security_state_preload_list = false
use_platform_icu_alternatives = true
use_errorprone_java_compiler = true
enable_reporting = true
use_hashed_jni_names = true
```
​

Tip: 其实最终的参数存在 out/Cronet/args.gn 这个文件里,也可以直接修改这个文件.

![args.gn](/img/cronet_build/args.gn.png)


执行编译操作
```shell
$ ninja -C out/Cronet cronet_package
```
​

![abi_success](img/cronet_build/abi_success.png)


生成的so文件在 src/out/Cronet/cronet/libs 下,因为我之前编译过 x86的,所以有两个.

![abi_so](/img/cronet_build/abi_so.png)



## 其他,iOS编译
曾经也编译过iOS版本,步骤差不多,按照文档来,但是当时有个问题,在此记录下.
1. 按照[iOS编译文档](https://chromium.googlesource.com/chromium/src/+/master/docs/ios/build_instructions.md) 操作执行,生成需要的文件夹;
2. 如果当时fetch的时候,参数不是 iOS,则需要确认 .gclient ,最后一行有  target_os = [ "ios" ]   ,然后再执行 gclient sync,下载iOS的依赖; [文档说明](https://chromium.googlesource.com/chromium/src/+/0e94f26e8/docs/ios_build_instructions.md)

## Ref
1. [Cronet build instructions](https://chromium.googlesource.com/chromium/src/+/HEAD/components/cronet/build_instructions.md)
