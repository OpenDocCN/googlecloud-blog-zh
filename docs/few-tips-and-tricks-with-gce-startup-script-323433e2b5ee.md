# 关于 GCE 启动脚本的一些提示和技巧

> 原文：<https://medium.com/google-cloud/few-tips-and-tricks-with-gce-startup-script-323433e2b5ee?source=collection_archive---------0----------------------->

在 Google Cloud 中，可以为虚拟机配置一个启动脚本，该脚本将在虚拟机每次启动时启动。运行启动脚本很容易，有很多博客文章描述了如何做，包括 GCP 公共文档。启动脚本为您提供了一个通用的问题解决方案，用于定义每次虚拟机启动时要运行的命令。像任何事情一样，很少有警告。

# 一次性启动脚本

AWS 提供了区分 EC2 启动和启动事件的功能。这提供了仅在 EC2 实例首次引导时运行脚本的功能。GCE 不直接提供这个功能。然而，方法很简单让脚本的运行有条件。实现它的最简单方法是在脚本执行结束时创建一个空文件，并以文件存在作为启动脚本执行的条件:

```
if [[ -f /etc/startup_was_launched ]]; then exit 0; fi
…
touch /etc/startup_was_launched
```

另一种方法是利用 GCE 实例元数据的[客户属性](https://cloud.google.com/compute/docs/storing-retrieving-metadata#guest_attributes)。它的工作方式与文件类似，但是测试必须向元数据服务器发送 REST API 调用。使用 guest 属性不太理想，因为在文件由 root 创建和拥有时，任何人都可以读取和修改它们。

# 如何知道启动脚本运行完毕？

有时，当您编写运行虚拟机的 shell 脚本或其他自动化程序时，您需要弄清楚启动脚本何时结束运行。例如，在虚拟机启动后，你可能需要做一些动作*，虚拟机上所有需要启动的东西都在运行。目前还没有可管理的解决方案。您可以使用云日志记录来捕获启动脚本输出，并查找以“startup-script”开头的行:*

```
# INSTANCE_ID=$(gcloud compute instances describe $VM_NAME \
    --zone=$VM_ZONE --format=”value(id)”)
# gcloud logging read "resource.type=gce_instance AND \
    resource.labels.instance_id=\"$INSTANCE_ID\" AND\   
    jsonPayload.message=~\"^startup-script:\""
```

但是，这些行表示执行事件，并不捕获执行实际完成的时间，因此不可能确定启动脚本的所有命令何时结束运行。另一种方法是应用前面的建议，使用文件或 guest 属性作为启动脚本执行结束的标记。要使用它，启动脚本应该知道这个标记，并在开始时重置它。还应仔细选择标记，以避免在 GCP 管理活动中被破坏(例如，禁用虚拟机上的来宾元数据)或被虚拟机操作系统的维护流程删除。由 [Nazia Mahimi](https://medium.com/u/a255b051be06?source=post_page-----323433e2b5ee--------------------------------) 提出的另一种方法是通过运行以下命令来捕获串行输出:

```
# gcloud compute instances get-serial-port-output $VM_NAME \
    --zone=$VM_ZONE | grep \
    "<instance_name> systemd: Startup finished"
```

该命令可以在 VM 启动后定期启动，以检查启动脚本是否执行完毕。它也不需要对启动脚本进行任何修改来清理和设置结束脚本标记。这种方法有几个考虑因素。(1)它要求允许到 GCE 实例的串行端口连接。如果你的 GCP 组织强制执行`constraints/compute.disableSerialPortAccess`组织。策略约束该命令将不起作用。(2)考虑到最小化攻击面的安全实践以及虚拟机已经提供 SSH/RDP 连接的事实，没有理由开放另一个连接选项。

推荐的解决方案是使用现有的 SSH/RDP 连接来解析启动脚本的执行日志。日志被捕获到`/var/log/syslog`中，可以通过查找带有`"startup-script exit status"`的行来查询:

```
grep -m 1 “startup-script exit status” /var/log/syslog
```

状态被捕获为 0(成功)或 1(失败),并且可以从捕获的行中解析。请注意检查时间戳，以确保它是最近发生的，并且您没有从上一次重启中捕获日志行。

要运行此行，您需要能够在虚拟机上运行 SSH 命令:

```
gcloud compute ssh $VM_NAME --zone=$VM_ZONE --ssh-flag="-q" \
    --command='grep -m 1 "startup-script exit status" /var/log/syslog' 2>&-
```

在您执行命令时，如果 VM 没有启动或者 SSH 端口不可用，那么`2>&-`保证您不会得到`gcloud`错误输出。

我使用下面的 bash 脚本探测 shell 脚本在第一次启动时完成。它提供了等待“正确”日志条目的进度指示壁:

对于多次虚拟机重启，该脚本将无法正常工作，因为它将总是捕获第一次出现的`"startup-script exit status"`并将退出。引入时间阈值(在以下示例中为 5 分钟)并自下而上反转系统日志解析可以解决问题(假设虚拟机不会频繁重启):

# 在启动脚本中使用环境变量

可以将自定义值注入到启动脚本中。然而，当启动脚本创建一个应该使用一些环境变量的 shell 脚本文件时，这可能是一个障碍。以下命令创建了一个具有`/etc/sample.sh` shell 脚本的虚拟机:

```
gcloud compute instances create $VM_NAME --zone=$VM_ZONE \
    --metadata startup-script='#!/bin/bash
cat > /etc/sample.sh <<EOF
#!/bin/bash
echo "my home directory is at $HOME"
EOF'
```

`/etc/sample.sh`的预期内容是:

```
#!/bin/bash
echo "my home directory is at $HOME"
```

但是运行上面的启动脚本会产生看起来像这样的`/etc/sample.sh`:

```
#!/bin/bash
echo "my home directory is at "
```

这是因为$HOME 将按照自定义值注入逻辑被计算为一个空字符串(在 VM 实例的元数据中没有这样的键)。解决方案就是对美元符号进行转义，这样 shell 脚本就会像预期的那样:

```
gcloud compute instances create $VM_NAME --zone=$VM_ZONE \
    --metadata startup-script='#!/bin/bash
cat > /etc/sample.sh <<EOF
#! /bin/bash
echo "my home directory is at \$HOME"
EOF'
```

享受启动脚本[👍](https://emojipedia.org/thumbs-up/)