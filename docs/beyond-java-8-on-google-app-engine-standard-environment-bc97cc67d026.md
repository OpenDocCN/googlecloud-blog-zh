# 超越 Google App Engine 上的 Java 8(标准环境)

> 原文：<https://medium.com/google-cloud/beyond-java-8-on-google-app-engine-standard-environment-bc97cc67d026?source=collection_archive---------2----------------------->

最近宣布的谷歌应用引擎标准环境对 Java 8 的支持是一个重要的里程碑。然而，坦白地说，这有点晚了，更重要的是，它可能会掩盖对底层基础设施所做的一些重要更改，从而带来新的开发人员体验(DX)。

这里的部分新内容是 Google 对开发者运行时实现是透明的:OpenJDK 8 和 Jetty 9。虽然 App Engine 的优势一直是提供本地 SDK 开发环境，但生产实施细节使其更有价值。

App Engine“标准环境”不依赖于 VM 技术，这个 Java 8 运行时只是第一个利用基础新基础设施的运行时。正如测试版用户所指出的，您很有可能会看到更好的性能和更低的资源消耗。但是当然 YMMV。

作为一个真正的无服务器解决方案，App Engine 提供隐式服务，如 CSS、JavaScript 和其他静态内容的全局缓存。它还支持针对多个已部署版本的[流量分流](https://cloud.google.com/appengine/docs/standard/java/splitting-traffic)，例如，这可以用于金丝雀部署到 Java 8 的迁移。这仍然是我最喜欢的产品特性之一。

虽然这些实现细节值得注意，但更重要的是它们带来了大规模的基础设施改进和新的开发人员体验。

如果你仔细注意了你已经读过的公告“*利用 OpenJDK 8 (…)必须提供的一切*”。这实际上意味着，作为一名开发人员，您现在可以利用任何 Java 类——不再仅仅是一系列受欢迎的 API——以及您或您最喜欢的库愿意使用的任何线程模型。这与 PaaS 时代大相径庭，当时 PaaS 意味着开发人员必须通过处理强制遵守 API 约束的缺点来获得自动扩展的优势。

无论您是希望完全使用微服务，还是简单地将开发分成多个部署单元，App Engine [内置服务和版本控制](https://cloud.google.com/appengine/docs/standard/java/microservices-on-app-engine)(已经推出一段时间)仍然是一组灵活而非常强大的功能。

gRPC 作为一种微服务通信技术[越来越受欢迎](https://trends.google.com/trends/explore?date=2015-04-05%202017-09-27&q=grpc,apache%20thrift),但在无状态是常态的云环境中使用它可能会很棘手。在这个版本中，完全支持的出站 gRPC 服务是修改后的 DX 的另一部分，通过 Maven 或 Gradle 支持谷歌[云 Java 库](http://googlecloudplatform.github.io/google-cloud-java/latest/index.html)。

随着开发者体验的改善，是时候以开放的态度重新审视 App Engine 了。如果你强烈感觉 PaaS 不适合你，考虑 App Engine 成为无服务器的，因为它是！