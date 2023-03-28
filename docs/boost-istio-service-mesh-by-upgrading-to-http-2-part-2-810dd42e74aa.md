# 通过升级到 HTTP 2-第 2 部分来增强 Istio 服务网格

> 原文：<https://medium.com/google-cloud/boost-istio-service-mesh-by-upgrading-to-http-2-part-2-810dd42e74aa?source=collection_archive---------1----------------------->

如果你正在寻找提高服务网络效率的方法，那么这篇文章是 2 部分系列文章的第 2 部分。前一篇文章讨论了如何将服务网格中的所有连接从 HTTP 1.1 升级到 HTTP/2。

这篇文章将讨论选项 2:一次升级目的地规则 1 服务。

# 履行

## 选项 2

安装 Istio，然后安装图书信息应用程序。

 [## Bookinfo 应用程序

### 该示例部署了一个由四个独立的微服务组成的示例应用程序，用于演示各种组织结构

istio.io](https://istio.io/latest/docs/examples/bookinfo/) 

产品应用程序(升级前)

```
kubectl logs -f productpage-v1–64794f5db4-fqpgp -c istio-proxy[2021-01-07T10:24:37.568Z] "GET /details/0 **HTTP/1.1**" 200 - "-" 0 178 5 4 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_6) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/13.0.5 Safari/605.1.15" "ee2a4bcb-ed3a-923e-8dc7-663f861ef2c6" "details:9080" "10.1.1.64:9080" **outbound**|9080||details.default.svc.cluster.local 10.1.1.71:43828 10.99.56.191:9080 10.1.1.71:34782 - default[2021-01-07T10:24:37.577Z] "GET /reviews/0 HTTP/1.1" 200 - "-" 0 375 26 26 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_6) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/13.0.5 Safari/605.1.15" "ee2a4bcb-ed3a-923e-8dc7-663f861ef2c6" "reviews:9080" "10.1.1.69:9080" outbound|9080||reviews.default.svc.cluster.local 10.1.1.71:60474 10.101.5.33:9080 10.1.1.71:43712 - default[2021-01-07T10:24:37.564Z] "GET /productpage **HTTP/1.1**" 200 - "-" 0 5179 41 41 "192.168.65.3" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_6) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/13.0.5 Safari/605.1.15" "ee2a4bcb-ed3a-923e-8dc7-663f861ef2c6" "localhost" "127.0.0.1:9080" **inbound**|9080|| 127.0.0.1:40024 10.1.1.71:9080 192.168.65.3:0 outbound_.9080_._.productpage.default.svc.cluster.local default
```

详细信息应用程序(升级前)

```
kubectl logs -f details-v1-5974b67c8-rcnr9 -c istio-proxy[2021-01-07T10:24:37.571Z] "GET /details/0 HTTP/1.1" 200 - "-" 0 178 1 1 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_6) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/13.0.5 Safari/605.1.15" "ee2a4bcb-ed3a-923e-8dc7-663f861ef2c6" "details:9080" "127.0.0.1:9080" inbound|9080|| 127.0.0.1:40030 10.1.1.64:9080 10.1.1.71:43828 outbound_.9080_._.details.default.svc.cluster.local default
```

请注意所有的连接(入站和出站)都在 HTTP 1.1 上。

现在，让我们修改 Details 应用程序的目标规则，将所有传入连接升级到 HTTP/2。

```
details-destinationrules-upgrade.yamlapiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: details
spec:
  host: details
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
 **trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
    connectionPool:
      http:
        h2UpgradePolicy: UPGRADE**
```

这指示 Istio 升级所有针对细节的连接

```
kubectl apply -f /details-destinationrules-upgrade.yaml
```

产品应用程序(升级后)

```
kubectl logs -f productpage-v1–64794f5db4-fqpgp -c istio-proxy[2021-01-07T10:35:01.692Z] "GET /details/0 **HTTP/1.1**" 200 - "-" 0 178 16 16 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_6) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/13.0.5 Safari/605.1.15" "1817e0a2-2fa2-9689-9f97-60d989011c98" "details:9080" "10.1.1.64:9080" **outbound**|9080||**details.default.svc.cluster.local** 10.1.1.71:52792 10.99.56.191:9080 10.1.1.71:43746 - default[2021-01-07T10:35:01.712Z] "GET /reviews/0 HTTP/1.1" 200 - "-" 0 375 52 51 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_6) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/13.0.5 Safari/605.1.15" "1817e0a2-2fa2-9689-9f97-60d989011c98" "reviews:9080" "10.1.1.69:9080" outbound|9080||reviews.default.svc.cluster.local 10.1.1.71:41206 10.101.5.33:9080 10.1.1.71:52676 - default[2021-01-07T10:35:01.688Z] "GET /productpage HTTP/1.1" 200 - "-" 0 5179 78 78 "192.168.65.3" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_6) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/13.0.5 Safari/605.1.15" "1817e0a2-2fa2-9689-9f97-60d989011c98" "localhost" "127.0.0.1:9080" inbound|9080|| 127.0.0.1:48988 10.1.1.71:9080 192.168.65.3:0 outbound_.9080_._.productpage.default.svc.cluster.local default
```

因为产品应用程序使用的是 HTTP 1.1 上的 HTTP/ReST 客户端库，所以出站从 HTTP 1.1 开始，但是 Istio 应该将连接升级到 HTTP/2。让我们检查详细的应用程序日志来验证。

详细信息应用程序(升级后)

```
kubectl logs -f details-v1-5974b67c8-rcnr9 -c istio-proxy[2021-01-07T10:35:01.696Z] "GET /details/0 **HTTP/2**" 200 - "-" 0 178 13 12 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_6) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/13.0.5 Safari/605.1.15" "1817e0a2-2fa2-9689-9f97-60d989011c98" "details:9080" "127.0.0.1:9080" **inbound**|9080|| 127.0.0.1:48994 10.1.1.64:9080 10.1.1.71:52792 outbound_.9080_._.details.default.svc.cluster.local default
```

Istio 已经将调用升级到 HTTP/2！

# 结论

当您不能在全局级别更新配置时，选项 2 是可行的。然而，就像前一篇文章中提到的，我们也应该通过使用使用 HTTP/2 的库来避免升级。