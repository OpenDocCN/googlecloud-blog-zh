# 持久磁盘和复制

> 原文：<https://medium.com/google-cloud/persistent-disks-and-replication-9b9412fd9565?source=collection_archive---------1----------------------->

如果你是一个云用户，你可能已经看到了非常规的存储选择。从虚拟机访问的磁盘也是如此。关于核心基础设施的底层细节，没有太多正在进行的对话或参考。其中一个缺乏的话题是持久磁盘和复制是如何工作的。

## 磁盘即服务

永久磁盘不是连接到物理机的本地磁盘。永久磁盘是网络服务，作为网络块设备连接到您的虚拟机。当您从持久性磁盘读取或写入数据时，数据会通过网络传输。

![](img/8fdb3976baa26373b83986e40c4681bf.png)

持久磁盘使用巨像作为存储后端。

持久磁盘严重依赖谷歌的文件系统，名为*巨像*。Colossus 是一个分布式块存储系统，它满足了 Google 的大部分存储需求。持久性磁盘驱动程序会在数据离开虚拟机并在网络上传输之前，自动对虚拟机上的数据进行加密。然后，巨像保存数据。读取时，驱动程序解密传入的数据。

将磁盘作为服务在各种情况下都很有用:

*   动态调整磁盘大小变得更加容易。在不停止虚拟机的情况下，您可以增加磁盘大小(甚至是异常增加)。
*   连接和分离磁盘变得很简单。假设磁盘和虚拟机不必共享相同的生命周期或位于同一位置，则可以停止一个虚拟机并使用其磁盘启动另一个虚拟机。
*   复制等高可用性功能变得更加容易。磁盘驱动程序可以隐藏复制细节，并提供自动写入时复制。

## 磁盘延迟

用户经常想知道将磁盘作为网络服务的延迟开销。有各种各样的基准测试工具可用。下面是对一个永久磁盘中的 4 个 KiB 数据块的一些读取:

```
$ ioping -c 5 /dev/sda1
4 KiB <<< /dev/sda1 (block device 10.00 GiB): time=293.7 us (warmup)
4 KiB <<< /dev/sda1 (block device 10.00 GiB): time=330.0 us
4 KiB <<< /dev/sda1 (block device 10.00 GiB): time=278.1 us
4 KiB <<< /dev/sda1 (block device 10.00 GiB): time=307.7 us
4 KiB <<< /dev/sda1 (block device 10.00 GiB): time=310.1 us
--- /dev/sda1 (block device 10.00 GiB) ioping statistics ---
4 requests completed in 1.23 ms, 16 KiB read, 3.26 k iops, 12.7 MiB/s
generated 5 requests in 4.00 s, 20 KiB, 1 iops, 5.00 KiB/s
min/avg/max/mdev = 278.1 us / 306.5 us / 330.0 us / 18.6 us
```

GCE 还允许将[本地固态硬盘](https://cloud.google.com/compute/docs/disks/#localssds)连接到虚拟机，以防你不得不避免更长的往返行程。如果您正在运行缓存服务器或运行有中间输出的大型数据处理作业，本地 SSD 将是您的选择。与永久磁盘不同，本地固态硬盘上的数据不是永久的，每次虚拟机重新启动时都会被刷新。它只适用于优化情况。下面，您可以看到从 NVMe 固态硬盘的 4 次 KiB 读取中观察到的延迟:

```
$ ioping -c 5 /dev/nvme0n1
4 KiB <<< /dev/nvme0n1 (block device 375 GiB): time=245.3 us(warmup)
4 KiB <<< /dev/nvme0n1 (block device 375 GiB): time=252.3 us
4 KiB <<< /dev/nvme0n1 (block device 375 GiB): time=244.8 us
4 KiB <<< /dev/nvme0n1 (block device 375 GiB): time=289.5 us
4 KiB <<< /dev/nvme0n1 (block device 375 GiB): time=219.9 us
--- /dev/nvme0n1 (block device 375 GiB) ioping statistics ---
4 requests completed in 1.01 ms, 16 KiB read, 3.97 k iops, 15.5 MiB/s
generated 5 requests in 4.00 s, 20 KiB, 1 iops, 5.00 KiB/s
min/avg/max/mdev = 219.9 us / 251.6 us / 289.5 us / 25.0 us
```

## 分身术

创建新磁盘时，您可以选择在区域内复制磁盘。持久磁盘被复制以获得更高的可用性。复制发生在两个区域之间的区域内。

![](img/176b3db0036068dbafa9c484fcc9de86.png)

可以选择复制永久磁盘。

一个区域不太可能完全失效，但区域性失效更为常见。从可用性和磁盘延迟的角度来看，在区域内的不同分区进行复制是一个不错的选择。如果两个复制区域都出现故障，则被认为是区域范围的故障。

![](img/84e23cced6535806e5e12d5feb590bd7.png)

磁盘在两个区域中复制。

在复制场景中，数据在本地区域(us-west1-a)中可用，该区域是虚拟机运行所在的区域。然后，数据被复制到另一个区域(us-west1-b)中的另一个 Colossus 实例。至少有一个区域应该与虚拟机运行的区域相同。

请注意，持久磁盘复制只针对*磁盘*的高可用性。分区停机还可能影响虚拟机或其他可能导致停机的组件。

## 读/写序列

繁重的工作由用户虚拟机中的磁盘驱动程序完成。作为用户，您不必处理复制语义，并且可以以通常的方式与文件系统进行交互。底层驱动程序处理读写序列。

当复制副本落后且复制未完成时，读/写将通过受信任的复制副本进行，直到发生协调。否则，我们处于完全复制模式。

在完整复制模式下:

*   写入时，写入请求会尝试写入两个副本，并在两次写入都成功时进行确认。
*   读取时，读取请求会发送到虚拟机运行所在区域的本地副本。如果读取请求超时，将向远程复制副本发送另一个读取请求。

—

对于用户来说，持久性磁盘是网络存储设备并不总是显而易见的。它们在容量、灵活性和可靠性方面支持许多使用情形和功能，而这些是传统磁盘无法提供的。