# 通过谷歌云平台免费试用 ConcourseCI

> 原文：<https://medium.com/google-cloud/playing-with-concourseci-via-a-google-cloud-platform-free-trial-65acfbdd02d2?source=collection_archive---------0----------------------->

作为一名软件开发人员，我已经开始练习每个周末花几个小时来磨练和提高我的技能。这个周末的课程是通过 [Bosh](https://bosh.io/) [Google CPI](https://github.com/cloudfoundry-incubator/bosh-google-cpi-release) 将我自己的[concoursesi](http://concourse.ci)集群部署到 [Google 计算平台](https://cloud.google.com)。

我猜大家都知道亚马逊的云计算平台叫做 [Amazon Web Services](https://aws.amazon.com/) ，简称 AWS。直到最近，我才意识到谷歌有一个竞争平台，叫做[谷歌计算平台](https://cloud.google.com)，简称 GCP。与 AWS 相比，我喜欢 GCP 的一点是定价模式。AWS 向你收取整小时的使用费，而 GCP 则在至少 10 分钟后按分钟收费。更不用说 300 美元的 60 天免费试用。

对我来说幸运的是，谷歌的人做了很好的工作，记录了将 Concourse 安装到 GCP 的过程。

以下是我对上述说明所做的一些调整，以使其适合免费计划(当你阅读这篇文章时，最新的版本号可能已经不同了):

1.  在[部署 BOSH director](https://github.com/cloudfoundry-incubator/bosh-google-cpi-release/tree/master/docs/concourse#deploy-a-bosh-director) 的第 8 步，将 **manifest.yml** 文件中的**resource _ pools->cloud _ properties->machine _ type**从 **n1-standard-4** 更改为 **n1-standard-1** 。每个区域最多有 8 个 CPU，所以为你使用 4 个会消耗很多容量，并且会导致安装失败。这将降低 director 的安装和后续更新速度。您可以在关闭免费试用后随时更改此设置。
2.  同样在[部署 BOSH director](https://github.com/cloudfoundry-incubator/bosh-google-cpi-release/tree/master/docs/concourse#deploy-a-bosh-director) 的第 8 步中，更新到最新版本的 [BOSH](http://bosh.io/releases/github.com/cloudfoundry/bosh?all=1) (257.15)、 [Google CPI](http://bosh.io/releases/github.com/cloudfoundry-incubator/bosh-google-cpi-release?all=1) (25.4.1)和 [Google stemcell](http://bosh.io/stemcells/bosh-google-kvm-ubuntu-trusty-go_agent) (3263.7)版本。旧的将会工作，但是为什么不在你设置东西的时候得到所有最新的好处。
3.  在 [Deploy Concourse](https://github.com/cloudfoundry-incubator/bosh-google-cpi-release/tree/master/docs/concourse#deploy-concourse) 的第 1 步，上传最新版本的 [Google stemcell](http://bosh.io/stemcells/bosh-google-kvm-ubuntu-trusty-go_agent) (3263.7)。
4.  在 [Deploy Concourse](https://github.com/cloudfoundry-incubator/bosh-google-cpi-release/tree/master/docs/concourse#deploy-concourse) 的第 2 步，上传最新版本的 [Concourse](http://bosh.io/releases/github.com/concourse/concourse) (2.3.1)和 [Garden RunC](http://bosh.io/releases/github.com/cloudfoundry/garden-runc-release?all=1) (0.9.2)。这些版本是捆绑在一起的，因此请查看 [Concourse 下载页面](https://concourse.ci/downloads.html)获取最新版本。
5.  在[Deploy concoure](https://github.com/cloudfoundry-incubator/bosh-google-cpi-release/tree/master/docs/concourse#deploy-concourse)的第 3 步，确保再次将 **cloud-config.yml** 中的 **machine_type** 更改为 **n1-standard-1** 以适应 CPU 配额。由于这是一个演示环境，您可能不需要额外的马力。免费试用期结束后，您可以随时将其改回来并重新部署。这就是波什的妙处。:-)
6.  在[部署 Concourse](https://github.com/cloudfoundry-incubator/bosh-google-cpi-release/tree/master/docs/concourse#deploy-concourse) 的第 4 步，将 **concourse.yml** 中的 **basic_auth_password** 修改为更复杂的内容。您不希望“黑客”在您的 Concourse 环境中四处窥探。:-)

仅此而已。按照这些小调整的指示，您应该可以快速启动并运行您自己的基于 GCP 的 Concourse 安装。

如果您正在寻找教程来帮助您学习 Concourse，请查看 [Concourse 教程页面](https://concourse.ci/tutorials.html)。飞行学校的例子是一个很好的开始方式。你可以在这里查看我的跑步版本:[http://107.178.255.195/](http://107.178.255.195/)