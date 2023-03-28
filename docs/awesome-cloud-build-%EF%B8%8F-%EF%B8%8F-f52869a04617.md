# 令人敬畏的云构建🕶️🌩️ 🔧

> 原文：<https://medium.com/google-cloud/awesome-cloud-build-%EF%B8%8F-%EF%B8%8F-f52869a04617?source=collection_archive---------1----------------------->

> [*#牛逼云搭*](https://github.com/Timtech4u/awesome-cloudbuild)
> 
> *关于*[*Google Cloud Build*](https://console.cloud.google.com/cloud-build)上所有事物的资源列表

![](img/6109d82b4e51a1307e015e00c5606bcd.png)

我整理了一堆资源(**有用链接**，**快速入门**，**教程**，**文章**，**官方云构建者**，**社区云构建者**，**云构建配置文件模板**和 **Meetups** )都与 Google Cloud Build 有关，如果你想的话，可以随时跳到 [GitHub](https://github.com/Timtech4u/awesome-cloudbuild)

你可以在下面找到列表的初始内容，或者点击查看[的更新内容](https://github.com/Timtech4u/awesome-cloudbuild)

# 什么是云构建？

Cloud Build 是一项在 Google 云平台基础设施上执行构建的服务。

Cloud Build 可以从 Google Cloud Storage、Cloud Source Repositories、GitHub 或 Bitbucket 导入源代码，按照您的规范执行构建，并生成工件。

云构建将您的构建作为一系列构建步骤来执行，其中每个构建步骤都由一个构建器映像在 Docker 容器中运行。

# 内容

*   [有用链接](https://fullstackgcp.com/awesome-cloud-build-cjydaet2g000o5os1v71zb3v4#useful-links)
*   [快速入门](https://fullstackgcp.com/awesome-cloud-build-cjydaet2g000o5os1v71zb3v4#quickstarts)
*   [教程](https://fullstackgcp.com/awesome-cloud-build-cjydaet2g000o5os1v71zb3v4#tutorials)
*   [文章](https://fullstackgcp.com/awesome-cloud-build-cjydaet2g000o5os1v71zb3v4#articles)
*   [云构建者](https://fullstackgcp.com/awesome-cloud-build-cjydaet2g000o5os1v71zb3v4#cloud-builders)
*   [社区云建设者](https://fullstackgcp.com/awesome-cloud-build-cjydaet2g000o5os1v71zb3v4#community-cloud-builders)
*   [云构建配置文件模板](https://fullstackgcp.com/awesome-cloud-build-cjydaet2g000o5os1v71zb3v4#cloud-build-configuration-file-templates)
*   [聚会](https://fullstackgcp.com/awesome-cloud-build-cjydaet2g000o5os1v71zb3v4#meetups)

# 有用的链接

关于云构建的有用链接。

*   [产品概述](https://cloud.google.com/cloud-build/)
*   [官方文件](https://cloud.google.com/cloud-build/docs/)
*   [云构建指南](https://cloud.google.com/cloud-build/docs/how-to)
*   [云构建控制台](https://console.cloud.google.com/cloud-build)
*   [谷歌云上的 CI/CD](https://cloud.google.com/docs/ci-cd/)
*   [堆栈溢出](http://stackoverflow.com/questions/tagged/google-cloud-build)
*   [文件错误或功能请求](https://issuetracker.google.com/issues?q=componentid:190802)
*   [定价](https://cloud.google.com/cloud-build/pricing)
*   [配额&限制](https://cloud.google.com/cloud-build/quotas)
*   [发行说明](https://cloud.google.com/cloud-build/release-notes)

# 快速入门

云构建快速入门。

*   [Docker 快速入门](https://cloud.google.com/cloud-build/docs/quickstart-docker)
*   [Go 快速入门](https://cloud.google.com/cloud-build/docs/quickstart-go)
*   [打包机快速启动](https://cloud.google.com/cloud-build/docs/quickstart-packer)

# 教程

云构建教程。

*   [基于云构建的 GitOps 式持续交付](https://cloud.google.com/kubernetes-engine/docs/tutorials/gitops-cloud-build)
*   [为第三方服务配置通知](https://cloud.google.com/cloud-build/docs/configure-third-party-notifications)
*   [访问私有 GitHub 库](https://cloud.google.com/cloud-build/docs/access-private-github-repos)
*   [在 GitHub 上运行构建](https://cloud.google.com/cloud-build/docs/run-builds-on-github)
*   [使用 Google Cloud Build CI/CD 和 Gradle Docker 映像自动构建 Android APKs】](https://fullstackgcp.com/automate-building-android-apks-with-google-cloud-build-cicd-and-a-gradle-docker-image-cloud-cjy15jb3o0028css1m0og45nw)
*   [使用云构建构建奇点容器](https://cloud.google.com/community/tutorials/singularity-containers-with-cloud-build)
*   [使用云构建执行角度服务器端(预)渲染](https://cloud.google.com/community/tutorials/cloudbuild-angular-universal)
*   [简化谷歌云平台上的持续部署](/@timtech4u/simplified-continuous-deployment-on-google-cloud-platform-bc5b0a025c4e)
*   [使用打包程序创建云构建映像工厂](https://cloud.google.com/community/tutorials/create-cloud-build-image-factory-using-packer)
*   [采用云构建的自动化静态网站发布](https://cloud.google.com/community/tutorials/automated-publishing-cloud-build)
*   [使用云构建作为测试运行器](https://cloud.google.com/community/tutorials/cloudbuild-test-runner)
*   [通过云运行运行仙丹凤凰 app](https://cloud.google.com/community/tutorials/elixir-phoenix-on-cloud-build-cloud-run)
*   [WhiteSource —谷歌云构建集成](https://whitesource.atlassian.net/wiki/spaces/WD/pages/544604326/Google+Cloud+Build+Integration)
*   [在 Google Cloud Build 上构建 Go 项目时利用缓存](https://blog.container-solutions.com/utilizing-caches-when-building-go-projects-on-google-cloud-build)
*   [借助 GCP 云构建，自动构建触手可及](https://mydeveloperplanet.com/2019/05/08/automatic-builds-at-your-fingertips-with-gcp-cloud-build/)
*   [通过云构建简化云运行的持续部署，包括自定义域设置(SSL)](/google-cloud/simplifying-continuous-deployment-to-cloud-run-with-cloud-build-including-custom-domain-setup-ssl-22d23bed5cd6)
*   [与 Google Cloud Build-Fireship 的持续集成和部署](https://fireship.io/lessons/ci-cd-with-google-cloud-build/)
*   [使用 Spinnaker 构建谷歌云](https://www.spinnaker.io/setup/ci/gcb/)
*   [使用 Google Cloud Build (CI/CD)部署 Gatsby 站点到 Firebase】](https://dev.to/leomercier/deploying-a-gatsby-site-to-firebase-with-google-cloud-build-ci-cd-511c)
*   [持续部署云构建](https://codelabs.developers.google.com/codelabs/cloud-builder-gke-continuous-deploy/index.html#0)

# 文章

关于云构建的文章。

*   [建造配置概述](https://cloud.google.com/cloud-build/docs/build-config)
*   [circle ci vs Google Cloud Build](https://www.praqma.com/stories/circle-ci-google-cloud-build/)
*   [谷歌宣布推出新的持续集成和交付平台 Cloud Build](https://techcrunch.com/2018/07/24/google-announces-cloud-build-its-new-continuous-integration-continuous-delivery-platform/)
*   [更好的谷歌云持续部署方法](https://www.toptal.com/devops/better-google-cloud-continuous-deployment)
*   [git lab-Google cloud build](https://about.gitlab.com/devops-tools/cloudbuild/)
*   [Terraform —谷歌云构建](https://www.terraform.io/docs/providers/google/r/cloud_build_trigger.html)
*   [构建容器的最佳实践](https://cloud.google.com/solutions/best-practices-for-building-containers)

# 云构建者

Google Cloud Build 支持的构建器图像，代码在 [GitHub](https://github.com/GoogleCloudPlatform/cloud-builders) 上

*   [巴泽尔:运行巴泽尔工具](https://github.com/GoogleCloudPlatform/cloud-builders/tree/master/bazel)
*   [卷曲:运行卷曲工具](https://github.com/GoogleCloudPlatform/cloud-builders/tree/master/curl)
*   [docker:运行 docker 工具](https://github.com/GoogleCloudPlatform/cloud-builders/tree/master/docker)
*   [点网:运行点网工具](https://github.com/GoogleCloudPlatform/cloud-builders/tree/master/dotnet)
*   [gcloud:运行 gcloud 工具](https://github.com/GoogleCloudPlatform/cloud-builders/tree/master/gcloud)
*   [git:运行 git 工具](https://github.com/GoogleCloudPlatform/cloud-builders/tree/master/git)
*   [gke-deploy:运行 gke-deployer 工具](https://github.com/GoogleCloudPlatform/cloud-builders/tree/master/gke-deploy)
*   [go:运行 go 工具](https://github.com/GoogleCloudPlatform/cloud-builders/tree/master/go)
*   [格拉德:运行格拉德工具](https://github.com/GoogleCloudPlatform/cloud-builders/tree/master/gradle)
*   [gsutil:运行 gsutil 工具](https://github.com/GoogleCloudPlatform/cloud-builders/tree/master/gsutil)
*   [javac:运行 javac 工具](https://github.com/GoogleCloudPlatform/cloud-builders/tree/master/javac)
*   [kubectl:运行 kubectl 工具](https://github.com/GoogleCloudPlatform/cloud-builders/tree/master/kubectl)
*   [mvn:运行 maven 工具](https://github.com/GoogleCloudPlatform/cloud-builders/tree/master/mvn)
*   [npm:运行 npm 工具](https://github.com/GoogleCloudPlatform/cloud-builders/tree/master/npm)
*   [wget:运行 wget 工具](https://github.com/GoogleCloudPlatform/cloud-builders/tree/master/wget)
*   [纱线:运行纱线工具](https://github.com/GoogleCloudPlatform/cloud-builders/tree/master/yarn)

# 社区云构建者

Google Cloud Build 社区贡献的图片， [GitHub](https://github.com/GoogleCloudPlatform/cloud-builders-community) 上的代码

*   [安卓:运行安卓工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/android)
*   [ansible:运行 ansible 工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/ansible)
*   [awscli:运行 awscli 工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/awscli)
*   [az-kubectl:运行 azure-kubectl 工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/az-kubectl)
*   [az:运行 azure cli 工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/az)
*   [bandit:运行 bandit 工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/bandit)
*   [基础映像构建器:运行基础映像构建器工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/base-image-builder)
*   [binauthz-证明:运行二进制授权工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/binauthz-attestation)
*   [启动:运行启动工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/boot)
*   [bq:运行 bigquery 工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/bq)
*   [buildah:运行 buildah 工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/buildah)
*   [缓存:运行缓存工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/cache)
*   [货物:运行生锈货物工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/cargo)
*   [cft:运行云基础工具包](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/cft)
*   [compodoc:运行 compodoc 工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/compodoc)
*   [作曲:运行作曲工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/composer)
*   [容器比较:运行容器比较工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/container-diff)
*   [cron-helper:运行 cron-helper 工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/cron-helper)
*   [数据流-python:运行数据流-python 工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/dataflow-python)
*   [dep:运行 go dep 工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/dep)
*   [docker-compose:运行 docker-compose 工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/docker-compose)
*   [envsubst:运行 envsubst 工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/envsubst)
*   [esy:运行 esy 工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/esy)
*   [快速通道:运行快速通道工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/fastlane)
*   [firebase:运行 firebase 工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/firebase)
*   [颤动:运行颤动工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/firebase)
*   [fsharp:运行 fsharp 工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/fsharp)
*   [滑动:运行滑动工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/glide)
*   [谷歌闭包编译器:运行谷歌闭包编译器工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/google-closure-compiler)
*   [头盔:运行头盔工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/helm)
*   [中枢:运行中枢工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/hub)
*   [雨果:运行雨果工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/hugo)
*   [检查:运行检查工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/inspec)
*   [jfrog:运行 jfrog 工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/jfrog)
*   [jmeter:运行 jmeter 工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/jmeter)
*   [jsonnet:运行 jsonner 工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/jsonnet)
*   [kaniko:运行 kaniko 工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/kaniko)
*   [kubectl_wait_for_job:运行 kubectl_wait_for_job 工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/kubectl_wait_for_job)
*   [kustomize::运行 kustomize 工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/kustomize)
*   [制作:运行制作工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/make)
*   [makisu:运行 makisu 工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/make)
*   [砂浆:运行砂浆工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/mortar)
*   [ng:运行 ng 工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/ng)
*   [nix 构建:运行 nix 构建工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/nix-build)
*   [npm-jasmine-node:运行 npm-jasmine-node 工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/npm-jasmine-node)
*   [封隔器:运行封隔器工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/packer)
*   [鹈鹕:运行鹈鹕工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/pelican)
*   [协议:运行协议工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/protoc)
*   [pulumi:运行 pulumi 工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/pulumi)
*   [木偶皮棉:运行木偶皮棉工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/puppet-lint)
*   [pylint:运行 pylint 工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/pylint)
*   [远程构建器:运行远程构建器工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/remote-builder)
*   [摇杆:运行摇杆工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/rocker)
*   [s2i:运行 s2i 工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/s2i)
*   [scala-sbt:运行 scala-sbt 工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/scala-sbt)
*   [外壳检查:运行外壳检查工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/shellcheck)
*   [奇点:运行奇点工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/singularity)
*   [skaffold:运行 skaffold 工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/skaffold)
*   [slackbot:运行 slackbot 工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/slackbot)
*   [sonar cube:运行 sonar cube 工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/sonarqube)
*   [swift:运行 swift 工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/swift)
*   运行焦油工具
*   [地形:运行地形工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/terraform)
*   [terragrunt:运行 terragrunt 工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/terragrunt)
*   [追踪路径:运行追踪路径工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/traceroute)
*   [windows-builder:运行 windows-builder 工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/windows-builder)
*   [纱线操纵器:运行纱线操纵器工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/yarn-puppeteer)
*   [压缩:运行压缩工具](https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/zip)

# 云构建配置文件模板

**cloudbuild.yaml** 模板将指导您创建自己的模板。你可以在这里找到[docker files](https://github.com/jessfraz/dockerfiles)

*   [部署到云运行](https://fullstackgcp.com/build-config-templates/deploy-to-cloud-run/cloudbuild.yaml)
*   [部署到应用引擎](https://fullstackgcp.com/build-config-templates/deploy-to-app-engine/cloudbuild.yaml)
*   [部署到计算引擎](https://fullstackgcp.com/build-config-templates/deploy-to-compute-engine/cloudbuild.yaml)