# 谷歌云(GCE)上的 Apache Cassandra 集群

> 原文：<https://medium.com/google-cloud/apache-cassandra-clusters-on-google-cloud-gce-78c040b66e73?source=collection_archive---------1----------------------->

当谈到 1click 安装广泛的解决方案时(不仅是 Apache Cassandra，还有其他关系和非关系数据库、应用服务器、设备等等)，每个主要的云平台(AWS、Azure 和 GCE)都有预构建的解决方案。我还没有看到(所以我不能谈论)AWS 和 Azure 的模板，但是对于 GCE，你有一个模板来启动单节点或 3 节点集群。据我所知，在这两种情况下，您都不能选择机器的大小，也不能添加额外的节点(更不用说，例如，您将最终使用标准的‘Test Cluster’集群名称)。所以我花时间整理了几个可以运行的脚本，这些脚本有助于用所需的 VM 配置和一些配置更改来设置 Apache Cassandra 集群。所有的脚本都在[我们的 git](http://git.academyofdata.com)T2 这里。不仅如此，你还会得到一个[脚本来启动 GCE](https://github.com/academyofdata/cassandra-cluster/blob/master/cluster-setup.sh) 上的 Cassandra 集群，但是运行在 docker 容器中(都在同一台机器上)。但是让我们简单地看一下在单个 GCE 虚拟机上启动 Cassandra 集群的脚本/命令。需要注意的是，与 GCE 的交互是使用 gcloud utility 完成的。

1.  gcloud-server-setup.sh

这个脚本启动第一个(其他节点的种子)节点。

它首先使用一些可编辑的变量创建一台机器(节点代表机器名称，区域代表部署区域，项目代表项目 Id，机器代表虚拟机类型——我在这里使用了 g1-small，但是可以随意选择其他类型)

```
gcloud compute instances create $NODE --zone $ZONE --machine-type $MACHINE --network "default" --maintenance-policy "MIGRATE" --scopes …… --image "https://www.googleapis.com/compute/v1/projects/ubuntu-os-cloud/global/images/family/ubuntu-1604-lts" --boot-disk-size "10" --boot-disk-type "pd-standard" --boot-disk-device-name "${NODE}disk1" --project $PROJECT
```

创建机器后，我们使用 gcloud 实用程序 ssh 命令连接到机器，并执行“标准”Apache Cassandra 安装和配置步骤——所有这些都取自 git [上的另一个脚本，这里是](https://github.com/academyofdata/cassandra-cluster/blob/master/setup39.sh)(注意，这个特定的脚本安装 Cassandra 3.9)。安装 Cassandra 后，我们使用另一个脚本(来自这里的)进行最小配置:更改集群名称，更改 listen 和 rpc_address(以便它使用 GCE 内部网络而不是 127.0.0.1 ),甚至为 ssh 用户/pass 登录添加一个新用户(在这种情况下，我们还需要在 SSH 中启用密码登录——详细信息请参见脚本)

2.gcloud-add-replicas.sh

当第一个节点启动并运行时，可以使用该脚本向集群添加一个或多个节点。它主要执行与初始节点设置相同的任务，只是做了一些调整(它希望种子节点 IP 知道从哪里引导数据)。第二个脚本很可能包含在第一个脚本中(只需增加几个测试，并在某些情况下执行稍微不同的命令)

当然，这不是一次点击安装(对于一个 3 节点集群，在 CLI 中可能只需执行 3 行代码)，但它提供了更大的灵活性