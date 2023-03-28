# GCP 清单 8 —成本优化

> 原文：<https://medium.com/google-cloud/gcp-checklist-8-cost-optimisation-381370d3748?source=collection_archive---------0----------------------->

作为配置 GCP 组织和组织内应用程序环境的一部分，您需要考虑成本。如果不对自动伸缩服务进行合理的限制，会发生什么？您的资源利用不足吗？你有预算吗？如果达到预算，你会采取什么行动？

GCP 有许多特性可以帮助你管理成本，你应该了解并利用这些特性。

*   当你设计了你的应用程序，并对所需的资源有了一些概念后，你就可以把数据输入到价格计算器中了
*   当您使用虚拟机运行持续工作负载时，Google Compute Engine 的持续使用折扣会自动降低虚拟机(VM)的价格。您不需要采取任何主动措施来使折扣生效。
*   谷歌计算引擎的承诺使用折扣提供了购买承诺使用合同的能力，以换取虚拟机使用的大幅折扣价格。如果您的工作负载稳定且可预测，您可以以正常价格的折扣购买特定数量的 vCPUs 和内存，以换取 1 年或 3 年的使用期限。
*   设置一个特定金额的预算提醒，或者与上个月的金额相匹配，当花费超过预算的一个百分比时，就会发出提醒。给你时间采取适当的行动。
*   默认配额有助于您将配额保持在设定的使用量内，直到您明确请求提高配额
*   如果你的预算没有余地，并且一旦达到上限，你的应用程序就无法工作，那么你可以根据 API，以各种方式明确地限制请求，包括:*每天请求*，*每 100 秒请求*，以及*每用户每 100 秒请求*。
*   如果您的应用程序不符合标准实例类型，您可以使用经过优化的定制机器类型来匹配您的应用程序的配置文件，并最终为您的应用程序使用的资源付费，而不是为您在使用标准实例时可能最终需要的额外 vCpu 和内存付费
*   GCP 提供机器类型建议，帮助您优化虚拟机实例的资源利用率。这些建议是根据 Google Stackdriver 监控服务在过去 8 天中收集的系统指标自动生成的
*   GCP 提供计费报告，您还可以创建一个计费仪表板，让您了解您的成本和分析成本趋势。

当您准备好开始工作时，请确保首先阅读[计费入职清单](https://cloud.google.com/billing/docs/onboarding-checklist)

这份阅读清单比我预期的略长，但它们都值得一读:

[https://cloud.google.com/billing/docs/onboarding-checklist](https://cloud.google.com/billing/docs/onboarding-checklist)

【https://cloud.google.com/pricing/innovation 号

【https://cloud.google.com/products/calculator/ 

[https://cloud.google.com/billing/docs/how-to/billing-access](https://cloud.google.com/billing/docs/how-to/billing-access)

[https://cloud.google.com/iam/docs/job-functions/billing](https://cloud.google.com/iam/docs/job-functions/billing)

[https://cloud.google.com/billing/docs/how-to/reports](https://cloud.google.com/billing/docs/how-to/reports)

[https://cloud.google.com/compute/quotas#checking_your_quota](https://cloud.google.com/compute/quotas#checking_your_quota)

[标记&对您的 GCP 资源进行分组](https://cloudplatform.googleblog.com/2018/06/Labelling-and-grouping-your-Google-Cloud-Platform-resources.html)

[https://cloud.google.com/billing/docs/how-to/visualize-data](https://cloud.google.com/billing/docs/how-to/visualize-data)

[https://cloud . Google . com/compute/docs/instances/apply-sizing-suggestions-for-instances](https://cloud.google.com/compute/docs/instances/apply-sizing-recommendations-for-instances)

[https://cloud . Google . com/compute/docs/sustain-use-discounts](https://cloud.google.com/compute/docs/sustained-use-discounts)

[https://cloud . Google . com/compute/docs/instances/signing-up-committed-use-discounts](https://cloud.google.com/compute/docs/instances/signing-up-committed-use-discounts)

[https://cloud.google.com/custom-machine-types/](https://cloud.google.com/custom-machine-types/)

[https://cloud.google.com/apis/docs/capping-api-usage](https://cloud.google.com/apis/docs/capping-api-usage)

[https://cloud.google.com/billing/docs/how-to/budgets](https://cloud.google.com/billing/docs/how-to/budgets)

这是检查清单:

该系列中所有清单的列表可在[这里](/@grapesfrog/using-gcp-theres-a-checklist-for-that-76d61d1ffcbc)找到