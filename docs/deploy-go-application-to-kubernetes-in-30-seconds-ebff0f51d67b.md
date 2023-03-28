# 在 30 秒内将 Go 应用程序部署到 Kubernetes

> 原文：<https://medium.com/google-cloud/deploy-go-application-to-kubernetes-in-30-seconds-ebff0f51d67b?source=collection_archive---------0----------------------->

![](img/ccf07ce777aaad16920cf1721468c164.png)

我一直在用 [Go](https://golang.org/) 编写一个简单的网络应用，托管在 [Google Cloud](https://cloud.google.com/) ，使用 [Google Kubernetes 引擎](https://cloud.google.com/kubernetes-engine/)。这是一次非常棒的经历，我不能错过分享的机会。我花了不到一个小时的时间第一次设置好东西，阅读文档并试图弄清楚事情是如何工作的。但是，一旦我完成了，我可以在 30 秒内部署我的应用程序的新版本！

我将要部署的应用程序是一个简单的 RESTful 内容服务器，是更大的微服务设置的一部分。它将暴露很少的 API 来发布和获取新闻。数据将存储在 MySQL 数据库中。不过，目前我将把数据库部分放在一边，只关注将应用程序交付给服务器。

应用程序配置在 YAML 文件中，该文件包含数据库 DSN 和一些其他参数。

源代码可从 [GitHub](https://github.com/skolodyazhnyy/30-seconds-deployment) 获得。

# 部署

我们的部署流程将运行测试，构建二进制文件，将其打包到 [docker 映像](https://docs.docker.com/engine/userguide/storagedriver/imagesandcontainers/)，上传到 [docker 注册表](https://docs.docker.com/registry/)，然后使用 [kubectl](https://kubernetes.io/docs/user-guide/kubectl-overview/) 将 docker 映像部署到 [Kubernetes](https://kubernetes.io/) 集群。

让我们首先创建一个 [Makefile](https://www.gnu.org/software/make/manual/make.html#Overview) 来保存每个部署步骤的指令。我喜欢 Makefiles，它们简单却非常强大。Makefiles 由`make`执行。这是一个轻量级的应用程序，几乎在任何地方都预装了它。

我们需要我们的构建、工件和部署的一些标识符，某种版本。我们可以使用语义版本控制或者顺序构建号，但是我发现使用 git 提交的散列更简单。它足够独特，有助于获得提交日志和工件之间的关系，并且最重要的是不需要太多的工作来生成。让我们在 Makefile 中定义一个变量来保存我们的版本名

```
TAG?=$(shell git rev-list HEAD --max-count=1 --abbrev-commit)
**export** TAG
```

使用 little [shell magic](https://www.gnu.org/software/make/manual/html_node/Shell-Function.html) 我们已经定义了标签变量，可以在运行 make 时手动设置，或者，如果没有设置，它将使用上次 git 提交的一个短散列(前 7 个符号)。然后，我们导出这个变量，这样它就可以在 make 运行的命令中使用。

## 试验

通过运行测试来开始部署似乎是个好主意。它会阻止我们制造破碎的艺术品。让我们创建目标`test`，它将简单地运行`go test ./...`。

```
test:
   go test ./...
```

## 建设

现在，我们需要构建一个二进制文件。我们将定义`build`目标，它将简单地用很少的参数运行`go build`。

```
build:
   go build -ldflags "-X main.version=$(TAG)" -o news .
```

Go 将构建一个二进制文件，静态链接运行它所需的一切。但是我们仍然需要一些可部署的单位，一些我们可以很容易分发的单位。这就是 [docker](https://www.docker.com/) 发挥作用的地方， [docker 图像](https://docs.docker.com/engine/userguide/storagedriver/imagesandcontainers/)和 [docker 注册表](https://docs.docker.com/registry/)非常适合我们的目标。

让我们创建 [Dockerfile](https://docs.docker.com/engine/reference/builder/) 以便能够将我们的应用程序打包到 docker 映像中

```
**FROM** alpine:3.4

**RUN** apk **-**U add ca-certificates

**EXPOSE** 8080

**ADD** news **/**bin**/**news
**ADD** config.yml.dist **/**etc**/**news**/**config.yml

**CMD** ["news", "-config", "/etc/news/config.yml"]
```

我们可以从头开始创建一个映像，但是我更喜欢将 [alpine](https://alpinelinux.org/) 作为基础映像，只需要几个额外的 MB，我们就可以得到包管理器和 [busybox](https://busybox.net/about.html) 。

我们的映像构建过程安装 *ca-certificates* ，公开端口 8080，添加我们在上一步中构建的二进制文件，以及一些默认配置。最后，我们定义运行应用程序所需命令。

我们需要 Makefile 中的另一个目标来构建 docker 映像。

```
pack: build
   docker build -t gcr.io/myproject/news-service:$(TAG) .
```

在制作映像之前，构建一个二进制文件是很重要的，否则，docker 构建会失败，或者会使用错误的应用程序版本。这就是为什么我们将`build`步骤定义为`pack`的依赖项。步骤本身将运行`docker build`并使用我们的标签变量标记图像。

我用的是[谷歌容器注册表](https://cloud.google.com/container-registry/)，因为它能更好地与其他谷歌云服务集成，但任何 docker 注册表都可以。

最后一步是把我们的图像放到注册表中，让我们为它设定一个目标

```
upload:
   docker push gcr.io/myproject/news-service:$(TAG)
```

## 部署

我们将把我们的应用程序部署到 Kubernetes 集群，有许多方法可以设置。您可以使用 [minikube](https://github.com/kubernetes/minikube) 设置本地环境，或者使用 [kops](https://github.com/kubernetes/kops) 在 [AWS](https://aws.amazon.com) 中设置集群。我将使用[谷歌 Kubernetes 引擎](https://cloud.google.com/kubernetes-engine/)，它很容易设置，并完全由[谷歌云](https://cloud.google.com/)管理。

如何设置 Kubernetes 集群并不重要，但在继续之前，请确保`kubectl`已连接到您想要用来部署应用程序的集群。

云中的应用程序需要一个配置文件，因此它知道如何连接到数据库和其他参数。我们可以在 docker 映像中放置适当的配置，但这并不实际。如果我们这样做，我们的映像将是环境感知的，更改配置将需要重建映像，如果我们希望将应用程序部署到不同的 Kubernetes 集群(例如，登台和生产环境)，我们将需要构建不同的映像。

相反，我们将使用 [Kubernetes ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configmap/) ，一个将保持我们的配置的对象。我们将把它作为一个文件安装在应用程序容器中。

我们的部署过程将只部署应用程序容器。该配置将作为群集配置的一部分单独部署。

我更喜欢单独保存配置，而不是和应用程序代码放在同一个存储库中。它应该是群集配置过程的一部分，而不是应用程序本身。

以下是我的配置映射对象:

```
**kind:** ConfigMap
**apiVersion:** v1
**metadata:
  name:** news-config
**data:
  config.yml:** |-
    server:
     idletimeout: 5s
     readtimeout: 5s
     writetimeout: 5s
     addr: ":8080"

    database:
     dsn: "proxyuser:password@(localhost:3306)/news"
```

您可以使用`kubectl apply -f configmap.yml`来部署它。或者，使用`kubectl create`命令创建 ConfigMap，这不会有太大的区别。

根据我的经验，描述你在 YAML 的 kubernetes 资源是一个更好的选择。它允许你跟踪变化，你可以把它们存储在版本控制系统中。总的来说，YAML 似乎比 shell 脚本中的一批`kubectl create`命令更加一致和明确。但是用你认为更合适的。

现在，让我们回到我们的应用程序，为我们的 pod 和服务创建一个定义。我将在我们的应用程序存储库中创建`k8s`文件夹，以跟踪运行我们的应用程序所需的所有 kubernetes 资源。

首先，我们在文件`k8s/deployment.yml`中创建 [Kubernetes 部署](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)

```
**apiVersion:** apps/v1beta1
**kind:** Deployment
**metadata:
  name:** news
  **labels:** {app: news}
**spec:
  replicas:** 1
  **template:
    metadata:** {labels: {app: news}}
    **spec:
      containers:** - **name:** news
        **image:** gcr.io/myproject/news-service:***${TAG}***
        **command:
        ports:** - **containerPort:** 8080
        **volumeMounts:** - **name:** news-config
            **mountPath:** /etc/news/
            **readOnly:** true
      **volumes:** - **name:** news-config
          **configMap:** { name: news-config }
```

它定义了应用程序所需的所有容器的规范。目前我们只需要一个容器，它将运行我们的应用程序映像。它将公开端口 8080，我们将挂载之前部署的配置。

注意，我使用`${TAG}`占位符来定义容器图像，因为我们将为每个部署使用不同的标签。我们可以放入类似`latest`的东西，但是这样 Kubernetes 就看不到部署之间的区别，而且我们会引入关于我们想要部署什么的模糊性。相反，我更喜欢使用占位符和`envsubst`在部署期间用实际值替换占位符。

我们需要的另一个东西是 Kubernetes 服务，一个能够从外部访问我们的 API 的负载平衡器。所以，让我们在`k8s/deployment.yml` 后面加上以下内容

```
**---
kind:** Service
**apiVersion:** v1
**metadata:
  name:** news
**spec:
  type:** LoadBalancer
  **selector:
    app:** news
  **ports:** - **protocol:** TCP
    **port:** 80
    **targetPort:** 8080
```

它定义了负载平衡器应该如何发现目标 pods 以及使用哪些端口。

好了，就这样，我们准备将应用程序部署到集群中。让我们在 Makefile 中创建最后一个目标

```
deploy:
   envsubst < k8s/deployment.yml | kubectl apply -f -
```

在这一步中，我们将使用`envsubst`将 YAML 中的占位符替换为实际值，然后使用`kubectl apply`将更改应用到集群。

最后，让我们通过运行所有步骤来部署应用程序

```
make test pack upload deploy
```

或者，我们可以定义另一个目标来运行所有步骤，这样您就不需要输入太多内容

```
ship: test pack upload deploy
```

有了它，我们只需运行

```
make ship
```

在第一次部署之后，您需要等待 Google Cloud(或其他 Kubernetes 提供商)来创建您的负载平衡器。运行`kubectl get service`获取负载平衡器外部 IP。然后，在浏览器中输入这个 IP，您应该会看到带有应用程序版本的 JSON。

在 [Github](https://github.com/skolodyazhnyy/30-seconds-deployment/blob/master/Makefile) 查看完整的 Makefile。

好了，现在，让我们计时:)

```
$ time make ship...
real    0m23.103s
user    0m3.622s
sys     0m2.087s
```

利润！我们实际上在不到 30 秒的时间内部署了新版本的应用程序。当然，随着应用程序的增长，您会添加更多的测试，事情会变得更加复杂，部署时间也会增加。但如果你问我，这仍然是一个很好的开始。