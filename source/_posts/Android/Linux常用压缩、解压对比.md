---
title: Linux常用压缩、解压对比
permalink: zip-unzip-compare
categories:
  - 工具
tags:
  - 压缩
date: 2016-01-15 22:54:34
---

先上结论:

`结论`：
是空间与时间的选择。

压缩比率：
tar.bz2>tgz>tar


占用空间：
tar.bz2<tgz<tar

打包耗时:
tar.bz2>tgz>tar

解压耗时:
tar.bz2>tar>tgz

Linux下对于占用空间与耗费时间的折衷多选用`tgz`格式，不仅压缩率较高，而且打包、解压的时间都较为快速，是较为理想的选择。

所以我选择`tgz`.

常用的格式有：
tar, tar.gz(tgz), tar.bz2,

不同方式，压缩和解压方式所耗CPU时间和压缩比率也差异也比较大。

## tar
只是打包动作，相当于归档处理，不做压缩；解压也一样，只是把归档文件释放出来。

(1)打包归档格式：
```bash
tar -cvf examples.tar files|dir
```

`说明：`
```xml
-c, --create  create a new archive 创建一个归档文件
-v, --verbose verbosely list files processed 显示创建归档文件的进程
-f, --file=ARCHIVE use archive file or device ARCHIVE  后面要立刻接被处理的档案名,比如--file=examples.tar
```

`举例：`
```bash
tar -cvf file.tar file1       #file1文件
tar -cvf file.tar file1 file2 #file1，file2文件
tar -cvf file.tar dir         #dir目录
```

(2)释放解压格式：
```bash
tar -xvf examples.tar （解压至当前目录下）
tar -xvf examples.tar  -C /path (/path 解压至其它路径)
```

`说明：`
```xml
-x, --extract, extract files from an archive 从一个归档文件中提取文件
```

`举例：`
```bash
tar -xvf file.tar
tar -xvf file.tar -C /temp  #解压到temp目录下
```

## tar.gz tgz

`tar.gz和tgz只是两种不同的书写方式，后者是一种简化书写，等同处理`

这种格式是Linux下使用非常普遍的一种压缩方式，
兼顾了压缩时间（耗费CPU）和压缩空间（压缩比率）
其实这是对tar包进行gzip算法的压缩

(1)打包压缩格式：
```bash
tar -zcvf examples.tgz examples (examples当前执行路径下的目录)
```

`说明：`
```xml
-z, --gzip filter the archive through gzip 通过gzip压缩的形式对文件进行归档
```

`举例：`
```bash
tar -zcvf file.tgz dir #dir目录
```

(2)释放解压格式：
```bash
tar -zxvf examples.tar （解压至当前执行目录下）
tar -zxvf examples.tar  -C /path (/path 解压至其它路径)
```

`举例：`
```xml
tar -zcvf file.tgz
tar -zcvf file.tgz -C /temp
```

## tar.bz

Linux下压缩比率较tgz大，即压缩后占用更小的空间，使得压缩包看起来更小。
但同时在压缩，解压的过程却是非常耗费CPU时间。

(1)打包压缩格式：

```bash
tar -jcvf examples.tar.bz2 examples   (examples为当前执行路径下的目录)
```

`说明：`
```xml
-j, --bzip2 filter the archive through bzip2 通过bzip2压缩的形式对文件进行归档
```

`举例：`
```xml
tar -jcvf file.tar.bz2 dir #dir目录
```

(2)释放解压：
```bash
tar -jxvf examples.tar.bz2 （解压至当前执行目录下）
tar -jxvf examples.tar.bz2  -C /path (/path 解压至其它路径)
```

`举例：`
```xml
tar -jxvf file.tar.bz2
tar -jxvf file.tar.bz2 -C /temp
 ```

## gz
压缩：
```bash
gzip -d examples.gz examples
```

解压：
```bash
gunzip examples.gz
```

## zip
zip 格式是开放且免费的，所以广泛使用在 Windows、Linux、MacOS 平台，要说 zip 有什么缺点的话，就是它的压缩率并不是很高，不如 rar及 tar.gz 等格式。
压缩：
```bash
zip -r examples.zip examples (examples为目录)
```
解压：
```bash
zip examples.zip
```
## .rar
压缩：
```bash
rar -a examples.rar examples
```
解压：
```bash
rar -x examples.rar
```
 

压缩比率，占用时间对比

为了保证能够让压缩比率较为明显，需选取一个内容较多、占用空间较大的目录作为本次实验的测试。
找了一个大概有23G的目录来测试,首先要明确由于执行环境的变化，误差在所难免

首先明确一个概念：
压缩比率=原内容大小/压缩后大小，压缩比率越大，则表明压缩后占用空间的压缩包越小

![](https://user-images.githubusercontent.com/35097187/44308296-60b34600-a3e5-11e8-9dd0-35f605880467.png)

转自:https://www.cnblogs.com/joshua317/p/6170839.html
