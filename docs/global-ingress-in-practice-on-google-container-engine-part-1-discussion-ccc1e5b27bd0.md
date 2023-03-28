# Google 容器引擎的全球入口实践—第 1 部分:讨论

> 原文：<https://medium.com/google-cloud/global-ingress-in-practice-on-google-container-engine-part-1-discussion-ccc1e5b27bd0?source=collection_archive---------0----------------------->

在本文中，我将介绍在使用 GCE ingress 控制器将一个真实的应用程序部署到一个全局联合集群时，我所面临的各种挑战和我找到的解决方案。我已经在另一篇文章中讲述了如何分三步建立全球 Kubernetes。本文将集中讨论一旦设置好如何使用它。在第 1 部分中，我将讨论这些概念，在第 2 部分[中，我们将使用真实代码进行端到端部署。](/@cgrant/global-ingress-in-practice-on-google-container-engine-part-2-demo-cf587765702)

内容:
-简单示例
-变化
-全局 IP
-注释
-健康检查
-节点端口类型和端口
-路径上下文
-集群平衡

让多个部署的服务在一个域名下响应是大型应用程序中的常见做法。通过 Kubernetes，您可以使用 ClusterIPs、NodePort 和 LoadBalancers 将部署作为独立的服务公开。您还可以使用入口资源将多个服务公开为单个虚拟实体。

理论上，Ingress 资源简单易用，但在实践中，学习曲线可能会更陡。在本文中，我们将回顾创建入口资源的基础知识，以及您在现实生活中会遇到的一些怪癖。

# 简单的例子

这个过程涉及三个主要资源:部署、it 服务和入口本身。让我们回顾一个简单的 Hello World 入口。

从 Kubernetes 关于入口资源的文档中，我们可以看到许多关键元素

从文档导入示例 Yaml

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test
  annotations:
    ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - path: /foo
        backend:
          serviceName: s1
          servicePort: 80
      - path: /bar
        backend:
          serviceName: s2
          servicePort: 80
```

这里只介绍一下，任何对主机 foo.bar.com 的请求都将由该块中包含的规则处理，`foo.bar.com/foo`将路由到端口`80`上的服务`s1`，对`foo.bar.com/bar`的请求将路由到端口`80`上的服务`s2`。

# 文件中未明确列出的变更

**所有主机**

这里有一个例子

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test
  annotations:
    ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - http:
      paths:
      - path: /foo
        backend:
          serviceName: s1
          servicePort: 80
      - path: /bar
        backend:
          serviceName: s2
          servicePort: 80
```

**默认后端**
文档讨论了一个[单一服务入口](https://kubernetes.io/docs/concepts/services-networking/ingress/#single-service-ingress)选项，没有规则。您可以将它与路径规则结合起来，用附加规则定义您自己的默认后端。这里有一个例子。

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test
  annotations:
    ingress.kubernetes.io/rewrite-target: /
spec:
  backend:
    serviceName: testsvc
    servicePort: 80
  rules:
  - http:
      paths:
      - path: /foo
        backend:
          serviceName: s1
          servicePort: 80
      - path: /bar
        backend:
          serviceName: s2
          servicePort: 80
```

在这里，对这个入口的任何请求用`/foo`转到服务`s1`，对`/bar`的任何请求将路由到`s2`，所有其他请求将路由到 `testsvc`

> 需要注意的是，如果你没有定义一个默认的后端，Kubernetes 会在后台为你创建一个。此外，为您创建的后端仅存在于一个集群中，稍后查看在 GCP 创建的负载平衡器后端时，您会看到这一点。

# 全球知识产权

这是全球入口的另一个重要点。你需要明确地创建和使用谷歌的全球 IP。创建的默认临时 IP 仅是区域性的，无法支持来自不同地区的后端服务。

> 我花了很长时间在这上面，不要错过它

从命令行创建一个全局 IP

`gcloud compute addresses create ingress-ip --global`

然后在 ingress.yaml 中将其作为注释引用，如下所示:

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test
  annotations:
    kubernetes.io/ingress.global-static-ip-name: ingress-ip      
    ingress.kubernetes.io/rewrite-target: /
spec:
  backend:
    serviceName: testsvc
    servicePort: 80
  rules:
  - http:
      paths:
      - path: /foo
        backend:
          serviceName: s1
          servicePort: 80
      - path: /bar
        backend:
          serviceName: s2
          servicePort: 80
```

# 入口注释

在入口定义的元数据部分，可以提供各种注释来帮助 kubernetes 更好地理解您的意图。

**入口控制器**
根据您部署 kubernetes 的位置和方式，您可以选择在您提供的入口 yaml 定义上实际执行什么。许多文档将 nginx 称为入口控制器，但是对于这个例子，我将展示如何使用本地 GCE。

虽然不是必需的，但是添加一个注释来指出您打算使用哪个控制器是一个很好的做法。例如，如果在给定的环境中有多个选项，这是很有帮助的

因为我想为这个入口控制器使用 GCE，所以我将使用`kubernetes.io/ingress.class: “gce”`注释显式地指定它

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test
  annotations:
     kubernetes.io/ingress.class: "gce"
    ingress.kubernetes.io/rewrite-target: /
spec:
  backend:
    serviceName: testsvc
    servicePort: 80
  rules:
  - http:
      paths:
      - path: /foo
        backend:
          serviceName: s1
          servicePort: 80
      - path: /bar
        backend:
          serviceName: s2
          servicePort: 80
```

# 健康检查

默认情况下，ingress 将通过 ping 您的服务的根来设置 loadblancer 健康检查。如果您选择不在服务中发送“/”，了解这一点很重要。为确保您的应用程序正确注册其健康状态，请提供一个“/”路径或[根据您的需求配置活性和就绪探测器](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/)。

为了简单起见，我最终只在应用程序中留下了根上下文

# 节点端口

这是另一个在完成后完全有意义的项目。首先，使用 GCE 入口控制器需要一个通过 NodePort 公开的服务，它不能仅仅是 ClusterIP。其次，当在联合集群中部署时，容器的节点端口需要在所有集群中相同，因此我们需要显式地定义它。默认情况下，每个容器都会为 NodePort 类型提供一个随机端口，但是在联邦模型中，我们需要它们是相同的，这样健康检查才是准确的。一旦启动并运行，您将看到运行状况检查在所有节点上为您的应用程序轮询同一个端口。

为了定义这一点，我们将在部署的服务定义中指定我们想要的确切值。这里有一个例子

```
apiVersion: v1
kind: Service
metadata:
  name: s1
  labels:
    app: app1
spec:
  type: NodePort
  ports:
  - port: 80
    nodePort: 30041
  selector:
    app: app1
```

您需要为每个服务定义不同的节点端口，这样就不会发生冲突

# **路径**上下文

这可能是我遇到的最烦人的问题，只有在一个真正的应用中才会出现。到目前为止，所有的演示和 hello world 应用程序都运行良好。`/foo`到`svc1`的路线，`/bar`到`svc2`的路线。然而，当我部署一个真正的应用程序时，事情就不那么清楚了。我将获得我的服务的主页，但其他一切都将恢复到默认的负载平衡器。开始发现有一个【入口的怪癖】([https://github.com/kubernetes/contrib/issues/885](https://github.com/kubernetes/contrib/issues/885))Nginx 和 GCE 入口控制器的工作方式不一样。

基本上在 Nginx 上，`/foo`正在寻找前缀为`/foo`的任何东西，gce 控制器将它视为显式映射。要解决这个问题，我们需要添加*映射到入口 yaml 中的规则路径，如下所示:

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test
  annotations:
     kubernetes.io/ingress.class: "gce"
    ## ingress.kubernetes.io/rewrite-target: /
spec:
  backend:
    serviceName: testsvc
    servicePort: 80
  rules:
  - http:
      paths:
      - path: /foo
        backend:
          serviceName: s1
          servicePort: 80
      - path: /foo/*
        backend:
          serviceName: s1
          servicePort: 80
      - path: /bar
        backend:
          serviceName: s2
          servicePort: 80
      - path: /bar/*
        backend:
          serviceName: s2
          servicePort: 80
```

通过添加额外的映射，对`/foo`或`/foo/baz/bar`的任何请求将正确地路由到服务`s1`，对`/bar`或`/bar/baz/foo`的请求将路由到服务`s2`

# 集群平衡

在大多数情况下，kubernetes 会尝试平衡集群，这样应用程序会均匀地分布在集群中，但是您可以使用如下的`federation.kubernetes.io/deployment-preferences:`注释来反映您的意图

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: app1
spec:
  replicas: 4
  template:
    metadata:
      annotations:
        federation.kubernetes.io/deployment-preferences: |
          {
            "rebalance": true,
            "clusters": {
              "east-cluster": {
                  "minReplicas": 1
              },
              "west-cluster": {
                  "minReplicas": 1
              }
            }
          }
      labels:
        app: app1
    spec:
      containers:
        - name: app1
          image: myrepo/appi:v7
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 100m
              memory: 100Mi
```

上面的注释要求 kubernetes 重新平衡并在东部和西部各保留一个副本。

# 结论

Kubernetes 入口控制器是一个强大的工具。几乎没有额外的洞察力，他们也成为一个简单的管理资源。利用 GCE 控制器类型，您可以在 Google Container Engine 上快速轻松地实现 ingress，而不需要额外的资源。

我希望这能有所帮助，请务必查看[设置指南](/google-cloud/global-kubernetes-in-3-steps-on-gcp-8a3585ec8547)和[端到端演示](/@cgrant/global-ingress-in-practice-on-google-container-engine-part-2-demo-cf587765702)。