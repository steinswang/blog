---
title: 'Build LineageOS for Nexus 5'
permalink: 'Build-LineageOS-for-Nexus-5'
date: 2018-01-05 12:05:21
categories:
- Android
tags:
- android
- cm
- lineageos
---

## dependences
To build LineageOS, you’ll need:

For Ubuntu 14.04:
`
bc bison build-essential ccache curl flex g++-multilib gcc-multilib git gnupg gperf imagemagick lib32ncurses5-dev lib32readline-dev lib32z1-dev libesd0-dev liblz4-tool libncurses5-dev libsdl1.2-dev libssl-dev libwxgtk2.8-dev libxml2 libxml2-utils lzop pngcrush rsync schedtool squashfs-tools xsltproc zip zlib1g-dev
`

For Ubuntu 16.04 (xenial), substitute:
`
libwxgtk3.0-dev → libwxgtk2.8-dev
`
**Java**
Different versions of LineageOS require different JDK (Java Development Kit) versions.

LineageOS 14.1: OpenJDK 1.8 (install openjdk-8-jdk)
LineageOS 11.0-13.0: OpenJDK 1.7 (install openjdk-7-jdk)

**Enter the following to download the repo binary and make it executable (runnable):**
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
chmod a+x ~/bin/repo


## Get source code
```bash
$ repo init -u https://github.com/LineageOS/android.git -b cm-14.1
$ repo sync
```

**Prepare the device-specific code**
```bash
$ source build/envsetup.sh
$ breakfast hammerhead
```



**Turn on caching to speed up build**

Make use of ccache if you want to speed up subsequent builds by running:
```bash
$ export USE_CCACHE=1
```
and adding that line to your ~/.bashrc file. Then, specify the maximum amount of disk space you want ccache to use by typing this:
```bash
$ ccache -M 50G
```

## 添加私有库
vim .repo/local_manifests/lge.xml

```
<!-- nexus5 -->
<?xml version="1.0" encoding="UTF-8"?>
<manifest>
  <project name="TheMuppets/proprietary_vendor_lge.git" path="vendor/lge" remote="github" />
</manifest>
```

```
<!-- nexus6p -->
<?xml version="1.0" encoding="UTF-8"?>
<manifest>
  <project name="TheMuppets/proprietary_vendor_huawei.git" path="vendor/huawei" remote="github" />
</manifest>
```
之后再 repo sync 一次。

## 编译
```bash
export WITH_SU=true //打开root
brunch hammerhead
```

## 刷机
```bash
adb reboot bootloader
source build/envsetup.sh

#fastboot oem ramdump enable //如果bootloader download 模式未打开
breakfast hammerhead
fastboot -w flashall
```

PS:
关于科学上网:
1.将repo文件 https://android.googlesource.com/ 全部使用 https://aosp.tuna.tsinghua.edu.cn/ 代替。
2.替换已有的 AOSP 源代码的 remote，将 .repo/manifest.xml 把其中的 aosp 这个 remote 的 fetch 从 https://android.googlesource.com 改为 https://aosp.tuna.tsinghua.edu.cn/。
```
<manifest>
   <remote  name="aosp"
-           fetch="https://android.googlesource.com"
+           fetch="https://aosp.tuna.tsinghua.edu.cn"
            review="android-review.googlesource.com" />
   <remote  name="github"
```
同时，修改 .repo/manifests.git/config，将
``
url = https://android.googlesource.com/platform/manifest
``
更改为
``
url = https://aosp.tuna.tsinghua.edu.cn/platform/manifest
``
