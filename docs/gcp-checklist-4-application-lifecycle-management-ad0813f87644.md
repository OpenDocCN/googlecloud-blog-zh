# GCP 清单 4 —应用生命周期管理

> 原文：<https://medium.com/google-cloud/gcp-checklist-4-application-lifecycle-management-ad0813f87644?source=collection_archive---------1----------------------->

部署到 GCP 的每个应用程序在其生命周期中都有特定的管理要求。这包括开发和测试环境，以及作为应用程序生命周期一部分的其他阶段。下面列出了应用程序生命周期的一些指南:

*   使用文件夹和项目对开发团队和/或应用程序进行分组
*   尽早设置您的 CI/CD 环境。
*   使用版本控制—确保您可以像推出更新一样轻松地回滚
*   使用基础设施作为代码的原则来定义环境。像对待代码一样对待基础设施环境。将您的应用程序代码和它将被部署到的环境视为一个版本
*   使用标签——标识组件、谁拥有东西、版本，以进行分类
*   开发应用程序时，使用小段代码(微服务或功能)、小功能单元、使用组件间的契约、解耦、定义服务边界以及如何实现该边界(例如 kubernetes 的防火墙规则或 RBAC 规则)
*   测试失败场景—测试失败场景如果您不习惯在生产环境中这样做，请创建一个反映您的生产环境的测试环境。重复学习
*   自动化所有的事情
*   使用标签对资源进行逻辑分组
*   为规模而设计—您可能开始时只有几个用户，但预计会快速增长
*   在设计和开发您的应用程序时，将安全性视为一等公民
*   实施变更管理过程——即使您尽可能地自动化，您也需要有一个定义的变更管理过程
*   应用程序安全性—您如何对登录到您的应用程序的用户进行身份验证？您是否考虑过如何对 API 网关的调用进行身份验证，或者保护您的应用程序免受潜在威胁
*   如果您需要遵守定义谁可以访问应用程序中的内容的法规，请定义这些角色
*   高架通道的使用应该有更严格的控制。不应将超级用户访问作为应用程序日常访问的一部分，尤其是在应用程序涉及敏感数据的情况下。应该是个例外。
*   创建一个启动清单，以确保您拥有所有项目，或者至少是您认为能够成功启动的项目

你在期待一份阅读清单，所以我不想让你失望:

[https://cloud.google.com/docs/tutorials](https://cloud.google.com/docs/tutorials)

[标签&分组你的 GCP 资源](https://cloudplatform.googleblog.com/2018/06/Labelling-and-grouping-your-Google-Cloud-Platform-resources.html)

[https://landing . Google . com/sre/book/chapters/reliable-product-launchs . html](https://landing.google.com/sre/book/chapters/reliable-product-launches.html)

https://cloud.google.com/source-repositories/

[https://cloud . Google . com/solutions/automated-canary-analysis-kubernetes-engine-spinnaker](https://cloud.google.com/solutions/automated-canary-analysis-kubernetes-engine-spinnaker)

[https://cloud . Google . com/solutions/continuous-delivery-spinnaker-kubernetes-engine](https://cloud.google.com/solutions/continuous-delivery-spinnaker-kubernetes-engine)

[https://cloud . Google . com/solutions/ansi ble-with-spinnaker-tutorial](https://cloud.google.com/solutions/ansible-with-spinnaker-tutorial)

[https://cloud . Google . com/solutions/Jenkins-on-kubernetes-engine-tutorial](https://cloud.google.com/solutions/jenkins-on-kubernetes-engine-tutorial)

[https://cloud . Google . com/solutions/continuous-delivery-Jenkins-kubernetes-engine](https://cloud.google.com/solutions/continuous-delivery-jenkins-kubernetes-engine)

[https://cloud . Google . com/solutions/continuous-delivery-Jenkins-kubernetes-engine](https://cloud.google.com/solutions/continuous-delivery-jenkins-kubernetes-engine)

[https://cloud.google.com/docs/platform-launch-checklist](https://cloud.google.com/docs/platform-launch-checklist)

这是检查清单:

可以在[这里](/@grapesfrog/using-gcp-theres-a-checklist-for-that-76d61d1ffcbc)找到该系列中所有清单的列表