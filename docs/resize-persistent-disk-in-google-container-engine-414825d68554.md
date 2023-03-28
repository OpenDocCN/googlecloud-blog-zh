# 在谷歌容器引擎中调整持久磁盘的大小

> 原文：<https://medium.com/google-cloud/resize-persistent-disk-in-google-container-engine-414825d68554?source=collection_archive---------1----------------------->

目前，似乎没有任何自动化的方式来支持 GKE 持久磁盘的大小调整。换句话说，你不能简单地跑:

```
$ gcloud compute disks resize my-disk-1 --size=1TB
```

让库伯内特斯集装箱把它捡起来…

我有两种不同的情况。第一，我在 deployment.yaml 中定义了一个分区号，因为我从一个具有预定义大小和分区的映像构建了持久磁盘；第二，我没有从映像构建，只是创建了没有分区信息的持久磁盘。我们来谈谈前者是这样创造出来的:

```
$ gcloud compute images create jenkins-home-image --source-uri \
[https://storage.googleapis.com/solutions-public-assets/jenkins-cd/jenkins-home-v3.tar.gz](https://storage.googleapis.com/solutions-public-assets/jenkins-cd/jenkins-home-v3.tar.gz)$ gcloud compute disks create --size=50GB jenkins-home \ 
--image jenkins-home-image
```

这里需要注意一些事情。这张图是 10G。如果映像有一个预定义的大小和分区，就像这个一样，那么即使在创建它的时候将持久磁盘的大小设置为 50G，您仍然需要使用 fdisk 并调整大小。

```
WARNING: Some requests generated warnings:- Disk size: '50 GB' is larger than image size: '10 GB'. You might need to resize the root repartition manually if the operating system does not support automatic resizing. See https://cloud.google.com/compute/docs/disks/persistent-disks#repartitionrootpd for details.
```

下面是 yaml:

```
volumes:
 - gcePersistentDisk:
    fsType: ext4
    partition: 1
    pdName: jenkins-home
   name: jenkins-home
```

所以…我必须登录到运行容器的虚拟机实例，并实际运行 fdisk。首先删除分区，然后用相同的起始扇区号重新创建分区，然后写入分区。

您可以通过运行 lsbk 找到该设备:

```
$ lsblk |grep -1 kubernetes
sdb       8:16   0   50G  0 disk
└─sdb1    8:17   0   50G  0 part /home/kubernetes/containerized_mounter/rootfs/var/lib/kubelet/pods/c195bc3a-b055-11e7-9337-42010a8e001b/volumes/kubernetes.io~gce-pd/my-home
```

然后做你的 fdisk 如上所述的东西，再次注意开始扇区，并使用相同的，否则你会被软管..

```
fdisk /dev/sdbWelcome to fdisk (util-linux 2.28.2).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.Command (m for help): p
Disk /dev/sdb: 100 GiB, 107374182400 bytes, 209715200 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: dos
Disk identifier: 0x642e6196Device     Boot Start      End  Sectors Size Id Type
/dev/sdb1        2048 20971519 20969472  10G 83 LinuxCommand (m for help): d
Selected partition 1
Partition 1 has been deleted.Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1): 
First sector (2048-209715199, default 2048): 
Last sector, +sectors or +size{K,M,G,T,P} (2048-209715199, default 209715199):Created a new partition 1 of type 'Linux' and of size 100 GiB.Command (m for help): p
Disk /dev/sdh: 100 GiB, 107374182400 bytes, 209715200 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: dos
Disk identifier: 0x642e6196Device     Boot Start       End   Sectors  Size Id Type
/dev/sdb1        2048 209715199 209713152  100G 83 LinuxCommand (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Re-reading the partition table failed.: Device or resource busyThe kernel still uses the old table. The new table will be used at the next reboot or after you run partprobe(8) or kpartx(8).
```

然后你可以运行:

```
$ partx -u /dev/sdb
```

一旦我这样做了，我就可以在设备上运行 resize2fs 了:

```
$ resize2fs /dev/sdb1
```

在运行 fdisk 之前，resize2fs 只会让您知道您已经使用了所有的块。一旦所有这些都完成了，你应该能够从你的容器里面看到正确的尺寸。

```
$ kubectl exec jenkins-3730059860-td55q --namespace jenkins \
-c master -- df -h
Fetching cluster endpoint and auth data.
kubeconfig entry generated for iac-test-cluster.
Filesystem      Size  Used Avail Use% Mounted on
overlay          95G  3.7G   91G   4% /
tmpfs           1.9G     0  1.9G   0% /dev
tmpfs           1.9G     0  1.9G   0% /sys/fs/cgroup
/dev/sda1        95G  3.7G   91G   4% /etc/hosts
/dev/sdb1        **50G**  353M   47G   1% /var/jenkins_home
```

后一种情况非常简单。我可以简单地登录虚拟机并运行 resize2fs。集装箱很快就发现了它。这些磁盘是这样创建:

```
$ gcloud compute disks create --size=50GB my-data
```

yaml 在这里:

```
volumes:
 - gcePersistentDisk:
    fsType: ext4
    pdName: my-data
   name: my-data
```

所以希望在我们等待的时候:

[https://github.com/kubernetes/kubernetes/issues/35941](https://github.com/kubernetes/kubernetes/issues/35941)

..以上是有帮助的。