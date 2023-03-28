# Istio 服务网格 101 —零件(2/3)

> 原文：<https://medium.com/google-cloud/istio-service-mesh-101-part-2-3-ceff88a38558?source=collection_archive---------0----------------------->

![](img/94bb00809bdc28037b83ae010d0a092b.png)

欢迎回到 Istio 服务网格 101 系列的另一部分。在[之前的博客](https://pilotudesh.medium.com/istio-service-mesh-101-part-1-3-f07a8fedeea8)中，我们了解了 Istio 及其安装。我们还看到了如何使用 Istio 部署应用程序。在这篇博客中， ***我们将探讨交通管理，并简要介绍一下 Istio*** 中的安全概念。

在前一篇文章中，我们已经使用 Istio 建立了一个服务网格，并部署了一个示例应用程序。现在，我们希望外部用户可以访问它。对于本演示，用户需要访问产品页面。

> 我们如何让它变得可访问？

假设我们需要使用[http://bookapp.com](http://bookapp.com)访问该网站，它应该显示产品页面，其中显示了所列产品的详细信息。

> 我们如何实现这一点？

使用 Kubernetes，我们可以使用入口资源来控制进入集群的流量。为此，我们可以 ***使用类似 NGINX 控制器的入口控制器，并定义一组规则来将流量路由到服务*** 。Istio 支持 Kubernetes Ingress，但他们还提供了另一种他们推荐的方法，该方法也提供了 Istio 的更多功能，如高级监控和路由规则。 *这个服务叫做****Istio Gateway***。

# 方法

> Istio 网关是位于服务网格边缘的负载平衡器。

**它们是管理服务网格的入站和出站流量的主要配置。**正如我们在之前关于在集群中安装 Istio 的讲座中所讨论的，Istio 安装在一个名为 istio-system 的名称空间中，有 3 个组件。

> 1.Istiod
> 
> 2.istio-Ingres 网关
> 
> 3.Istio-egressgateway

Istiod 是守护进程，另外两个是 Istio 网关控制器。***Istio-ingressgateway 管理所有进入这些服务的入站流量，而 Istio-egressgateway 管理所有来自这些服务的出站流量*** 。Kubernetes 上的 ingress 是使用 NGINX 这样的控制器部署的，而 Istio 使用 Envoy 代理部署这些网关控制器。

> 但是，请记住，这些网关控制器是独立的代理，独立于安装在每个服务中的代理。

*您也可以拥有自己的一套定制网关控制器。*

现在，让我们回到应用程序。我们需要让流量到达这个网关，并将其路由到产品页面所在的主机名。为此，我们需要首先 ***创建一个网关对象*** 。

创建一个入口网关

```
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: app-gateway
spec:
  servers:
  - port:
    number: 80
    name: http
    protocol: HTTP
  hosts:
  - "bookapp.com"
```

如果您有多个入口控制器，也可以将以下代码添加到 spec 部分。

```
selector:
  istio: ingressgateway //Or labels that points toward your services
```

一旦我们有了 YAML 文件，我们可以运行:

```
$ kubectl apply -f ingress-gateway.yaml
```

> 要列出网关，请使用

```
$ kubectl get gateway
```

我们现在有了一个入口。我们如何将流量路由到服务？这就是虚拟服务发挥作用的地方。

# 虚拟服务

> 虚拟服务为来自入口网关进入服务网格的流量定义了一组路由规则。

使用虚拟服务，您可以管理不同版本的服务，并为一个或多个主机名指定流量行为。创建虚拟服务时，Istio 控制平面将配置传递给安装在 sidecar 容器中的所有特使代理。

创建一个虚拟服务

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: app-service
spec:
  hosts:
  - "bookapp.com"
  gateways:
  - app-gateway
  http:
  - match:
    - uri:
        exact: /productpage
    route:
    - destination:
        host: productpage 
        port:
          number: 9080
```

通过这种方式，我们将能够访问产品页面。现在，想象一下，如果我们有多个版本的评论服务，比如 2 个版本 v1 和 v2。你需要将流量分成不同的版本。按照 Kubernetes 的方式，您可以像金丝雀部署一样在部署中添加副本。但这只分流了这些豆荚的流量。

> 如果您只想重定向特定比例的流量，该怎么办？

这就是 Istio 扩展虚拟服务 流量分流特性的地方。

通过编辑如下所示的目标块来修改您的虚拟服务 YAML 文件:

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - "reviews"
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
        weight: 95
    - destination:
      host: reviews 
      subset: v2
      weight: 5
```

***权重参数保证只有 5%的流量流向 v2 版本的应用*** 。太好了！ ***但是，什么是子集？*** 这是目的地规则发挥作用的地方。

# 目的地规则

> 在流量被路由到特定服务后，目标规则应用路由器策略。

让我们在更深的层次上谈论那件事。我们讨论了虚拟服务中的子集。 ***它们是如何定义的，在哪里？***

***子集是在目的规则中定义的。*** 那么，让我们创建一个目的地规则。

创建 dest-rule.yaml

```
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: review-dest
spec:
  host: reviews
  subsets:
  - name: v1
    labels:
      version: v1 //Label mentioned in the pod
  - name: v2
    labels:
      version: v2 //Label mentioned in the pod
```

*默认情况下，Envoy 使用 ROUND_ROBIN 算法进行负载平衡，但这也可以定制。*各种算法选项有**循环、通过、最小连接和随机。**这也可以通过在目的地规则的 **spec** 部分下添加以下代码来提及。

```
trafficPolicy:
  loadBalancer:
    simple: PASSTHROUGH
```

***我们也有特权在子集级别设置流量策略。*** 除了配置负载均衡器，还有 TLS 等其他选项。

> 请访问 [Istio 文档](https://istio.io/latest/docs/)进行参考。

# 故障注入

到目前为止，我们已经配置好了所有的东西，应用程序正在平稳地运行。但是我们如何检查错误处理机制呢？

这就是我们使用**故障注入**的地方。**它帮助我们识别所定义的策略是否有效运行，并且没有过多的限制**。 ***错误可以通过虚拟服务注入。*** 错误有两种:**和* ***中止。****

> *为了引起延迟，在 **spec.http** 部分下添加以下代码*

```
*- fault:
    delay:
      percentage:
        value: 1
      fixedDelay: 10s //By default, 15s*
```

> *为了引起中止，在 **spec.http** 部分下添加以下代码*

```
*- fault:
    abort:
      percentage:
        value: 1
      httpStatus: 400*
```

****Abort 通过拒绝请求并返回错误码的方式模拟错误。****

# *超时设定*

*我们可以在服务级别配置超时失败，这样依赖的服务就不会让请求排队。*

> *要添加超时，请使用虚拟服务 YAML 文件中**规范**部分下的以下代码:*

```
*timeout: 5s*
```

*要测试它，请在依赖于此服务的另一个服务上使用故障注入。*

# *重试次数*

*如果希望虚拟服务再次尝试操作，可以配置重试次数。为此，在虚拟服务 YAML 文件的 **spec.http** 部分使用以下内容:*

```
*retries:
  attempts: 3
  perTryTimeout: 5s*
```

***如果一个服务出现故障，对该服务的所有请求都会堆积起来，造成响应延迟。在这种情况下，此类请求应立即标记为失败。这个过程被称为断路**。*

> *这将使我们能够创建弹性微服务应用，限制故障或任何网络问题的影响。*

> *对于电路中断，在目标规则 YAML 文件的**spec . subsets . traffic policy**部分添加以下代码:*

```
*connectionPool:
  tcp:
  maxConnections: 5*
```

*现在，让我们也来简单了解一下 ***安全是如何在 Istio*** 上工作的。*

# *安全性概述*

*微服务有一定的安全需求。 ***当一个服务与另一个服务通信时，黑客有可能干扰通信并修改请求。这就是所谓的中间人攻击。*** 为了防止这一点，流量需要加密。实施访问控制来限制对每个服务的访问。 *Istio 使用相互 TLS 和细粒度访问策略实施访问控制*。此外，Istio 还提供了对审计日志的支持。我们将在下一篇博客中详细讨论它们。*

*在此阅读第 1 部分: [**Istio 服务网格 101 —零件(1/3)**](https://pilotudesh.medium.com/istio-service-mesh-101-part-1-3-f07a8fedeea8)*

*在此阅读第 3 部分: [**Istio 服务网格 101 —部分(3/3)**](/google-cloud/istio-service-mesh-101-part-3-3-eeda77ab6244)*

*希望这有帮助！祝您愉快！干杯！*