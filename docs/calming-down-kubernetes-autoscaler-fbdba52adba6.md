# 平息 Kubernetes 自动缩放器

> 原文：<https://medium.com/google-cloud/calming-down-kubernetes-autoscaler-fbdba52adba6?source=collection_archive---------0----------------------->

[Kubernetes Autoscaler](https://github.com/kubernetes/autoscaler) 是一个巨大的省钱省！它会持续监控集群，并在可能的情况下尝试重新分配单元，以最大限度地利用节点。不幸的是，有时它太咄咄逼人，但**用一个小技巧 Kubernetes Autoscaler 可以平静下来！**

例如， [Cirrus CI](https://cirrus-ci.org/) 可以在 Google Kubernetes 引擎集群上运行 CI 任务。Cirrus CI 使用 Kubernetes Jobs API 为每个 CI 任务安排一个作业。当我们刚刚提出这种方法并对其进行负载测试时，似乎 Kubernetes Autoscaler 正在**杀死活动作业**并在其他节点上重新启动它们以最大化整体利用率。一般来说，这对于长时间运行的作业和 pod 是有意义的，但是在我们的例子中，我们确实知道 CI 作业有超时，并且通常在几分钟内完成。没有理由在另一个节点上重新启动 CI 作业。

在深入研究了 Kubernetes Autoscaler 的[内部之后，似乎默认情况下它并不驱逐具有本地存储的 pod。所以**只要安装一个** `**emptyDir**` **就可以避免驱逐！**](https://github.com/kubernetes/autoscaler/blob/fae2c903a3a6615fc53b748e7a17398128b60745/cluster-autoscaler/utils/drain/drain.go#L197-L199)

```
**apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: k8s.gcr.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /tmp
      name: temp-volume
  volumes:
  - name: temp-volume
    emptyDir: {}**
```

仅此而已！现在 Kubernetes Autoscaler 不会碰你的豆荚！但是如果您仍然希望对一些 pod 进行例外处理，您可以简单地向这些 pod 添加`cluster-autoscaler.kubernetes.io/safe-to-evict: true`注释。

**更新(09/28/2018):** `cluster-autoscaler.kubernetes.io/safe-to-evict: false`现在注释作品。

请阅读 Kubernetes Autoscaler 上的[常见问题解答，了解更多详细信息。我希望这篇小博文能够帮助您避免集群中的意外驱逐！](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/FAQ.md#when-does-cluster-autoscaler-change-the-size-of-a-cluster)