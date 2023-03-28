# 在 GCE 中运行 Gitlab CI Runner

> 原文：<https://medium.com/google-cloud/running-gitlab-ci-runner-in-gce-b74b5343c044?source=collection_archive---------0----------------------->

本周早些时候，我决定尝试一下 Gitlab CI，并开始研究如何建立一个 Docker 容器集群来处理所有的测试。

经过大量的反复试验和数小时的挫折(大约 20 小时)，我终于想出了一个看起来相当可靠的组合。

现在我想把这个组合分享给任何人使用，这样他们就不用花那些令人沮丧的时间(希望如此)在调试 [Gitlab Multi Runner](https://gitlab.com/gitlab-org/gitlab-ci-multi-runner) 、 [Docker Machine](https://docs.docker.com/machine/) 和 [GCE](https://cloud.google.com/compute/) 组合的怪异之处了。

在我的 Github repo[https://github.com/jerryjj/gitlab-runner-gce](https://github.com/jerryjj/gitlab-runner-gce)中可以找到关于如何设置这个自动伸缩基础设施的完整设置和说明。

简而言之，这个设置将执行以下操作:
1 .在 GCP 项目中设置一个 NAT-gateway 供工人使用
2。设置一个 g1-small 实例作为 Gitlab Runner
3。将这个 runner 注册到您的 Gitlab 实例
4。配置 Docker 机器以启动 G1-小型可抢占虚拟机作为工作机。

在空闲状态下，这种配置的成本约为。每月 20 美元。

安装完成并且集群稳定后，使用它就非常简单了。
在 Node.js 项目的例子中，你可以写一个这样的文件，并将其命名为”。gitlab-ci-yml”并将其推送到您的存储库

```
image: node:6cache:
 paths:
 — node_modules/test_all:
 stage: test
 script:
 — npm install
 — npm test
 tags:
 — backend
```

就这样，你的跑步者集群介入并开始为你工作。

*原载于 2016 年 9 月 22 日*[*total Lyon . me*](http://totallyon.me/2016/09/22/running-gitlab-ci-runner-in-gce/)*。*