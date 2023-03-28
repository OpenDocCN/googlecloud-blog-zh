# 如何在 Google 云平台上制作一个自毁式 VM

> 原文：<https://medium.com/google-cloud/how-to-make-a-self-destructing-vm-on-google-cloud-platform-b99883745b62?source=collection_archive---------2----------------------->

## 通过启动具有预设生命周期的计算实例来节省资金。

> 更新 2022-12-06:GCP 增加了一个名为[自动终止](https://cloud.google.com/compute/docs/instances/limit-vm-runtime)的功能，让你的虚拟机自毁变得非常简单。试试看！

云的一个优点是它非常容易启动新的虚拟机(VM)。另一件不太好的事情是，忘记那些你制造的机器也非常容易，你会发现自己在为那些你并不真正需要的东西买单。在这篇文章中，我将展示如何使用 [Google Compute Engine](https://cloud.google.com/compute/) (GCE)来创建虚拟机(在 GCE 中也称为“实例”)，这些虚拟机在指定的时间后会自行删除，因此您可以启动它们，然后忘记它们。

![](img/9e21a0ef98706552e87bbe6875145ee9.png)

# 它是如何工作的

这项技术利用了 GCE 的[启动脚本](https://cloud.google.com/compute/docs/startupscript)特性:当创建一个实例时，我们在`metadata-from-file`参数中提供了一个**启动脚本**。一旦实例完成引导，它就执行脚本，开始倒计时删除。为了实现这一点，脚本使用 linux `at`命令调度一个任务:在预定的时间，该任务将指示 GCE API 删除其主机实例。

为了增加灵活性，我们可以在实例创建时将自毁间隔作为变量传递。这允许不同的寿命用于不同的目的。我们将使用一个 GCE [定制元数据](https://cloud.google.com/compute/docs/storing-retrieving-metadata#custom)字段:当创建每个实例时，在该实例上设置一个名为`SELF_DESTRUCT_INTERVAL_MINUTES`的字段。在启动时，实例将向 GCE 元数据服务器请求指定的时间间隔，并相应地安排其自我销毁。

# 尝试一下

在 [**这个 GitHub repo**](https://github.com/davidstanke/samples/tree/master/self-destructing-vm) 中，你会发现[启动脚本](https://github.com/davidstanke/samples/blob/master/self-destructing-vm/self-destruct.sh)，以及一个创建自毁实例的示例`gcloud`命令。以下是该命令的一部分，有趣的部分以粗体显示:

```
gcloud compute instances create \
  self-destructing-vm \
  [...] \
 **--metadata SELF_DESTRUCT_INTERVAL_MINUTES=2 \
  --metadata-from-file startup-script=self-destruct.sh**
```

您可以根据需要修改实例名称和区域。这个命令已经在 Ubuntu 16.04 和 18.04 上测试过了，可能也适用于其他 Linux 发行版。(类似的技术很可能被用于自删除 Windows 实例。)

# 但是我究竟为什么要这么做呢？！？

因为好玩！比如吹泡泡，看泡泡爆掉。但同时:有时我们需要确保我们提供的资源将被取消，这样它们就不会永远闲置，浪费金钱。也许，对于持续集成:作为 CI 管道的一部分，我们可以创建一个一次性使用的环境，并针对它运行测试。如果管道失败(正如管道所做的那样)，我们知道环境将在合理的时间范围内被自动删除。

*在即将发布的帖子中，我将描述这一点:使用自毁虚拟机作为专用、短暂测试环境的 CI 管道。请稍后再来查看！*