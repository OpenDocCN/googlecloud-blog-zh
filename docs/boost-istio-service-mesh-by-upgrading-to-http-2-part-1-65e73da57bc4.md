# 通过升级到 HTTP 2-第 1 部分来增强 Istio 服务网格

> 原文：<https://medium.com/google-cloud/boost-istio-service-mesh-by-upgrading-to-http-2-part-1-65e73da57bc4?source=collection_archive---------0----------------------->

如果你正在使用 Istio，并且你正在寻找简单的方法来加速服务网格，这是给你的帖子。

![](img/153ed02d1ab04503e2eb462a50dec15b.png)

关于 HTTP 2 比 HTTP 1.1 有多快，效率有多高，已经有很多文章写了。你可以在这里阅读[，在这里](https://www.thewebmaster.com/hosting/2015/dec/14/what-is-http2-and-how-does-it-compare-to-http1-1/)阅读[。](https://www.tunetheweb.com/blog/http-versus-https-versus-http2/)

![](img/2ac82f10f30e6d3eb0d97915f3785907.png)

我看到的大多数服务网格实现仍然使用古老的 HTTP 1.1，并且没有将其连接升级到 HTTP/2。

然而，一个网格是由许多应用程序组成的。其中一些应用程序可能使用 HTTP 1.1 或 HTTP/2。以我的经验来看，大部分 app(如果不是全部的话)还是在 HTTP 1.1 上。升级网格上的所有微服务可能需要一些时间和精力，即使风险可以忽略不计。

**如果我们将所有的网状网流量升级到 HTTP/2，我们应该会立即得到提升，而不会冒任何额外的风险。**

我们可以通过简单地检查 istio-proxy 日志来验证我们的服务是在 HTTP1.1 还是 HTTP/2 上。

为了演示这一点，我安装了 Istio 附带的图书信息应用程序。

[](https://istio.io/latest/docs/setup/getting-started/) [## 入门指南

### 设置入口端口:$ export INGRESS _ PORT = $(ku bectl-n istio-system get service istio-Ingres gateway-o…

istio.io](https://istio.io/latest/docs/setup/getting-started/) 

从浏览器导航到产品页面后，打开产品应用程序和下游应用程序的 istio-proxy 日志-详细信息。

产品应用程序(升级前):

```
kubectl logs -f productpage-v1-64794f5db4-fqpgp -c istio-proxy[2021-01-06T13:44:45.994Z] "GET /details/0 **HTTP/1.1**" 200 - "-" 0 178 2 1 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_6) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/13.0.5 Safari/605.1.15" "9a425a61-b88f-9b49-858d-252ed262e549" "details:9080" "10.1.1.31:9080" **outbound**|9080||details.default.svc.cluster.local 10.1.1.36:59852 10.99.56.191:9080 10.1.1.36:42486 - default[2021-01-06T13:44:46.000Z] "GET /reviews/0 **HTTP/1.1**" 200 - "-" 0 379 15 15 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_6) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/13.0.5 Safari/605.1.15" "9a425a61-b88f-9b49-858d-252ed262e549" "reviews:9080" "10.1.1.34:9080" **outbound**|9080||reviews.default.svc.cluster.local 10.1.1.36:56232 10.101.5.33:9080 10.1.1.36:48422 - default[2021-01-06T13:44:45.990Z] "GET /productpage **HTTP/1.1**" 200 - "-" 0 5183 28 28 "192.168.65.3" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_6) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/13.0.5 Safari/605.1.15" "9a425a61-b88f-9b49-858d-252ed262e549" "localhost" "127.0.0.1:9080" **inbound**|9080|| 127.0.0.1:50076 10.1.1.36:9080 192.168.65.3:0 outbound_.9080_._.productpage.default.svc.cluster.local default
```

请注意，**入站**和**出站**连接的默认 TCP 协议都在 HTTP 1.1 上。

让我们检查一个下游应用程序的日志。

详细信息应用程序(升级前):

```
kubectl logs -f details-v1-5974b67c8-rcnr9 -c istio-proxy[2021-01-06T13:44:46.171Z] "GET /details/0 **HTTP/1.1**" 200 - "-" 0 178 1 1 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_6) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/13.0.5 Safari/605.1.15" "c00e80f6-bef2-9843-b4b9-eb1112ae3887" "details:9080" "127.0.0.1:9080" **inbound**|9080|| 127.0.0.1:49528 10.1.1.31:9080 10.1.1.36:59788 outbound_.9080_._.details.default.svc.cluster.local default
```

我们现在要做的就是让 Istio 知道我们要升级所有的 HTTP 1.1。HTTP 2 的调用。

注意:还有一个重要的秘密步骤，我稍后会透露。

# 履行

有两种方法可以解决这个问题:

选项 1:升级全局网格配置，将所有 HTTP 1.1 连接升级到 HTTP/2

选项 2:一次升级目标 rules 1 服务

在本帖中，我们将只探讨选项 1。

## 选项 1

这很容易实现。我们必须为“istio-system”名称空间中的“istio”配置图打补丁

```
**h2UpgradePolicy: UPGRADE**
```

描述 istio-system 命名空间中的当前 istio 配置映射。

```
kubectl describe configmap istio -n istio-system
```

记下数据/网格下的所有值。

创建 configmap-patch.yaml

将数据/网格下的上述值添加到下面的 yaml 文件+ **h2UpgradePolicy: UPGRADE。**

```
data:
  mesh: |-
    **h2UpgradePolicy: UPGRADE**
    accessLogFile: /dev/stdout
    defaultConfig:
      discoveryAddress: istiod.istio-system.svc:15012
      proxyMetadata:
        DNS_AGENT: ""
      tracing:
        zipkin:
          address: zipkin.istio-system:9411
    enablePrometheusMerge: true
    rootNamespace: istio-system
    trustDomain: cluster.local
```

运行 kubectl patch 命令。

```
kubectl patch configmap istio -n istio-system --patch "$(cat /configmap-patch.yaml)"
```

再次描述 istio-system 名称空间中的 istio configmap，并验证除了新添加的值(**H2 UPGRADE policy:UPGRADE)**之外，旧值是否存在。

```
kubectl describe configmap istio -n istio-system
```

注意:我们的意图是添加**H2 升级策略:升级**并避免修补现有配置。

就是这样！

像以前一样点击产品页面上的刷新。

产品应用程序(升级后):

```
kubectl logs -f productpage-v1-64794f5db4-fqpgp -c istio-proxy[2021-01-06T13:46:07.929Z] "GET /details/0 **HTTP/1.1**" 200 - "-" 0 178 6 6 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_6) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/13.0.5 Safari/605.1.15" "e1a591e7-2624-9616-b7b6-45d40346bcb4" "details:9080" "10.1.1.31:9080" **outbound**|9080||**details.default.svc.cluster.local** 10.1.1.36:33304 10.99.56.191:9080 10.1.1.36:43680 - default[2021-01-06T13:46:07.940Z] "GET /reviews/0 **HTTP/1.1**" 200 - "-" 0 375 26 26 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_6) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/13.0.5 Safari/605.1.15" "e1a591e7-2624-9616-b7b6-45d40346bcb4" "reviews:9080" "10.1.1.35:9080" **outbound**|9080||**reviews.default.svc.cluster.local** 10.1.1.36:46090 10.101.5.33:9080 10.1.1.36:49620 - default[2021-01-06T13:46:07.925Z] "GET /productpage **HTTP/2**" 200 - "-" 0 5179 45 44 "192.168.65.3" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_6) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/13.0.5 Safari/605.1.15" "e1a591e7-2624-9616-b7b6-45d40346bcb4" "localhost" "127.0.0.1:9080" **inbound**|9080|| 127.0.0.1:51270 10.1.1.36:9080 192.168.65.3:0 outbound_.9080_._.productpage.default.svc.cluster.local default
```

等等！为什么有些**出站**日志还是说协议是 HTTP 1.1，而**入站**日志是 HTTP/2？

发生这种情况是因为示例中 Product App 使用的 HTTP 客户端库使用 HTTP 1.1 协议。我们需要使用支持 HTTP/2 的库(参见下面的**重要提示**)。

详细信息应用程序(升级后):

```
kubectl logs -f details-v1-5974b67c8-rcnr9 -c istio-proxy[2021-01-06T13:46:07.933Z] "GET /details/0 **HTTP/2**" 200 - "-" 0 178 2 1 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_6) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/13.0.5 Safari/605.1.15" "e1a591e7-2624-9616-b7b6-45d40346bcb4" "details:9080" "127.0.0.1:9080" **inbound**|9080|| 127.0.0.1:51276 10.1.1.31:9080 10.1.1.36:33304 outbound_.9080_._.details.default.svc.cluster.local default
```

源自产品应用程序的请求位于 HTTP 1.1 上(请参见升级后的产品应用程序日志)；Istio 现在将其升级到 HTTP/2，并将其传递给 Details App。

## 重要说明

*记住，我们正在指示 Istio 将连接从 HTTP 1.1 升级到 HTTP/2。它不能神奇地让部署的应用程序中的 HTTP/ReST 客户端库使用 HTTP/2。我们需要升级我们的应用程序以使用 HTTP/2。*

例如，Spring Boot 支持 HTTP/2，这取决于所使用的 web 服务器。要完全迁移到 HTTP/2，我们需要指示 Spring Boot 迁移到 HTTP/2。

 [## Spring Boot 参考文献

### 包 com . example . my application；导入 org . spring framework . boot . spring application；导入…

docs.spring.io](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#howto-configure-http2) 

这里有一个有用的帖子:

 [## 在您的 Spring Boot 应用程序中使用 HTTP/2

### 对于本教程，我假设您有一个基于 Unix 的环境，在这个环境中您可以运行 openssl 和…

byte27.com](https://byte27.com/2020/02/03/using-http-2-in-your-spring-boot-application/) 

# 结论

Istio 将我们的连接升级到一个更高效的协议真是太好了。但是，应该通过将网格上的所有应用程序移动到 HTTP/2 来避免升级。考虑到 HTTP/2 已经存在了很长时间，无论涉及到什么技术，这都不会太难。

风险极小，而收益巨大。