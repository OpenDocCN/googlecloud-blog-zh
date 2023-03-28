# 谷歌云平台:将 HTTP 流量重定向到 HTTPS

> 原文：<https://medium.com/google-cloud/google-cloud-platform-redirecting-http-traffic-to-https-f63c1d7dbc6d?source=collection_archive---------0----------------------->

我们正在将我们在 Kubernetes 上的所有项目转移到使用 GCE(谷歌计算引擎)ingress 而不是 Nginx ingress，以便与整个谷歌云生态系统更紧密地集成。

但不幸的是，目前不可能将 HTTP 流量直接重定向到 HTTPS😦。是的，你没听错。这个看似基本的功能并不直接受支持。

如果我们阅读 Github 上的文档([https://github.com/kubernetes/ingress-gce](https://github.com/kubernetes/ingress-gce))。

要将流量从“80”重定向到“443 ”,您需要检查 GCE L7 插入的“x-forwarded-proto”报头，因为入口不支持重定向规则。在 Nginx 中，这很简单，只需在配置中添加以下几行:

```
# Replace '_' with your hostname.server_name _;if ($http_x_forwarded_proto = "http") {return 301 https://$host$request_uri;}
```

可悲的是，我们需要手动完成这项工作，但幸运的是，我们仍然有 Nginx 来拯救我们。