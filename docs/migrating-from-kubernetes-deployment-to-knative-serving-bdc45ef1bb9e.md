# 从 Kubernetes 部署迁移到 Knative 服务

> 原文：<https://medium.com/google-cloud/migrating-from-kubernetes-deployment-to-knative-serving-bdc45ef1bb9e?source=collection_archive---------2----------------------->

当我谈到 Knative 时，我经常会问如何将一个应用程序从 Kubernetes 部署(有时使用 Istio)迁移到 Knative，以及这两种设置之间有什么区别。

首先，你可以用一个 Knative 服务做的所有事情，你可以用一个纯粹的 Kubernetes + Istio 设置和正确的配置来完成。然而，要做对会困难得多。Knative 的全部目的是为您简化和抽象 Kubernetes 和 Istio 的细节。

在这篇博文中，我想用一种不同的方式回答这个问题。我想从一个 Knative 服务开始，并展示如何用 Kubernetes + Istio“艰难地”设置相同的服务。

# 免费服务

在我之前的[帖子](https://meteatamel.wordpress.com/2019/07/24/serverless-grpc-asp-net-core-with-knative/)中，我展示了如何使用 Knative 部署一个自动扩展的、支持 gRPC 的 ASP.NET 核心服务。这是无效服务定义文件`yaml`:

```
apiVersion: serving.knative.dev/v1beta1
kind: Service
metadata:
  name: grpc-greeter
  namespace: default
spec:
  template:
    spec:
      containers:
        - image: docker.io/meteatamel/grpc-greeter:v1
          ports:
          - name: h2c
            containerPort: 8080
```

请注意`yaml`文件的简单性。它有容器图像和端口信息(HTTP2/8080 ),没有其他的了。部署完成后，Knative Serving 负责在 Kubernetes pod 中部署容器的所有细节，通过 Istio ingress 将 pod 暴露给外部世界，并设置自动缩放。

在没有 Knative 的情况下，在 Kubernetes + Istio 集群中部署相同的服务需要什么？让我们来看看。

# Kubernetes 部署

首先，我们需要一个 Kubernetes 部署来将容器封装到一个 pod 中。这是部署`yaml`的样子:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grpc-greeter
spec:
  selector:
      matchLabels:
        app: grpc-greeter
  template:
    metadata:
      labels:
        app: grpc-greeter
    spec:
      containers:
      - name: grpc-greeter
        image: docker.io/meteatamel/grpc-greeter:v1
        ports:
        - name: h2c
          containerPort: 8080
```

这已经比 Knative 服务定义更加冗长。一旦部署，我们将有一个吊舱运行集装箱。

# Kubernetes 服务

下一步是公开 Kubernetes 服务背后的 pod:

```
apiVersion: v1
kind: Service
metadata:
  name: grpc-greeter-service
spec:
  ports:
  - name: http2
    port: 80
    targetPort: h2c
  selector:
    app: grpc-greeter
```

这将暴露端口 80 后面的 pod。然而，在我们在 Istio 中建立网络之前，它还不能公开访问。

# Istio 网关和虚拟服务

在 Istio 集群中，我们需要首先设置一个网关，以便在一个端口/协议上启用外部流量。在我们的例子中，我们的应用程序需要端口 80 上的 HTTP。这是我们需要的网关定义:

```
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: grpc-gateway
spec:
  selector:
    istio: ingressgateway # use istio default controller
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
```

我们现在在端口 80 上启用了流量，但是我们需要将流量映射到我们之前创建的 Kubernetes 服务。这是通过虚拟服务实现的:

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: grpc-virtualservice
spec:
  hosts:
  - "*"
  gateways:
  - grpc-gateway
  http:
  - route:
    - destination:
        host: grpc-greeter-service
```

我们的豆荚终于可以公开了。您可以使用我之前博客中的 [GrpcGreeterClient](https://github.com/meteatamel/knative-tutorial/tree/master/serving/grpc/csharp/GrpcGreeterClient) 指向 Istio 入口网关 IP，您应该会看到我们服务的响应:

```
> dotnet run 
Greeting: Hello GreeterClient 
Press any key to exit...
```

唷！在没有 Knative 的情况下部署一个公共可访问的容器需要很多步骤。我们仍然需要设置 pod 的自动缩放，以获得与 Knative Serving 相同的效果，但我将把它作为一个练习留给读者。

我希望现在清楚了，Knative 使得用更少的配置来部署自动缩放的容器变得更容易。Knative 的高级 API 允许您更多地关注容器中的代码，而不是容器如何部署以及如何使用 Kubernetes 和 Istio 管理其流量的底层细节。

*感谢 Knative 团队的* [*马特·摩尔*](https://twitter.com/mattomata) *给了我这篇博文的灵感。*

*原载于 2019 年 7 月 31 日*[*http://meteatamel.wordpress.com*](https://meteatamel.wordpress.com/2019/07/31/migrating-from-kubernetes-deployment-to-knative-serving/)*。*