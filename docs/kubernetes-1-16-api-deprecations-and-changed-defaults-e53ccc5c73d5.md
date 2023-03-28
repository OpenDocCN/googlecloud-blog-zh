# Kubernetes 1.16 API 弃用和更改默认值

> 原文：<https://medium.com/google-cloud/kubernetes-1-16-api-deprecations-and-changed-defaults-e53ccc5c73d5?source=collection_archive---------1----------------------->

**TL；dr** : Kubernetes 1.16 正在[移除对一些仍在广泛使用的废弃 API 的支持](https://kubernetes.io/blog/2019/07/18/api-deprecations-in-1-16/)。解决方案理论上很简单:只需使用例如*版本:apps/v1* 而不是*版本:extensions/v1beta1* 。然而，有一个副作用:默认值可能会改变。

Kubernetes 做得很好的一点是处理 API 的不断变化和改进，同时避免破坏向后兼容性。其工作方式是，每个 API 都声明了一个特定的版本，如果您继续使用该版本，那么在升级 Kubernetes 时，API 应该继续做与以前相同的事情。

当 Kubernetes 开发人员想要更改 API 中某些参数的默认值时，可以使用相同的向后兼容性概念。当 API 的新版本发布时，可以引入新的默认值。如果您使用旧版本的 API，您将获得旧的默认值，如果您使用新版本，您将获得新的默认值。这是一个美丽的概念，但也可能让用户感到惊讶，当他们只是改变版本，如果他们的资源定义。

现在，一些 API 版本将被删除，您应该知道在使用这些 API 的新版本时会发生什么变化。

# 1.16 中删除了什么

Kubernetes 正在移除对一些 beta APIs 的支持，这些 API 已经被 GA APIs 取代很久了(在 Kubernetes 1.9 中)。其中大部分是核心工作负载 API 的一部分，这些 API 通常用于定义 Kubernetes 中的应用程序。以下是摘自 Kubernetes 1.16 的[发行说明的核心工作负载 API 列表，这些 API 仅受版本 **apps/v1** 支持:](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG-1.16.md#deprecations-and-removals)

*   达蒙塞特
*   部署
*   状态集
*   复制集

其他被删除的 API(作为参考，我们将在本文的其余部分重点关注上面的):

*   网络策略(扩展/v1beta1，改为使用 networking.k8s.io/v1)
*   PodSecurityPolicy(扩展/v1beta1，请改用 policy/v1beta1)

# 如何解决您的工作负载

修复非常简单:只需更改您的资源定义以使用新的 API 版本。例如，如果你有:

```
apiVersion: **extensions/v1beta1**
kind: Deployment
metadata:
  name: mydeployment
```

只需更改第一行:

```
apiVersion: **apps/v1**
kind: Deployment
metadata:
  name: mydeployment
```

就这些吗？几乎…如上所述，您应该考虑到更改您的资源定义的 API 版本会对用于您没有定义的参数的默认值产生影响。下一节包含核心工作负载 API 中已更改的默认值列表。

# 更改默认值

## 达蒙塞特

DaemonSets 用于部署需要出现在每个节点上的 pod。这些默认值从 **v1** 开始改变:

*   [**spec . update strategy . type**](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#update-strategies):on delete->RollingUpdate
    用“rolling update”，当 DaemonSet 定义改变时，会自动重新创建 pod。与 OnDelete 相比，这是一个很大的变化，在 on delete 中，您必须手动删除 pod 才能推出新版本。

## 部署

部署可能是定义工作负载最常用的方式。这些默认值从**v1β2**开始更改:

*   [**spec . progressDeadlineSeconds**](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#writing-a-deployment-spec)**:**2147483647→600
    progressDeadlineSeconds 确定部署在被宣布为失败(例如由于准备就绪检查失败)之前将保持“进行中”的时间。
*   [**spec . revisionHistoryLimit**](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#writing-a-deployment-spec)**:**2147483647→10
    revisionHistoryLimit 限制在被部署控制器删除之前将保留的 ReplicaSet 对象的数量。
*   [**spec . strategy . rolling update . max surge**](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#writing-a-deployment-spec)**:**1→25%
    max surge 定义滚动更新可以创建的超过 spec.replicas 中指定的所需 pod 数量的最大 pod 数量
*   [**spec . strategy . rolling update . max unavailable**](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#writing-a-deployment-spec)**:**1→25%
    仅当不可用(未就绪)pod 的数量低于此数量时，滚动更新才会继续。如果是百分比，就是规格复制品的百分比

## 复制集

副本集很少被直接使用，通常被视为部署所拥有的子对象。extensions/v1beta1 和 apps/v1 之间的副本集默认值没有改变。

## 状态集

StatefulSets 类似于 Deployment，但是更适合需要存储、可寻址实例、创建顺序控制等的有状态应用程序。与 DaemonSet 相同的参数从 **v1beta2** 开始改变:

*   [**spec . update strategy . type**](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#update-strategies):on delete→RollingUpdate
    用“rolling update”，当 StatefulSet 定义改变时，会自动重新创建 pod。与 OnDelete 相比，这是一个很大的变化，在 on delete 中，您必须手动删除 pod 才能推出新版本。

# 删除默认选择器和选择器不变性

当转移到非 beta 核心工作负载 API 时，您可能会注意到其他一些东西。如果您有未定义 pod 选择器( *spec.selector* )的 Deployments 或 StatefuSet，您现在将得到一个该字段丢失的错误。在 beta APIs 中，您可以省略 *spec.selector* ，它将从 pod 模板中的标签推断出来。

另一件不再被允许的事情是改变现有资源中的选择器。*规格选择器*字段现在被认为是不可变的。

您可以在核心工作负载 API [GA 公告](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)中找到关于这些变化原因的更多详细信息。

# 关于您从 kubectl get 获得的版本的说明

为了准备在 Kubernetes 1.16 中删除这些资源，您应该在升级之前更改资源定义中的版本。当您开始使用新版本(例如 apps/v1)时，您可能会注意到一些奇怪的事情:该版本显然没有注册:

```
$ cat test-deployment.yaml
apiVersion: **apps/v1**
kind: Deployment
metadata:
  name: test
...$ kubectl apply -f test-deployment.yaml
$ kubectl get deployment test -o yaml
apiVersion: **extensions/v1beta1** kind: Deployment
```

发生什么事了？正如这里的[所解释的那样](https://github.com/kubernetes/kubernetes/issues/58131#issuecomment-356823588)，原因是“kubectl get deployment”在您想要使用哪个 API 版本进行部署的问题上不明确。可以使用不同版本的可用 API 来检索相同的资源，Kubernetes < 1.16 仍然为所有这些旧 API 版本的部署提供服务。为了向后兼容，当版本不明确时，使用 API 的最老版本。

您可以使用以下语法检索特定的 API 版本:

```
$ kubectl get deployment.v1.apps test -o yaml
apiVersion: apps/v1
kind: Deployment
```

感谢阅读！

*感谢 Balazs Pinter 和 Janet Kuo 帮助撰写本文。*