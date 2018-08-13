---
title: android2.3.1编译
permalink: android2.3.1-build
categories:
  - Android
tags:
  - android2.3.1
date: 2018-08-13 14:35:48
---

##前言

&ensp;&ensp;&ensp;&ensp;一直想系统的把android的内容重新捋一遍，今天京东优惠，66入了`《Android系统源代码情景分析》`，这书是基于android2.3.1来说明的，为了和方便真机编译与调试，入了nexus s. 好了话不多说，直接上 命令以及error的fix工作.

## android2.3.1编译

我的环境:

系统环境为`ubuntu 14.04`

jdk使用 `jdk1.6.0_45`

```bash
repo init -u https://android.googlesource.com/platform/manifest -b android-2.3.1_r1
repo sync -j4
```

**gcc downgrade:**

```bash
sudo apt-get install g++-4.4-multilib
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-4.4 20 --slave /usr/bin/g++ g++ /usr/bin/g++-4.4
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-4.8 20 --slave /usr/bin/g++ g++ /usr/bin/g++-4.8
sudo update-alternatives --config gcc
```

`去google aosp 官网下载对应的driver sh 脚本， 运行生成vendor`

```bash
$ source build/envsetup.sh
$ lunch
$ full_crespo-userdebug
$ make (这边千万别使用-j, 会引起多线程编译build error)
```

**以下是build时产生的error以及处理：**

`error:Can't locate Switch.pm in @INC`

```bash
sudo apt-get install libswitch-perl
```

`dalvik/vm/native/dalvik_system_Zygote.c:191: error: storage size of ‘rlim’ isn’t known`

Fix:

```bash
diff --git a/vm/native/dalvik_system_Zygote.c b/vm/native/dalvik_system_Zygote.c
index bcc2313..112563a 100644
--- a/vm/native/dalvik_system_Zygote.c
+++ b/vm/native/dalvik_system_Zygote.c
@@ -21,6 +21,7 @@
 #include "native/InternalNativePriv.h"
 
 #include <signal.h>
+#include <sys/resource.h>
 #include <sys/types.h>
 #include <sys/wait.h>
 #include <grp.h>
```

`/usr/include/zlib.h:34: fatal error: zconf.h: No such file or directory`

Fix:

```bash
sudo ln -s /usr/include/x86_64-linux-gnu/zconf.h /usr/include
```

*编译成功后:*

去twrp 下载对应的recovery 的 image.

```bash
fastboot flash recovery twrp.img
adb reboot bootloader
```

从bootloader 进入recovery模式，进行双清后，重新进入bootloader，使用一下命令进行刷机.

```bash
adb reboot bootloader
fastboot -w flashall
```

