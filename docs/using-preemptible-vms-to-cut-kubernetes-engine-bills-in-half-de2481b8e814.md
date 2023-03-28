# 使用可抢占的虚拟机将 Kubernetes 引擎账单削减一半

> 原文：<https://medium.com/google-cloud/using-preemptible-vms-to-cut-kubernetes-engine-bills-in-half-de2481b8e814?source=collection_archive---------0----------------------->

可抢占虚拟机的固定价格高达常规实例的 80%,但不幸的是，它们大多被宣传为运行短期批处理作业。在这篇博文中，我们将看到在 Google 云平台上混合使用可抢占的和常规的 Kubernetes 节点如何在不牺牲应用程序稳定性的情况下节省大量资金。

首先，**它对任何应用都不起作用**。这不是银弹！您的应用程序应该对一些 pod 的意外故障具有**容忍度**。让我们看一个例子，我们的意思是什么。

在我们的例子中，我们在 Kubernetes 引擎上运行 Cirrus CI。Cirrus CI 由 20 个微服务和其他一些很酷的技术支持。每项服务都由至少两个配置了[水平机架自动缩放器](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)的机架提供支持，以便在需要时进行纵向扩展。除了两个微服务之外，所有微服务都是无状态的，可以容忍任何 pod 的故障。其他两个一般来说也是无状态的，但是在他们的情况下，我们希望最小化意外的 pod 故障，因为客户端重新连接并继续使用它们的成本非常高。例如，其中一个负责从 CI 代理上传缓存，如果发生故障，重新上传缓存的成本会高得不合理。

**想法很简单:我们希望调度 pods，我们可以容忍可抢占节点上的故障。**我们还需要确保我们永远不会在可抢占节点上调度我们不能容忍的 pods 故障。

# Kubernetes 引擎设置

您的 Kubernetes 集群应该有一个可抢占的节点池。如果没有，这里有一个`gcloud`命令来创建一个可自动扩展的可预优化虚拟机节点池:

```
*gcloud* container node-pools create preemtible-pool \
  --cluster $CLUSTER_NAME \
  --zone $CLUSTER_ZONE \
  --scopes cloud-platform \
  --enable-autoupgrade \
  --preemptible \
  --num-nodes 1 --machine-type n1-standard-8 \
  --enable-autoscaling --min-nodes=0 --max-nodes=6
```

**注意:**可抢占池中的节点将有`cloud.google.com/gke-preeptible: true`标签。

# 节点关联性

为了控制 pods 的调度我们可以使用 [***节点亲和***](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#node-affinity-beta-feature) 功能。通过节点关联，我们可以为 pod 的调度设置硬偏好和软偏好。

## 硬性偏好

硬首选项是确保某些 pods 永远不会被安排在可抢占的节点上的理想选择。我们不能容忍的失败。使用`requiredDuringSchedulingIgnoreDuringExecution`,我们可以确保 pods 不会被安排在带有`cloud.google.com/gke-preeptible`标签的节点上。只需将以下几行添加到您的*吊舱*或*部署*规范中:

```
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: cloud.google.com/gke-preemptible
          operator: DoesNotExist
```

## 软偏好

另一方面，软偏好只是告诉 Kubernetes 调度的偏好。例如**Google 计算引擎不能保证可抢占的虚拟机始终可用**。对于我们的用例，我们只想告诉 Kubernetes 类似“如果可以的话，请在带有`cloud.google.com/gke-preemptible`标签的节点上安排这些 pods。如果不是，那就无所谓了”。以下是如何修改您的*吊舱*或*部署*规格，以便使用`preferredDuringSchedulingIgnoreDuringExecution`实现这一点:

```
affinity:
  nodeAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - preference:
        matchExpressions:
        - key: cloud.google.com/gke-preemptible
          operator: Exists
      weight: 100 
```

# 结果

对于我们的 18 个无状态微服务，我们有这样的软偏好，而对于我们上面描述的两个特殊情况，我们有硬偏好。结果令人印象深刻！**平均而言，我们 75%的生产工作负载运行在可抢占的节点上，这可以节省 50%的 GCP 费用。**

请[在 Twitter 上关注我们](https://twitter.com/cirrus_labs),如果您有任何问题，请随时提问！