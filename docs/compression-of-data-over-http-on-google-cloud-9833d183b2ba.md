# Google Cloud 上基于 HTTP 的数据压缩

> 原文：<https://medium.com/google-cloud/compression-of-data-over-http-on-google-cloud-9833d183b2ba?source=collection_archive---------0----------------------->

# 背景

云用户有时会想，是否有比 gzip 更有效的方法来压缩 web 上的文本和 JSON 数据。首先，我们来看看用 gzip 怎么做。

如果一个 App Engine HTTP 客户端发送一个值为 gzip 的 Accept-Encoding 头，那么它将压缩返回的内容( [documentation](https://cloud.google.com/appengine/docs/standard/go/how-requests-are-handled#response_compression) )。目前，gzip 是 App Engine 唯一直接支持的选项。这类似于 NGINX 的开箱即用。HTTP/2 用 [HPACK](https://httpwg.org/specs/rfc7541.html) 替换了 gzip，因为使用 gzip ( [doc](https://http2.github.io/faq/#why-hpack) )的压缩 HTTP 流发现了安全漏洞。然而，这只是针对 HTTP 头。QUIC 利用一种类似于 HTTP/2 的方法来处理报头。

要验证您的服务器是否正在返回 gzip 内容，请使用 Curl 命令，如

```
curl -H "Accept-Encoding: gzip" -I [http://localhost:8080/test.json](http://localhost:8080/test.json)
```

假设您的服务器在本地主机上运行。您应该会看到类似这样的输出

```
Content-Encoding: gzip
```

这个要点给出了设置 Nginx 服务器来压缩 JSON 文件的说明。默认情况下，HTML 文件应该被压缩。

# 布罗特利

[Brotli](https://tools.ietf.org/html/rfc7932) (br)可以提供比 gzip 大得多的压缩，但压缩效率可能较低([简单比较](https://www.opencpu.org/posts/brotli-benchmarks/))。因此，它可能比动态创建的应用程序数据更适合经常读取的文件。但是，Brotli 也可以控制压缩级别，这可以为您的应用程序提供最佳平衡。Google 开源了 Brotli ( [github 项目](https://github.com/google/brotli))，列出了更详细的基准。Brotli 由 [Cronet](https://chromium.googlesource.com/chromium/src/+/master/components/cronet) 网络客户端库支持，适合在移动应用中使用。在 Cronet 中，Brotli 默认不启用，但是可以通过调用 [enableBrotli(true)](https://chromium.googlesource.com/chromium/src/+/lkgr/components/cronet/android/api/src/org/chromium/net/CronetEngine.java#175) 来启用。

Brotli 也受到主流浏览器的支持。Mozilla 文档给出了正确的 HTTP 头来使用

```
Accept-Encoding: br
```

虽然，GCP 并不直接支持 Brotli，但 Brotli 的一个开源 Nginx 插件([这里](https://github.com/google/ngx_brotli))可以与 Docker 镜像([这里](https://github.com/fholzer/docker-nginx-brotli))一起使用。这对于带有定制容器的 Flex 和 Kubernetes 来说应该足够了。在[本要点](https://gist.github.com/alexamies/71f17f878a1efb764e631920ac3b7076)中提供了使用和验证的简单说明。在 Nginx 服务器配置完成后，您应该能够使用以下命令验证它是否返回 Brotli 压缩数据

```
curl -H "Accept-Encoding: br" -I [http://localhost:8080/test.json](http://localhost:8080/test.json)
...
Content-Encoding: br
```