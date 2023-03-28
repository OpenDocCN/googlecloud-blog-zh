# Google Cloud 上 kubernetes 服务的内部负载平衡

> 原文：<https://medium.com/google-cloud/internal-load-balancing-for-kubernetes-services-on-google-cloud-f8aef11fb1c4?source=collection_archive---------1----------------------->

正如我最近在 kubernetes ingress 上的帖子中所讨论的，集群外部的流量只有一种方式到达集群中运行的服务。您可以阅读这篇文章了解更多细节，但是 TL；灾难恢复是指所有外部流量通过节点端口进入集群，节点端口是在每个主机/节点上打开的端口。节点是短暂的，而集群被设计成可伸缩的，因此在客户端和节点端口之间总是需要某种负载平衡器。如果你在像 GKE 这样的云平台上运行，那么通常的方法是使用一个类型[负载平衡器](https://kubernetes.io/docs/concepts/services-networking/service/#type-loadbalancer)服务或者一个[入口](https://kubernetes.io/docs/concepts/services-networking/ingress/)，其中任何一个都会构建一个负载平衡器来处理外部流量。

这并不总是，甚至不经常是你想要的。您的情况可能有所不同，但在 Olark，我们部署的内部服务比外部服务多得多。直到最近，kubernetes 在 GKE 上创建的负载平衡器一直是外部可见的，也就是说，它们被分配了一个非私有的 IP，可以从项目外部到达。维护防火墙规则以保护大量内部服务不是我们想要做的权衡，因此对于这些用例，我们将服务创建为类型[节点端口](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport)，然后使用 [terraform](https://www.terraform.io/) 为它们提供内部 TCP 负载平衡器。

创建外部负载平衡器仍然是默认行为，但幸运的是，现在有一种方法来注释负载平衡器服务，以便 Google Compute Engine 将使用私有 IP 构建内部负载平衡器。当这个特性在几个月前第一次发布时，它不能正常工作，所以我们继续手动连接这些东西。它已经被修复，我最近能够测试它，并确认它的工作，所以现在似乎是一个很好的时间来快速看看这可以为你做什么。下面是一个示例服务:

```
**apiVersion: v1
kind: Service
metadata:
  name: test
  annotations:
    cloud.google.com/load-balancer-type: "Internal"
  labels:
    app: test-app
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
      name: http
  selector:
    app: test-app**
```

正如您在上面看到的，这是一个简单的单行注释。假设我们有一些端点(pods)来处理这个流量，那么当我们创建这个服务时，一两分钟后，我们会看到一个内部 lb 指向它:

```
**$ kubectl get svc test
NAME TYPE         CLUSTER-IP  EXTERNAL-IP   PORT(S)           AGE
test LoadBalancer 10.3.250.65 10.130.15.192 80:31755/TCP      3m**
```

该列标题中的“EXTERNAL_IP”表示“在集群之外”，正如您所看到的，分配的地址位于项目的私有地址空间中。相当酷。但是，您应该知道内部负载平衡器有一些限制。它们是区域性设备，无法处理与其他地区的虚拟机之间的流量。它们是第 4 层，不能处理第 7 层的 http/s 流量，这意味着没有虚拟主机、基于路径的路由、tls 终止等。它们最多只能处理五个端口。之后，你需要再做一磅。最后一个不是很大的限制，因为据我所知，kubernetes 将驱动 GCE 为每个服务端口创建一个新的内部 lb。这是我将尝试测试并在稍后报告的内容。

在任何情况下，这都是一个很好的功能，它将使我们更容易从我们的 [helm](https://github.com/kubernetes/helm) 图表存储库中为我们的 kubernetes 服务规范和构建网络堆栈，而不是必须维护一个独立的 terraform 配置，该配置依赖于后端实例组的手动发现，以便正确配置内部负载平衡器。一如既往，在配置方面，越少越好。