# Kubernetes:一个关于自定义图像、打包器和 sysdig 的故事

> 原文：<https://medium.com/google-cloud/kubernetes-a-story-about-custom-images-packer-and-sysdig-e18ed8d7f403?source=collection_archive---------0----------------------->

我们刚刚将我们的 Kubernetes 集群从 1.5.2 更新到 1.9.1。自 1.5.2 以来，Kubernetes 发生了很多变化，但最臭名昭著的，至少对我来说，是 Kubernetes 授权从 [ABAC](https://kubernetes.io/docs/admin/authorization/abac/) (基于属性的访问控制)到 [RBAC](https://kubernetes.io/docs/admin/authorization/rbac/) (基于角色的访问控制)。然而，今天我不打算写 Kubernetes 的授权，我会留到以后再写。今天，我们要用定制的操作系统映像将 Kubernetes 部署到 GCE。

1.5.2 中默认的 Kubernetes 节点操作系统是 Debian，但它在某个时候切换到了容器优化操作系统(COS)。在解决了上面提到的授权问题和其他一些小问题之后，我让我们的集群运行在 COS 上没有任何问题，除了一件事: [sysdig](https://sysdig.com/) 。正如 GCP 文档中所描述的，sysdig 目前只在 Ubuntu 中受支持。COS 中 sysdig 的一个问题是 COS 不提供构建 sysdig 内核模块所需的内核头。如果是这种情况，sysdig 会尝试下载一个预编译的二进制文件，但是也不行。

那时我意识到我必须使用 Ubuntu 而不是 COS，所以我使用了下面的 Kubernetes 环境变量:

```
export KUBE_OS_DISTRIBUTION=ubuntu
export KUBE_GCE_MASTER_PROJECT=ubuntu-os-cloud
export KUBE_GCE_MASTER_IMAGE=ubuntu-1604-xenial-v20180109
export KUBE_GCE_NODE_PROJECT=ubuntu-os-cloud
export KUBE_GCE_NODE_IMAGE=ubuntu-1604-xenial-v20180109
```

不幸的是，这并没有走多远，Kubernetes 的部署失败了。我登录到 Kubernetes 主节点检查了一下，它告诉我:

```
Broken (or in progress) Kubernetes node setup!Check the cluster initialization status using the following commands.Master instance:
  - sudo systemctl status kube-master-installation
  - sudo systemctl status kube-master-configuration
```

运行*sudo system CTL status kube-master-installation*时我看到 python 的 yaml 包不存在。那现在呢。如果我尝试不同的操作系统会怎样？这就是我所做的，我改变了上面提到的 Kubernetes 变量来使用 CoreOS(实际上它现在被称为 Container Linux ),但这也很糟糕。Container Linux 的问题不是缺少 python 包，而是最糟糕的是不再支持 Container Linux 脚本。有趣的是，我刚刚检查了一下，对容器 Linux 的支持在 10 小时前就被移除了。

那么，GCE Ubuntu 镜像有什么问题呢？为了测试另一件事，我试着用 Ubuntu 启动了一个 GKE 集群，并且成功了。那时我意识到 GCE Ubuntu 的图像肯定不同于 GKE 的图像(现在看来很明显)。GCE one 基本上是一个普通的 Ubuntu，带有一些 GCP 特有的东西，比如内核。

下一个问题是:我如何在 Kubernetes 发布之前安装缺失的依赖项？由于我们手动部署 Kubernetes，我最初的想法是将其添加到 Kubernetes 启动脚本中，但事情变得棘手，感觉不太干净。因此，我在 GCP slack 频道询问，有人( *mickej* )建议建立我自己的定制形象。我不知道该怎么做，但是 GCP 有很好的关于[如何创建自定义图像](https://cloud.google.com/compute/docs/images/create-delete-deprecate-private-images)的文档，实际上它有很好的关于所有事情的文档。我编写了一个脚本来创建一个基于现有映像的映像，并安装 Kubernetes 缺少的所有依赖项。

下面的脚本基于现有的 GCE Ubuntu 映像创建一个新的实例，然后上传另一个脚本(见下文)来执行安装步骤，然后停止实例并基于实例的磁盘创建一个新的映像，最后删除实例。

到目前为止一切顺利。此时，我崭新的 Kubernetes 1.9.1 集群已经启动并运行，除了 sysdig。那么，现在怎么办？我检查了 sysdig 的 pod 日志，驱动程序正在构建，但由于某种原因，它没有被加载到内核中。在 Kubernetes minion 中运行 *dmesg* 向我展示了原因:

```
[  507.078542] sysdigcloud_probe: failed to find page_fault_user tracepoint
[  507.142649] sysdigcloud_probe: driver loading, sysdigcloud-probe 0.75.0
```

我的反应是:这是什么？这些[跟踪点](https://www.kernel.org/doc/Documentation/trace/tracepoints.txt)是什么？现在怎么办？我从 github 克隆了 [sysdig repo](https://github.com/draios/sysdig) 并寻找 *page_fault_user* 。代码较新，这些消息有适当的错误处理。所以，我看了看历史，瞧！提交是几个小时前的事了，是由 Ubuntu 最近的一个变化引起的。但是我是怎么得到的呢？嗯，还记得我的基本 Ubuntu 镜像的名字吗？*Ubuntu-1604-xenial-v 2018 01 09*。我只是被最近的变化所困扰。好消息是 sysdig 的人非常快，几个小时后 sysdig 代理 0.76.0 就出来了。sysdig 现在是我们集群的快乐运行公民。

等等，你不是在标题里提到了 [packer](https://www.packer.io/) 吗？啊，对了。嗯，在 sysdig 发现之后，已经是凌晨 1:30 了，我已经在考虑写一篇关于这一切的文章。有一件事感觉很尴尬；也许有更好的方法来创建自定义图像。我睡着了。

今天早上，一个同事在查看我的合并请求时，说了这样的话:这很酷，但是你有没有考虑过使用 packer 或类似的工具？我很惭愧，但我对帕克一无所知。快速搜索后，我找到了我真正需要的东西:[谷歌计算构建器](https://www.packer.io/docs/builders/googlecompute.html)。在大约 30 分钟和一些评论之后，我用下面的文件调用 packer，替换了 *create-image.sh* 脚本:

一切都结束得很好:我学习了如何创建自定义映像的手动步骤，什么是 Linux 内核跟踪点，关于 packer 和其他一些东西。这就是故事的结尾。