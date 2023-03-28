# GCP 清单 3 —产能规划

> 原文：<https://medium.com/google-cloud/gcp-checklist-3-capacity-planning-8d46b594da34?source=collection_archive---------1----------------------->

云中的容量规划有所不同，因为不需要预先调配闲置资源。服务可以针对您的工作负载进行优化，并根据需要调整容量。但是你需要考虑

*   区域和地区—如果可能的话，将您的应用程序分布在不同的区域，以获得弹性。利用区域性服务，如全球负载平衡
*   Google App Engine——将根据负载自动缩放您的应用程序
*   托管实例组—使用托管实例组来管理实例的相同副本
*   托管服务—为您自动扩展，例如 bigquery
*   您的配额需要提高到默认水平以上的什么水平
*   与您的客户团队合作

这是你的阅读清单:

[https://cloud.google.com/docs/geography-and-regions](https://cloud.google.com/docs/geography-and-regions)

https://cloud.google.com/compute/docs/instance-groups/

[https://cloud . Google . com/load-balancing/docs/https/cross-region-example](https://cloud.google.com/load-balancing/docs/https/cross-region-example)

[https://cloud.google.com/terms/service-terms](https://cloud.google.com/terms/service-terms)

[https://cloud . Google . com/blog/products/GCP/time-to-hello-world-VMs-vs-containers-vs-PAAs-vs-FAAS](https://cloud.google.com/blog/products/gcp/time-to-hello-world-vms-vs-containers-vs-paas-vs-faas)

[https://cloud . Google . com/solutions/about-capacity-optimization-with-global-lb](https://cloud.google.com/solutions/about-capacity-optimization-with-global-lb)

[https://cloud . Google . com/load-balancing/docs/tutorials/optimize-app-latency](https://cloud.google.com/load-balancing/docs/tutorials/optimize-app-latency)

[](https://cloud.google.com/solutions/transferring-big-data-sets-to-gcp) [## 转移大数据集|解决方案|谷歌云

### 谷歌云提供安全、开放、智能和变革性的工具，帮助企业实现现代化，以适应当今的…

cloud.google.com](https://cloud.google.com/solutions/transferring-big-data-sets-to-gcp) 

[http://googlecloudplatform.github.io/PerfKitBenchmarker/](http://googlecloudplatform.github.io/PerfKitBenchmarker/)

你读完了《棒极了》,下面是检查清单:

可以在[这里](/@grapesfrog/using-gcp-theres-a-checklist-for-that-76d61d1ffcbc)找到该系列中所有清单的列表