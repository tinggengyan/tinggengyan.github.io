# AOSP编译


## override 
最近给电脑换了块SSD，装了Ubuntu 18.04。之前的aosp也不想copy过来了，直接重新编译一份，顺带看下新的SSD带来的提效。
因为手机是 nexus 6p，aosp 最后支持到 8.1. 记录下编译需要的操作。

## 步骤

1. open jdk(https://openjdk.java.net/install/)
```shell
sudo apt install openjdk-8-jdk
```

2. repo(https://gerrit.googlesource.com/git-repo/)

- AUTO
```shell
sudo apt-get install repo
```

- MANUALLY

```shell
$ mkdir -p ~/.bin
$ PATH="${HOME}/.bin:${PATH}"
$ curl https://storage.googleapis.com/git-repo-downloads/repo > ~/.bin/repo
$ chmod a+rx ~/.bin/repo

```

3. AOSP [mirror](https://mirrors.tuna.tsinghua.edu.cn/help/AOSP/)

```shell
mkdir aosp
```

```shell

git config --global user.email "tinggengyan@gmail.com"
git config --global user.name "Tinggeng Yan"

sudo apt install python

cd aosp

## 切换指定版本分支 
repo init -u https://mirrors.tuna.tsinghua.edu.cn/git/AOSP/platform/manifest -b android-8.1.0_r52 --depth=1 --repo-url=https://mirrors.tuna.tsinghua.edu.cn/git/git-repo/ --repo-branch=stable

repo sync --current-branch

```

4. build

```shell

sudo apt-get install libx11-dev:i386 libreadline6-dev:i386 libgl1-mesa-dev g++-multilib
sudo apt-get install -y git flex bison gperf build-essential libncurses5-dev:i386
sudo apt-get install tofrodos python-markdown libxml2-utils xsltproc zlib1g-dev:i386
sudo apt-get install dpkg-dev libsdl1.2-dev libesd0-dev
sudo apt-get install git-core gnupg flex bison gperf build-essential
sudo apt-get install zip curl zlib1g-dev gcc-multilib g++-multilib
sudo apt-get install libc6-dev-i386
sudo apt-get install lib32ncurses5-dev x11proto-core-dev libx11-dev
sudo apt-get install libgl1-mesa-dev libxml2-utils xsltproc unzip m4
sudo apt-get install lib32z-dev ccache

```


```shell

export LC_ALL=C

source build/envsetup.sh && lunch

make -j 4

```


5. flash into nexus 6p

```shell
cd /data/aosp/out/target/product/angler

adb reboot bootloader

fastboot devices

##下面这条命令可选
##fastboot flashall -w
##-w 选项会清除设备上的 /data 分区；
##该选项在您第一次刷写特定设备时非常有用，但在其他情况下则没必要使用。

fastboot flash vendor vendor.img
fastboot flash boot boot.img
fastboot flash recovery recovery.img
fastboot flash system system.img
fastboot flash userdata userdata.img
fastboot flash cache cache.img

fastboot reboot

```


### emulator
需要编译对应的模拟器的镜像。

```shell
source build/envsetup.sh

lunch 2 #这里填序号aosp_arm64-eng为2

make -j 4

emulator
```



如果编译完成后关闭了终端窗口，则需要用以下方式启动模拟器

```shell

source build/envsetup.sh

lunch 2 #这里填序号aosp_arm64-eng为2

emulator

```



## other

1. 可能出现的错误
```shell
error: insufficient permissions for device: udev requires plugdev group membership
```

add group
```shell
sudo usermod -aG plugdev $LOGNAME
```

[ref](https://developer.android.com/studio/run/device)

## ref

[sony developer](https://developer.sony.com/develop/open-devices/guides/aosp-build-instructions/build-aosp-nougat-8-1-oreo-4-4/#tutorial-step-2)
[version branch AOSP tags](https://source.android.com/setup/start/build-numbers#source-code-tags-and-builds)

