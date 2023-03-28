# 谷歌云平台上的基础设施代码:负载平衡器

> 原文：<https://medium.com/google-cloud/infrastructure-as-code-on-google-cloud-platform-load-balancers-22591561a2ec?source=collection_archive---------1----------------------->

这是我使用谷歌部署管理器(GDM)所学到的知识的系列文章的第 3 部分

距离我的上一篇帖子已经有一段时间了，我可以把它归咎于假期或我的健康或家庭，但我不会。真正的原因是拖延症。

当我开始这次旅程时，我一点也不知道在 GDM 建造负载平衡器会如此困难。事实证明，以这种方式构建时，并不存在负载平衡器这样的庞然大物。Google Cloud Console 会带你完成一个很好的设置过程，但是实际上还有更多的事情要做。在我们构建 HTTP(S)负载平衡器的过程中，我们将学习如何构建各种各样漂亮的小东西。

首先(也是最简单的)先做。没有 IP 地址，负载平衡器就没有任何价值，所以让我们创建一个 IP 地址。这可能是你能创建的最简单的资源。不要忘记将它添加到您的`outputs:`部分，以便了解该 IP 地址实际上是什么。

```
resources:
###################################
# IP addresses
###################################
- name: dav-ipaddress
  type: compute.v1.globalAddressoutputs:
- name: publicIP
  value: $(ref.dav-ipaddress.address)
```

这似乎有点违反直觉，但现在我们需要建立一个健康检查。负载平衡器将使用它来确定后端服务何时是健康的。

```
###################################
# Health Checks
###################################
- name: dav-health-check
  type: compute.v1.httpHealthCheck
  properties:
    requestPath: /robots.txt
    port: 80
    checkIntervalSec: 5
    timeoutSec: 5
    unhealthyThreshold: 2
    healthyThreshold: 2
```

现在让我们创建将托管我们的应用程序的后端服务。后端服务指的是一个 Google 计算引擎实例组，为了今天的目的，让我们假设它已经被创建了。在您的后端定义中，您可以指定`port`和`portName`，但是`port`已经被弃用，取而代之的是`portName`，它指的是实例组上的一个命名端口，我们也假设它已经被创建。请注意，我对两个后端使用了相同的健康检查——只要所有的设置在两个实例组上都有效。

```
###################################
# Backend Services
###################################
- name: dav-backend-svc-1
  type: compute.v1.backendService
  properties:
    backends:
    - group: 'https://www.googleapis.com/compute/v1/projects/dav-project/zones/us-central1-b/instanceGroups/dav-instance-group-1'
      maxUtilization: 0.8
    healthChecks:
    - $(ref.dav-health-check.selfLink)
    portName: dav-named-port-1
- name: dav-backend-svc-2
  type: compute.v1.backendService
  properties:
    backends:
    - group: 'https://www.googleapis.com/compute/v1/projects/dav-project/zones/us-east1-b/instanceGroups/dav-instance-group-2'
      maxUtilization: 0.8
    healthChecks:
    - $(ref.dav-health-check.selfLink)
    portName: dav-named-port-2
```

现在，我们需要构建一个 URL 映射，将请求路由到正确的后端服务。在下面的示例中，路径匹配/api 将路由到 dav-backend-svc-2，所有其他请求将路由到 dav-backend-svc-1。

```
###################################
# URL Maps
###################################
- name: dav-url-map
  type: compute.v1.urlMap
  properties:
    defaultService: $(ref.dav-backend-svc-1.selfLink)
    hostRules:
    - hosts:
      - dav.example.com
      pathMatcher: path-matcher-1
    pathMatchers:
    - defaultService: $(ref.dav-backend-svc-1.selfLink)
      name: path-matcher-1
      pathRules:
      - paths:
        - /
        service: $(ref.dav-backend-svc-1.selfLink)
        - /api
        service: $(ref.dav-backend-svc-2.selfLink)
```

现在我们将设置 HTTP(S)代理。使用下面的配置，负载平衡器会将 HTTP 或 HTTPS 请求路由到适当的后端。SSL 证书(我们假设在 GCP 已经存在)将由负载平衡器托管，因此您不需要在所有实例上托管。负载均衡器会将`x-forwarded-proto` http 头添加到所有请求中，如果您愿意，您可以配置 nginx 或 apache(或者您选择的 http 服务器)来强制基于该头的 SSL。如果您决定重定向，请记住不要将您的健康检查路径重定向到 HTTPS！

```
###################################
# HTTP Proxies
###################################
- name: dav-http-proxy
  type: compute.v1.targetHttpProxy
  properties:
    urlMap: $(ref.dav-url-map.selfLink)
- name: dav-https-proxy
  type: compute.v1.targetHttpsProxy
  properties:
    urlMap: $(ref.dav-url-map.selfLink)
    sslCertificates:
    - 'https://www.googleapis.com/compute/v1/projects/dav-project/global/sslCertificates/star-dav-example-com'
```

最后，我们需要设置网络转发规则，将流量路由到我们创建的静态 IP(还记得开始时的方法吗？)到适当的 HTTP(S)代理。

```
###################################
# Network Forwarding Rules
###################################
- name: dav-http-forwardingrule
  type: compute.v1.globalForwardingRule
  properties:
    target: $(ref.dav-http-proxy.selfLink)
    IPAddress: $(ref.dav-ipaddress.address)
    IPProtocol: TCP
    portRange: 80-80
- name: dav-https-forwardingrule
  type: compute.v1.globalForwardingRule
  properties:
    target: $(ref.dav-https-proxy.selfLink)
    IPAddress: $(ref.dav-ipaddress.address)
    IPProtocol: TCP
    portRange: 443-443
```

耶！我们完了！等等——你说这对你没用？就在我们以为已经完成的时候，还有一件事要做。我们需要防火墙规则来允许所有这些流量在项目中流动。在下面的例子中，`ports:`指的是后端服务上指定端口的端口号。下面`sourceRanges:`中的条目是 Google 为其所有负载平衡器使用的 CIDR 地址范围，因此您可以使用下面的值。

```
###################################
# Firewall Rules
###################################
- name: allow-port-80
  type: compute.v1.firewall
  properties:
    allowed:
      - IPProtocol: TCP
        ports: [ 80 ]
    sourceRanges: [ 130.211.0.0/22 ]
```

现在我们结束了。应用配置后，所有到 dav.example.com 的 HTTP(S)流量将根据请求的 URL 路由到我们的 2 个后端之一。因此，您可以将整个系统视为一个单独的配置，这里所有的配置都放在一起。

```
resources:
###################################
# IP addresses
###################################
- name: dav-ipaddress
  type: compute.v1.globalAddress###################################
# Health Checks
###################################
- name: dav-health-check
  type: compute.v1.httpHealthCheck
  properties:
    requestPath: /robots.txt
    port: 80
    checkIntervalSec: 5
    timeoutSec: 5
    unhealthyThreshold: 2
    healthyThreshold: 2###################################
# Backend Services
###################################
- name: dav-backend-svc-1
  type: compute.v1.backendService
  properties:
    backends:
    - group: 'https://www.googleapis.com/compute/v1/projects/dav-project/zones/us-central1-b/instanceGroups/dav-instance-group-1'
      maxUtilization: 0.8
    healthChecks:
    - $(ref.dav-health-check.selfLink)
    portName: dav-named-port-1
- name: dav-backend-svc-2
  type: compute.v1.backendService
  properties:
    backends:
    - group: 'https://www.googleapis.com/compute/v1/projects/dav-project/zones/us-east1-b/instanceGroups/dav-instance-group-2'
      maxUtilization: 0.8
    healthChecks:
    - $(ref.dav-health-check.selfLink)
    portName: dav-named-port-2###################################
# URL Maps
###################################
- name: dav-url-map
  type: compute.v1.urlMap
  properties:
    defaultService: $(ref.dav-backend-svc-1.selfLink)
    hostRules:
    - hosts:
      - dav.example.com
      pathMatcher: path-matcher-1
    pathMatchers:
    - defaultService: $(ref.dav-backend-svc-1.selfLink)
      name: path-matcher-1
      pathRules:
      - paths:
        - /
        service: $(ref.dav-backend-svc-1.selfLink)
        - /api
        service: $(ref.dav-backend-svc-2.selfLink)###################################
# HTTP Proxies
###################################
- name: dav-http-proxy
  type: compute.v1.targetHttpProxy
  properties:
    urlMap: $(ref.dav-url-map.selfLink)
- name: dav-https-proxy
  type: compute.v1.targetHttpsProxy
  properties:
    urlMap: $(ref.dav-url-map.selfLink)
    sslCertificates:
    - 'https://www.googleapis.com/compute/v1/projects/dav-project/global/sslCertificates/star-dav-example-com'###################################
# Network Forwarding Rules
###################################
- name: dav-http-forwardingrule
  type: compute.v1.globalForwardingRule
  properties:
    target: $(ref.dav-http-proxy.selfLink)
    IPAddress: $(ref.dav-ipaddress.address)
    IPProtocol: TCP
    portRange: 80-80
- name: dav-https-forwardingrule
  type: compute.v1.globalForwardingRule
  properties:
    target: $(ref.dav-https-proxy.selfLink)
    IPAddress: $(ref.dav-ipaddress.address)
    IPProtocol: TCP
    portRange: 443-443###################################
# Firewall Rules
###################################
- name: allow-port-80
  type: compute.v1.firewall
  properties:
    allowed:
      - IPProtocol: TCP
        ports: [ 80 ]
    sourceRanges: [ 130.211.0.0/22 ]outputs:
- name: publicIP
  value: $(ref.dav-ipaddress.address)
```

您可能已经注意到，我对在这个配置中创建的任何资源都使用了引用。这是一个很好的实践，因为它将 A)在您的配置中创建隐式的依赖关系，这样资源就不会在它们所依赖的资源之前被创建，B)您不需要尝试和预测那些值或者分多个步骤运行您的构建。

如果您设法做到了这一步，那么您可能会非常致力于使用部署管理器。如果是这种情况，请继续关注本系列的下一期文章。在过去的一周左右的时间里，我一直在学习如何使用 GDM 模板，一旦我有了进一步的进展，我将会发布我的经历。