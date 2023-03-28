# GCP 清单 6 —记录、监控和警报(保持可靠性)

> 原文：<https://medium.com/google-cloud/gcp-checklist-6-monitoring-and-alerting-maintaining-reliability-5701a8b9e86f?source=collection_archive---------1----------------------->

当涉及到维护系统的可靠性时，理解系统的典型行为是至关重要的。一旦你理解了典型的行为，你就能识别异常并采取行动。这需要为日志记录、监控和警报建立适当的框架

日志记录—您需要收集和分析日志来查找应用程序异常，并审核您的应用程序和环境。

监控—与日志记录密切相关，通常与日志记录密切相关。典型的监控解决方案包括收集指标的方法、查看系统和应用程序状态的仪表板以及发送警报的方法。您需要对您的系统进行测试，以提供有意义的指标。

作为平台的一部分，GCP 拥有日志记录和监控服务

以下是一些参考资料，可以作为良好的起点:

[https://cloud.google.com/logging/docs/](https://cloud.google.com/logging/docs/)

https://cloud.google.com/monitoring/audit-logging

【https://cloud.google.com/monitoring/docs/ 

[https://cloud . Google . com//monitoring/alerts/using-alerting-ui](https://cloud.google.com//monitoring/alerts/using-alerting-ui)

[https://cloud . Google . com/solutions/design-patterns-for-export-stack driver-logging](https://cloud.google.com/solutions/design-patterns-for-exporting-stackdriver-logging)

[https://Cloud platform . Google blog . com/2018/03/best-practices-for-work-with-Google-Cloud-Audit-logging . html](https://cloudplatform.googleblog.com/2018/03/best-practices-for-working-with-Google-Cloud-Audit-Logging.html)

[https://cloud . Google . com/blog/products/management-tools/building-a-more-reliable-infra structure-with-new-stack driver-tools-and-partners](https://cloud.google.com/blog/products/management-tools/building-a-more-reliable-infrastructure-with-new-stackdriver-tools-and-partners)

这是你的清单:

可以在[这里](/@grapesfrog/using-gcp-theres-a-checklist-for-that-76d61d1ffcbc)找到该系列中所有清单的列表