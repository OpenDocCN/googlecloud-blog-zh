# 在 Google Cloud 上动态调整持久存储磁盘的大小

> 原文：<https://medium.com/google-cloud/resize-your-persist-disk-on-google-cloud-on-the-fly-b3491277b718?source=collection_archive---------0----------------------->

有多少次你不得不停止你的虚拟机并连接一个新的磁盘，因为你在下载一个巨大的数据集或训练一个模型时耗尽了空间。

好吧，你不必这样做。云计算已经发展了很多年，现在动态调整磁盘大小对于用户来说是一件很简单的事情。我相信云提供商已经开发了非常先进的虚拟磁盘技术，但我将跳过这一点，因为我不是专家。

这篇文章是一个非常简短的分步指南，介绍如何在谷歌云计算引擎上这样做。它只适用于最常见的情况——整个磁盘被保留给单个分区/

## 步骤 0.1:检查剩余的可用磁盘

```
$ df -h
Filesystem Size Used Avail Use% Mounted on
udev 77G 0 77G 0% /dev
tmpfs 16G 9.1M 16G 1% /run
/dev/sda1 970G 552G 418G 57% /
tmpfs 77G 228K 77G 1% /dev/shm
tmpfs 5.0M 0 5.0M 0% /run/lock
tmpfs 77G 0 77G 0% /sys/fs/cgroup
```

以/dev/sda1 开头的行是您要寻找的行。从上面的例子中，您注意到您已经使用了一半以上作为/挂载的磁盘

如果你看到使用率超过 80%，那很可能是一个危险的信号。

## 步骤 0.2:检查分区

```
$ sudo lsblk
NAME MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
sda 8:0 0 1000G 0 disk 
└─sda1 8:1 0 1000G 0 part /
```

您注意到设备 sda 只包含一个分区 sda1，这使得事情变得简单多了。

## 步骤 1:增加磁盘大小

在您的虚拟机中。实例设置，单击磁盘(引导磁盘或附加磁盘)，然后根据需要增加容量。

## 步骤 2:扩展分区

转到您的虚拟机控制台

```
$ sudo growpart /dev/sda 1
CHANGED: partition=1 start=2048 old: size=419428319 end=419430367 new: size=2097149919
,end=2097151967
```

## 步骤 3:调整文件系统的大小

```
$ sudo resize2fs /dev/sda1
resize2fs 1.42.13 (17-May-2015)
Filesystem at /dev/sda1 is mounted on /; on-line resizing required
old_desc_blocks = 13, new_desc_blocks = 63
The filesystem on /dev/sda1 is now 262143739 (4k) blocks long.
```

## 第四步:验证

```
$ df -h
```

完成了。尽情享受吧！