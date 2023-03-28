# 零停机更新 Google Kubernetes 引擎虚拟机范围

> 原文：<https://medium.com/google-cloud/updating-google-container-engine-vm-scopes-with-zero-downtime-50bff87e5f80?source=collection_archive---------0----------------------->

我经常在谷歌云平台 Slack 上闲逛，这是一个学习和讨论 GCP 的好社区。这是我见过很多人遇到的一种常见情况:

*“我想用云 SQL / Datastore / PubSub /等。从我的豆荚在 Kubernetes 引擎运行。我想使用自动虚拟机凭据，但我的虚拟机没有正确的范围或权限，我无法更改它！我该如何解决这个问题？”*

所有谷歌计算引擎虚拟机都带有内置的 OAuth2 服务帐户，可用于自动认证各种 GCP 服务。您可以通过给帐户分配不同的“范围”来选取此服务帐户有权访问的服务。

例如，您可以赋予服务帐户“devstorage.read_only”范围，这样它只能从 Google 云存储中读取数据。您可以给另一个服务帐户“devstorage.read_write ”,这样它就可以读写数据。一个帐户可以有任意组合的范围，因此您可以给它完成工作所需的确切权限。你可以在这里找到所有谷歌搜索范围[的列表](https://developers.google.com/identity/protocols/googlescopes)。

这个服务帐户通常被称为“[应用程序默认凭证](https://developers.google.com/identity/protocols/application-default-credentials)

# 应用程序默认凭据的问题

̶o̶n̶c̶e̶̶y̶o̶u̶̶c̶r̶e̶a̶t̶e̶̶t̶h̶e̶̶i̶n̶s̶t̶a̶n̶c̶e̶,̶̶y̶o̶u̶̶c̶a̶n̶'̶t̶̶c̶h̶a̶n̶g̶e̶̶t̶h̶e̶̶s̶c̶o̶p̶e̶s̶！̶̶t̶h̶e̶r̶e̶̶i̶s̶̶n̶o̶̶w̶a̶y̶̶t̶o̶̶f̶l̶i̶p̶̶o̶u̶t̶̶t̶h̶e̶̶a̶p̶p̶l̶i̶c̶a̶t̶i̶o̶n̶̶d̶e̶f̶a̶u̶l̶t̶̶c̶r̶e̶d̶e̶n̶t̶i̶a̶l̶s̶̶w̶i̶t̶h̶̶a̶n̶o̶t̶h̶e̶r̶̶o̶n̶e̶.̶(情况不再如此，[计算引擎现在支持在运行实例上改变范围](https://cloud.google.com/compute/docs/access/create-enable-service-accounts-for-instances#changeserviceaccountandscopes)。但是，我仍然建议遵循这个指南。如果虚拟机崩溃并重新运行，或者您更改了群集大小，新虚拟机将不会有更新的范围)

有两种解决方案:

1.  [创建一个具有正确作用域的服务帐户](https://www.youtube.com/watch?v=tSnzoW4RlaQ)，并在应用程序中直接使用它*而不是使用应用程序默认凭证*。
2.  创建一个具有正确作用域的新实例，并移动所有内容。

第一种方法肯定更健壮，因为如果需要的话，每个 pod 可以有自己的服务帐户，但是有额外的开销，迫使您管理这些帐户。我不会在这篇文章中讨论这种方法，但是如果你想要更多的控制，这是一个很好的选择。

第二种方法意味着您不必进行任何代码更改或管理帐户，但在新虚拟机启动时，您有停机的风险。

使用谷歌 Kubernetes 引擎，如果你遵循一些简单的步骤，你可以避免这种停机时间。我们来看看吧！

# 初始设置

在这篇博文中，我们将在 Google Kubernetes 引擎上运行一个小型的 3 节点 Kubernetes 集群，该集群运行一个由部署支持的服务。该部署将有 6 个副本。

以下是节点:

```
$ kubectl get nodes
NAME                                      STATUS  AGE
gke-cluster-1-default-pool-7d6b79ce-0s6z  Ready   2m
gke-cluster-1-default-pool-7d6b79ce-9kkm  Ready   2m
gke-cluster-1-default-pool-7d6b79ce-j6ch  Ready   2m
```

以下是窗格(根据屏幕进行了修改):

```
$ kubectl get pods -o wide
NAME                        NODE
hello-1959708372-25x63      gke-cluster-1-default-pool-7d6b79ce-0s6z
hello-1959708372-c13v2      gke-cluster-1-default-pool-7d6b79ce-9kkm hello-1959708372-fdx7z      gke-cluster-1-default-pool-7d6b79ce-j6ch hello-1959708372-n510f      gke-cluster-1-default-pool-7d6b79ce-0s6z hello-1959708372-xhz0h      gke-cluster-1-default-pool-7d6b79ce-9kkm hello-1959708372-zdmvb      gke-cluster-1-default-pool-7d6b79ce-0s6z
```

您可以看到这些单元分布在各个节点上。

# 灾难来袭！

哦不不。节点没有正确的权限！

首先要做的是创建具有正确权限的新节点。为此，您可以创建一个与旧池大小相同的新节点池。这个新的节点池将与旧的节点池并排放置，可以在其上安排新的单元。

假设您的代码需要“devstorage.read_write”和“pubsub”范围。

要创建新的节点池，请运行以下命令:

```
$ gcloud container node-pools create adjust-scope \
   --cluster <YOUR_CLUSTER_NAME> --zone <YOUR_ZONE> \
   --num-nodes 3 \
   --scopes https://www.googleapis.com/auth/devstorage.read_write,https://www.googleapis.com/auth/pubsub
```

和往常一样，您可以定制这个命令来满足您的需求。

现在，如果您检查节点，您会注意到还有三个节点使用了新的池名称:

```
$ kubectl get nodes
NAME                                        STATUS  AGE
gke-cluster-1-adjust-scope-9ca78aa9–5gmk    Ready   9m
gke-cluster-1-adjust-scope-9ca78aa9–5w6w    Ready   9m
gke-cluster-1-adjust-scope-9ca78aa9-v88c    Ready   9m
gke-cluster-1-default-pool-7d6b79ce-0s6z    Ready   3h
gke-cluster-1-default-pool-7d6b79ce-9kkm    Ready   3h
gke-cluster-1-default-pool-7d6b79ce-j6ch    Ready   3h
```

但是，豆荚还在老节点上！

# 该排水了

此时，我们可以简单地删除旧的节点池。Kubernetes 将检测到 pod 不再运行，并将它们重新安排到新的节点。

然而，这会给应用程序带来一些停机时间，因为 Kubernetes 需要时间来检测新主机上的节点停止运行并启动容器。这可能只有几秒或几分钟，但这可能是不可接受的！

更好的方法是一次从旧节点中删除一个 pod，然后从集群中删除该节点。幸运的是，kubernetes 有一个内置的命令来做到这一点。

第一，[警戒线](https://kubernetes.io/docs/user-guide/kubectl/kubectl_cordon/)各个旧节点。这将阻止新的 pod 被安排到它们上面。

```
$ kubectl cordon <NODE_NAME>
```

然后，[漏](https://kubernetes.io/docs/user-guide/kubectl/kubectl_drain/)各节点。这将删除该节点上的所有窗格。

> 警告:确保你的 pod 由副本集、部署、状态集或类似的东西管理。独立舱不会被重新安排！

```
$ kubectl drain <NODE_NAME> --force
```

清空一个节点后，确保新的单元已经启动并运行，然后再继续下一个节点。

完成后，您可以看到所有的 pod 都在新节点上运行！

```
$ kubectl get pods -o wide
NAME                        NODE
hello-1959708372-neu42      gke-cluster-1-adjust-scope-9ca78aa9–5gmk
hello-1959708372-vvjd8      gke-cluster-1-adjust-scope-9ca78aa9-v88c hello-1959708372-cn28s      gke-cluster-1-adjust-scope-9ca78aa9–5gmk hello-1959708372-cm9sd      gke-cluster-1-adjust-scope-9ca78aa9–5w6w hello-1959708372-d92jh      gke-cluster-1-adjust-scope-9ca78aa9-v88c hello-1959708372-b039s      gke-cluster-1-adjust-scope-9ca78aa9–5w6w
```

# 删除旧池

现在所有的 pod 都已安全地重新安排，是时候删除旧池了。

将“默认池”替换为您要删除的池。

```
$ gcloud container node-pools delete default-pool \
   --cluster <YOUR_CLUSTER_NAME> --zone <YOUR_ZONE>
```

就是这样！您已经用新的作用域更新了您的集群，并且零停机！