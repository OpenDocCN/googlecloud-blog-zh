# TWiGCP —“编年史、深度学习容器、GKE 上的最低权限、分析器 GA、数据目录和 Equiano”

> 原文：<https://medium.com/google-cloud/twigcp181-6e6c367cecb?source=collection_archive---------1----------------------->

如果你是本周从 [**来到谷歌云的视频系列**](http://gtech.run/ju4em) 的，以下是本周所涉及主题的链接:

*   [Istio，改良！](http://gtech.run/vz27w) #1.2
*   [Easy ML](http://gtech.run/q8guy) #BigQueryML
*   [心不在焉？](http://gtech.run/dxdl9) #CloudNativeArchitecture
*   [提交&保存！](http://gtech.run/g6ksg) #CommittedUseDiscounts

过去一周 GCP 的其他头条新闻包括:

*   [谷歌云+ **编年史**:安全 moonshot 加入谷歌云](http://gtech.run/q8mw8)(谷歌博客)#NoSuchThingAsTooMuchSecurity
*   [介绍**深度学习容器**:一致且可移植的环境](http://gtech.run/y6dqg)(谷歌博客)gcr.io/deeplearning-platform-release
*   [介绍**工作负载身份**:为您的 GKE 应用提供更好的身份验证](http://gtech.run/6l323) (Google 博客)#LeastPrivilege
*   [用 **Stackdriver Profiler** ，现在 GA](http://gtech.run/2re8f) (Google 博客)#FlameGraphsFTW，看看你的代码实际上是如何执行的
*   [谷歌**云数据目录**现已公开测试](http://gtech.run/t4h2c)(谷歌博客)#MetadataManagement4all

来自“不是产品或功能，但仍然是基础”部门:

*   [介绍 Equiano，一条从葡萄牙到南非的海底电缆](http://gtech.run/4lez4)(谷歌博客)
*   负责任的人工智能:将我们的原则付诸行动

来自“GCP 操作方法和最佳实践”部门:

*   [GCP·德沃普斯的技巧:创建一个自定义的云壳图像，包括地形和头盔](http://gtech.run/nwmpu)(谷歌博客)
*   [如何用 AutoML 实现文档标记](http://gtech.run/rtf3g)(谷歌博客)
*   [掌控你的数据:标记化如何在不牺牲隐私的情况下让数据变得可用](http://gtech.run/j8j6m)(谷歌博客)
*   粒子云功能&普罗米修斯(medium.com

来自“GCP Kubernetes 安全选项”部门:

*   使用云构建和 GKE 实现二进制授权(cloud.google.com)
*   内部敌人:用 gVisor 运行不可信的代码(speakerdeck.com)

来自“花生酱和果冻”部门:

*   [用 Kaggle 内核笔记本分析 BigQuery 数据](http://gtech.run/5hnnu)(谷歌博客)
*   [社区聚焦:BigQuery 插件](http://gtech.run/xthjr)(grafana.com)# doit international

来自“金融市场管理局在瑞士很重要”部门:

*   [支持瑞士受 FINMA 监管的客户](http://gtech.run/eyzze)(谷歌博客)

来自“其他人是如何做到的”部分:

*   [SRE 团队是如何组织的，如何开始](http://gtech.run/2e42h)(谷歌博客)
*   [为每个数据管道建立一个仪表板](http://gtech.run/9xeed)(medium.com)

来自“云运行和底层技术”部门:

*   [云跑 meetup 社区素材](http://gtech.run/xffrr)(jhanley.com)
*   [Knative 教程(0.7 版本更新)【github.com ](http://gtech.run/gagt3)

来自我最喜欢的“客户和合作伙伴对 GCP 的最佳评价”部分:

*   [genome on 通过谷歌云平台提供基因组变异数据](http://gtech.run/4dttw)(clpmag.com)
*   [威尼托大区:G 套房& GCP 为 500 万意大利公民改造市政服务](http://gtech.run/hujbz)(youtube.com)

**Beta，GA，还是什么？**“部门:

*   [GA] [云 SDK 252.0.0](http://gtech.run/wfg8w)
*   [交通总监](http://gtech.run/dtmub)
*   [GA] [堆栈驱动程序分析器](http://gtech.run/2re8f)
*   【GA】[云扳手——导入导出 CSV 格式的数据](http://gtech.run/55d4x)
*   【GA】[云 NAT 日志](http://gtech.run/xrcub)
*   [GA] [传送装置—欧盟](http://gtech.run/pyanb)
*   【Beta】[云数据目录](http://gtech.run/4vykk)
*   [Beta] [具有资源位置约束的组织策略](http://gtech.run/repeu)
*   [0.7] [Knative](http://gtech.run/zn3gl)

来自“**多媒体**”部门:

*   [YouTube][youtube.com CNCF 终端用户案例研究:Jai Chakrabarti，Spotify](http://gtech.run/8lw42)
*   [播客] Kubernetes 播客[第 59 集——班仔云，Janos Matyas](/google-cloud/gtech.run/dgcnk)(kubernetespodcast.com)
*   (gcppodcast.com)GCP 播客[第 182 集——迈克尔·克莱纳曼的谷歌云平台 UX](http://gtech.run/ayk9p)

[![](img/ee79df63a0afbebb2f99a1052e61199e.png)](http://gtech.run/nwmpu)

本周的图片取自“创建一个包含地形和头盔的自定义云壳图像”

这就是本周的全部内容！亚历克西斯