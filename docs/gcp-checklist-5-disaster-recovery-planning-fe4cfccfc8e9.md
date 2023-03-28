# GCP 清单 5 —灾难恢复规划

> 原文：<https://medium.com/google-cloud/gcp-checklist-5-disaster-recovery-planning-fe4cfccfc8e9?source=collection_archive---------3----------------------->

作为规划生产环境的一部分，您还需要有一个灾难恢复计划。服务中断事件随时都可能发生。您的网络可能会中断，您最新的应用程序推送可能会引入一个严重的错误，或者某一天您可能不得不应对一场自然灾害。出现问题时，制定一个强大、有针对性且经过充分测试的灾难恢复计划非常重要。

*   根据您的 RTO/RPO 值进行设计—应用程序的不同部分可能具有不同的 RTO 和 RPO 值
*   设计端到端恢复—确保您有返回生产环境的流程，在灾难恢复作为主站点时重放更新的数据，并向您的日志记录系统重放日志
*   配置安全控制，使其反映生产环境中的权限
*   检查软件许可证，以运行任何需要许可的应用程序的恢复版本
*   确保您的 CI/CD 系统能够部署到 GCP 上的恢复环境中
*   确保用户能够以适当的权限访问灾难恢复环境
*   定期测试你的计划
*   让您的灾难恢复环境保持最新
*   如果您愿意，可以将常规故障注入到生产环境中，以模拟微故障，并强制执行恢复过程或故障切换到灾难恢复环境

这次给你一份不错的简短阅读清单😃

[https://cloud . Google . com/solutions/dr-scenarios-planning-guide](https://cloud.google.com/solutions/dr-scenarios-planning-guide)

[https://cloud . Google . com/solutions/dr-scenarios-building-blocks](https://cloud.google.com/solutions/dr-scenarios-building-blocks)

【https://cloud.google.com/solutions/dr-scenarios-for-data 

[https://cloud . Google . com/solutions/dr-scenarios-for-applications](https://cloud.google.com/solutions/dr-scenarios-for-applications)

随附清单:

可以在[这里](/@grapesfrog/using-gcp-theres-a-checklist-for-that-76d61d1ffcbc)找到该系列中所有清单的列表