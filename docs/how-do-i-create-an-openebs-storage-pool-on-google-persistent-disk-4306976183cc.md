# 如何在 Google 持久磁盘上创建 OpenEBS 存储池

> 原文：<https://medium.com/google-cloud/how-do-i-create-an-openebs-storage-pool-on-google-persistent-disk-4306976183cc?source=collection_archive---------2----------------------->

本文属于 Kubernetes 和 OpenEBS 上的#HowDoI 系列。

OpenEBS 卷副本是 OpenEBS iSCSI 目标的实际后端存储单元，当前将数据存储在 Kubernetes 节点上的 hostPath 中。默认情况下，在根文件系统的父目录(/var/openebs)中创建一个名为 volume (PV)的文件夹，并在复制副本 pod 实例化期间绑定到容器中。这个父目录(如果还不可用，也创建)基本上是保存各个卷的持久路径，被称为 ***存储池*** 。

![](img/12d3f1321c99605320803c611294113d.png)

> 注意:上述存储池的概念是特定于当前的默认存储引擎，即 Jiva。未来版本可能会推出更多存储引擎，它们可以使用数据块设备而不是主机目录来创建存储池

出于各种原因，可能需要在安装到 Kubernetes 节点上特定位置的外部磁盘(GPD、EBS、SAN)上创建这个存储池。这是由 ***OpenEBS 存储池策略*** *，*促成的，该策略将存储池定义为一个 ***Kubernetes 自定义资源*** ，并将持久路径作为一个属性。

这个博客将关注在 Google 持久磁盘(GPD)上创建 OpenEBS PV 的步骤。

**先决条件**

*   安装了 OpenEBS 操作器的 3 节点 GKE 集群(参见:[https://docs.openebs.io/docs/cloudsolutions.html](https://docs.openebs.io/docs/cloudsolutions.html))
*   3-Google 持久磁盘，每个集群节点一个。这可以使用***g cloud compute disks create****&****g cloud compute 实例 attach-disk*** 命令来完成(请参考控制台步骤:[https://cloud . Google . com/compute/docs/disks/add-persistent-disk # create _ disk](https://cloud.google.com/compute/docs/disks/add-persistent-disk#create_disk))

**步骤 1:将 GPDs &挂载格式化到所需路径**

在每个节点上，执行以下操作:

*   切换到根用户 *sudo su -*
*   识别 GPD 连接的 *fdisk -l*

```
root@gke-oebs-staging-default-pool-7cc7e313-0xs4:~# fdisk -l
Disk /dev/sda: 100 GiB, 107374182400 bytes, 209715200 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: dos
Disk identifier: 0x635eaac1Device Boot Start End Sectors Size Id Type
/dev/sda1 * 2048 209715166 209713119 100G 83 Linux Disk /dev/sdb: 10 GiB, 10737418240 bytes, 20971520 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
```

*   用 ext 4 fs(*mkfs . ext 4/dev/SD<>)*格式化磁盘

```
root@gke-oebs-staging-default-pool-7cc7e313-0xs4:~# mkfs.ext4 /dev/sdb 
mke2fs 1.42.13 (17-May-2015)
/dev/sdb contains a ext4 file system
 last mounted on /openebs on Fri Apr 13 05:03:42 2018
Proceed anyway? (y,n) y
Discarding device blocks: done 
Creating filesystem with 2621440 4k blocks and 655360 inodes
Filesystem UUID: 87d36681-d5f3-4169-b7fc-1f2f95bd527e
Superblock backups stored on blocks: 
 32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632Allocating group tables: done 
Writing inode tables: done 
Creating journal (32768 blocks): done
Writing superblocks and filesystem accounting information: done
```

*   将磁盘挂载到所需的挂载点(*mount-o sync/dev/SD</mnt/open EBS*)

```
root@gke-oebs-staging-default-pool-7cc7e313-0xs4:~# mount -o sync /dev/sdb /mnt/openebs/root@gke-oebs-staging-default-pool-7cc7e313-0xs4:~# mount | grep openebs 
/dev/sdb on /mnt/openebs type ext4 (rw,relatime,sync,data=ordered)
```

**步骤 2:创建存储池自定义资源**

*   构建如下所示的存储池资源规范并应用它(注意，存储池的自定义资源定义已经作为操作员安装的一部分应用)

**步骤 3:在自定义存储类中引用存储池**

**步骤 4:在应用程序的 PVC 规范中使用自定义存储类**

**步骤 5:确认在存储池**上创建了卷

*   一旦创建了 open EBS PV(*ku ectl get PV，ku ectl get pods*，列出存储池自定义资源中提到的自定义持久路径的内容。它应该包含一个文件夹，其 PV 名称由稀疏文件(磁盘映像文件)组成

**逮到你了！！**

*问题*:如果 a)集群调整大小(向下/向上)，b)升级& c)虚拟机暂停，GPD 将被分离

*   在群集创建期间，没有添加“附加磁盘”的选项
*   实例模板是“不可变的”，磁盘必须单独添加到实例中

*解决方法*:在上述情况下执行手动重新连接(扩大根磁盘是一个选项，但通常不建议使用)

*原载于 2018 年 4 月 13 日*[*medium.com*](/@karthik.s_5236/how-do-i-create-an-openebs-storage-pool-on-google-persistent-disk-66089d9abb81)*。*