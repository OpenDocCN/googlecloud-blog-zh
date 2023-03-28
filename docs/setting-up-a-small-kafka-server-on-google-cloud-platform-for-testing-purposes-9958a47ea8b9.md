# 在 Google 云平台上设置一个小型 Kafka 服务器用于测试

> 原文：<https://medium.com/google-cloud/setting-up-a-small-kafka-server-on-google-cloud-platform-for-testing-purposes-9958a47ea8b9?source=collection_archive---------1----------------------->

![](img/ea679e7d5ada15a4bbeb6d0fc8fdb819.png)

我们一起玩卡夫卡吧！

在许多情况下，我们可能希望在[谷歌云平台(GCP)](http://cloud.google.com) 上有一个 [Kafka 服务器](https://kafka.apache.org/)来做一些测试——例如，测试一个流[数据流](http://cloud.google.com/dataflow)管道如何与 Kafka 一起工作。

GCP 市场上有几个选项，其中一些提供了在集群上部署 Confluent 的 Kafka 的全功能部署。然而，很可能我们不想建立一个可伸缩的 Kafka 集群，也不想为一些小的测试支付许可费。同时，在 VM 中手动部署 Kafka 也很麻烦。难道我们不能有一个中间选项，允许我们在半托管的解决方案中使用开源的 Kafka 版本吗？

市场中可用的选项之一解决了这个问题:[https://console . cloud . Google . com/market place/details/click-to-deploy-images/Kafka](https://console.cloud.google.com/marketplace/details/click-to-deploy-images/kafka?)

该选项部署了一台 *n1-standard-1* 机器，估计每月花费 24.75 美元——如果我们想用我们的免费 GCP 信用来测试 Kafka，这是完美的选择！(如果我们根据需要打开和关闭它，我们只需支付非常少的费用)。

在这篇文章的其余部分，我假设你已经准备好了一个谷歌云平台项目。你可以按照这个例子使用免费的 GCP 信用点数。

# 设置 Kafka 服务器

转到以下链接并选择您的 GCP 项目:[https://console . cloud . Google . com/market place/details/click-to-deploy-images/Kafka](https://console.cloud.google.com/marketplace/details/click-to-deploy-images/kafka)

点击蓝色按钮*在计算引擎上启动。*

将*部署名称*改为 *kafka* (会生成一个名为 *kafka-vm* 的 VM)，选择一个 zone。在我的例子中，我将选择*欧洲-西方 1-d.*

让我们在控制台的[云壳中查看一下。在提示中，您应该看到黄色的项目名称，在这里我们检查它以确保我们列出了我们项目的虚拟机(在我的例子中，我的项目是 *ihr-kafka-dataflow* )。我们在云 Shell 中编写的命令以粗体显示。在本文的其余部分，云 Shell 提示符将只是`**$**`(其他虚拟机的提示符将在`**$**`符号前有机器的名称):](https://console.cloud.google.com/home/dashboard?cloudshell=true)

```
**$ gcloud config get-value project**
Your active configuration is: [cloudshell-14881]
ihr-kafka-dataflow
**$ gcloud compute instances list**
NAME     ZONE MACHINE_TYPE PRE... INTERNAL_IP EXTERNAL_IP    STATUS
kafka-vm eu.. n1-standard-1       10.132.0.3  146.148.22.176 RUNNING
```

注意`EXTERNAL_IP`字段。我们可能认为我们的 Kafka 服务器是暴露在世界面前的。但是，默认情况下，不允许任何流量连接到该 IP。

# 检查子网中与 Kafka 的连接

流量被限制在同一个子网内。让我们在同一个区域创建另一个小型虚拟机来测试与 Kafka 服务器的连接。我们创建机器(如果您的首选区域不同，请更改区域):

```
**$ export ZONE=europe-west1-d**
**$ gcloud compute instances create test-vm --zone=$ZONE                 --machine-type=g1-small** Created [[https://www.googleapis.com/compute](https://www.googleapis.com/compute/v1/projects/ihr-kafka-dataflow/zones/europe-west1-d/instances/test-vm)...
NAME     ZONE            MACHINE_TYPE  PREEMPTIBLE  INTERNAL_IP ...
test-vm  europe-west1-d  g1-small                   10.132.0.4  ....
```

现在让我们连接到这台机器，并检查它是否可以到达 Kafka 服务器(我们将必须创建一个 SSH 密钥，您可以留下一个空密码作为访问私钥的密码)。我们还安装了命令行 Kafka 客户端，以检查我们是否能够连接到 Kafka 代理。请注意，在对 test-vm 执行 SSH 之后，提示符是`test-vm`之一:

```
**$ gcloud compute ssh test-vm --zone=$ZONE** ...
**ihr@test-vm:~$ export BROKER_IP=10.132.0.3
ihr@test-vm:~$ sudo apt install -y kafkacat** Reading package lists... Done
Building dependency tree       
Reading state information... Done
The following additional packages will be installed:
  librdkafka1 libyajl2
The following NEW packages will be installed:
  kafkacat librdkafka1 libyajl2
[...]
Setting up kafkacat (1.3.0-1+b1) ...
**ihr@test-vm:~$ echo 'Hi from test vm' | kafkacat -b $BROKER_IP:9092 -t test_topic**
% Auto-selecting Producer mode (use -P or -C to override)
**ihr@test-vm:~$ kafkacat -e -b $BROKER_IP:9092 -t test_topic**
% Auto-selecting Consumer mode (use -P or -C to override)
***Hi from test vm***
% Reached end of topic test_topic [0] at offset 3
**ihr@test-vm:~$ exit**
```

因此，与 Kafka 服务器的连接工作正常！

如果您需要将连接扩展到其他网络，您可能需要添加一些防火墙规则。注意:你的虚拟机有一个公共接口，取决于你如何设置你的防火墙规则，你可能会向世界开放你的 Kafka 服务器！

# 将此服务器与其他服务一起使用

现在我们有了一个 Kafka 服务器，我们如何与其他服务共享呢？

当我们需要指定连接到 Kafka 服务器的设置时，该服务将需要 2 或 3 个参数:

*   Kafka 经纪人地址
*   卡夫卡经纪人港
*   主题名称

在我们的机器中，因为我们没有使用集群，而只是使用了一个虚拟机，所以 Kafka 代理地址就是虚拟机的 IP 地址。如果我们让设置如上所示，我们可以使用私有 IP。有了适当的防火墙规则，您也可以使用公共 IP 来提供外部服务。但是请注意，在这个小的测试虚拟机中没有身份验证，所以您的 Kafka 服务器将对世界开放。

端口是 9092，这是 Kafka 中默认的代理端口。

至于主题名，服务器是“空的”，所以您需要创建一个主题。《Apache Kafka 快速入门》包含了一些关于如何创建主题的细节:[https://kafka.apache.org/quickstart](https://kafka.apache.org/quickstart)(参见步骤 3)。

Kafka 安装在`/opt/kafka`中的 VM 中，在那里你也可以找到一些管理服务器的工具。您将在`/opt/kafka/bin/kafka-topics.sh.`中找到第 3 步中使用的实用程序，选项`—-bootstrap-server`在我们虚拟机的脚本中不存在，您需要使用`--zookeeper`来代替(本地主机作为服务器，端口 2181 是默认的 Zookeeper 端口)。

如果你想创建一个主题，连接到`kafka-vm`并使用那个脚本(从云外壳，使用下面的命令)

```
**$ gcloud compute ssh kafka-vm --zone=$ZONE** ...
**ihr@kafka-vm:~$ /opt/kafka/bin/kafka-topics.sh --create               --zookeeper localhost:2181  --replication-factor 1 --partitions 1   --topic mytopic** Created topic "mytopic".
**ihr@kafka-vm:~$ /opt/kafka/bin/kafka-topics.sh --list                
--zookeeper localhost:2181** mytopic
```

在任何情况下，许多服务将能够自己创建主题，因此可能没有必要手动创建它。

# 清除

现在，您可以通过使用云外壳中的 gcloud 来移除`test-vm`:

```
**$ gcloud compute instances delete test-vm --zone=$ZONE**
```

如果您想保留您的 Kafka 服务器，并为将来的测试节省一些资金，只需停止机器并根据需要启动它。要停止机器:

```
**$ gcloud compute instances stop kafka-vm --zone=$ZONE**
```

开始吧:

```
**$ gcloud compute instances start kafka-vm --zone=$ZONE**
```

磁盘容量为 10 GB，因此如果机器停止运行，您每月需要支付大约 0.40 美元来维护存储的磁盘。

如果您想完全删除您在此项目中所做的一切，您可以在 GCP 控制台中删除该项目。要删除使用 Marketplace 部署脚本完成的设置，部署可从[https://console . cloud . Google . com/DM/deployments/details/Kafka](https://console.cloud.google.com/dm/deployments/details/kafka)获得，您可以使用控制台中的页面删除它(打开该页面时选择您的特定项目)。

# 总结

在这篇文章中，我们已经看到了如何设置你自己的小型 Kafka 服务器进行测试。

*   我们使用了在 GCP 市场上可以买到的 Kafka 开源版本，在单个虚拟机中部署 Kafka。
*   你可以使用 GCP 的免费版本来学习这个教程。如果这台机器正常运行，每月大约花费 25 美元，如果停止运行，每月大约花费 0.40 美元。
*   如果你需要改变服务器的配置，通过 SSH 连接到`kafka-vm`机器，Kafka 安装在`/opt/kafka`目录中。