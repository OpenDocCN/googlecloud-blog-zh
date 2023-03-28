# Dockerfile Go 健康检查和 K8s

> 原文：<https://medium.com/google-cloud/dockerfile-go-healthchecks-k8s-9a87d5c5b4cb?source=collection_archive---------1----------------------->

**更新 2018–05–30**:修正了各种错误。抱歉。

直到最近，我在探索谷歌 Trillian 和这个项目在 Kubernetes 和本地使用 Docker Compose 运行 Trillian 时，才遇到 Docker 的 HEALTHCHECK。

在探索最小化容器图像大小的方法时，我碰到了 Docker 的 HEALTHCHECK 需要在容器中有一些东西来执行 healthcheck，通常是`curl`。

在最小容器中，没有`curl`。

> **NB** : `curl`很优秀。这不是批评`curl`。这是对容器图像膨胀的批评。

聪明的人以前考虑过这个问题，这个故事总结了他们的工作和解决方案。下面是埃尔顿·斯通汉姆的“[为什么不用](https://blog.sixeyed.com/docker-healthchecks-why-not-to-use-curl-or-iwr/) `[curl](https://blog.sixeyed.com/docker-healthchecks-why-not-to-use-curl-or-iwr/)` [或者](https://blog.sixeyed.com/docker-healthchecks-why-not-to-use-curl-or-iwr/) `[iwr](https://blog.sixeyed.com/docker-healthchecks-why-not-to-use-curl-or-iwr/)`”

我以前讨论过最小容器图像的好处。在这种情况下，Trillian 是用 Golang 编写的，这提供了使用`[scratch](https://hub.docker.com/_/scratch/)`容器的机会，或者——在这种情况下——我想我会尝试 Google 的`[distroless](https://github.com/GoogleContainerTools/distroless)`图像。这些图片和 busybox 和 alpine 不包括`curl`。

下面是一个带有运行状况检查的 Dockerfile 文件示例:

```
ENV HTTP_PORT=8091HEALTHCHECK --interval=5m --timeout=3s \
  CMD curl -f [http://localhost:$HTTP_PORT/debug/vars](http://localhost:$HTTP_PORT/debug/vars) || exit 1
```

这取决于`curl`，但是对于`curl`的需求是微不足道的。`curl`用于使 HTTP GET。许多工作(间隔、超时、响应处理)都内置在 Dockerfile 的 HEALTHCHECK 功能中。

## Golang 健康检查

所以，如果我们能用更简单的东西(最好是 Golang)来代替`curl`的通用功能，我们就成功了。

参见 Soluto 的[golang-docker-health check-example](https://github.com/Soluto/golang-docker-healthcheck-example)。

没错。

在充分肯定他们的同时，我对他们的解决方案做了一点小小的调整:

您可以由此创建一个静态二进制文件，并将其合并到 Docker 映像中。这是针对 Linux AMD64 的:

```
CGO_ENABLED=0 \
GOOS=linux \
GOARCH=amd64 \
go build -a -tags netgo  healthcheck.go
```

或者，在发行版的情况下，有一个方便的模式结合了 Docker 的多阶段构建。

## 多阶段构建

**在**之前:这个 Dockerfile 生成一个 1.2GB 的图像(需要`curl`)。

```
FROM golang:1.10ENV HTTP_PORT=8091ADD . /go/src/github.com/google/trillian
WORKDIR /go/src/github.com/google/trillianRUN go get ./server/trillian_log_serverENTRYPOINT ["/go/bin/trillian_log_server"]HEALTHCHECK --interval=5m --timeout=3s \
  CMD curl -f [http://localhost:$HTTP_PORT/debug/vars](http://localhost:$HTTP_PORT/debug/vars) || exit 1
```

转换为多阶段，以便我们先构建，然后将二进制文件复制到运行时映像中。我们也必须在这里修复健康检查，因为`curl`要和`distroless.`说再见了

之后**:这个 docker 文件生成一个 45MB 的镜像(并转储`curl`)**

```
FROM golang:1.10 as buildADD . /go/src/github.com/google/trillian
WORKDIR /go/src/github.com/google/trillianRUN go get ./server/trillian_log_server
RUN go get ./healthcheckFROM gcr.io/distroless/base as runtimeCOPY --from=build /go/bin/trillian_log_server /
COPY --from=build /go/bin/healthcheck /ENTRYPOINT ["/trillian_log_server"]HEALTHCHECK \
  --interval=5m \
  --timeout=3s \
  CMD ["/healthcheck","[http://localhost:8091/debug/vars](http://localhost:$HTTP_PORT/debug/vars)"]
```

好吧，让我们来看看这个:

我们需要一种从一个步骤引用另一个步骤的方法，而`as [name]`就是我们如何做的。我们将构建步骤称为… `build`:

`FROM golang:1.10 **as build**`

我作弊，把 Golang `healthcheck`直接加到了 Trillian repo 的克隆里。这意味着`healthcheck`现在是由`ADD`命令添加的`Trillian`树的一部分，这意味着我可以这样做来构建它，在本例中是`trillian_log_server`:

`RUN go get ./healthcheck`

好，然后我们进入`distroless`容器。即使它没有被使用，它也被命名为`runtime`:

`FROM gcr.io/distroless/base as runtime`

> **NB** 在`distroless`下有很多运行时的图像，在这里查看列表[。有了 Golang，我们不需要太多，所以我们可以使用发行版的基本映像。](https://github.com/GoogleContainerTools/distroless)

现在我们需要使用`build`参考。我们需要将二进制文件从`build`映像复制到`runtime`映像中。对此我们使用`COPY — from-build`。每个二进制文件一个。服务器和运行状况检查:

```
**COPY --from=build** /go/bin/trillian_log_server /
**COPY --from=build** /go/bin/healthcheck /
```

不幸的是(而且我不明白这是为什么——有人吗？)在这个版本中不能使用环境变量(`${HTTP_PORT}`)。出于某种原因，它就是不工作:-(。Docker 在图像中的环境变量映射*总是*具有挑战性。

我们为容器定义入口点，以运行服务器。

我们用显式端口映射来定义我们的健康检查:

```
HEALTHCHECK \
  --interval=5m \
  --timeout=3s \
  CMD [**"/healthcheck"**,"[http://localhost:8091/debug/vars](http://localhost:$HTTP_PORT/debug/vars)"]
```

健康检查现在指向 Golang 二进制文件(`/healthcheck`)和我们指定的端点(`/debug/vars`),最重要的是，没有额外的依赖或工具。两个小 Golang 二进制文件。

如果我们要在本地部署这样一个容器，Docker 的 CLI 将用健康检查的状态来丰富例如容器列表:

```
docker container ls
CONTAINER ID        COMMAND               STATUS
**1c64**d39fbb9a        /trillian_log_server  Up 15 seconds (healthy)
```

您可以列举运行状况检查:

```
docker inspect --format='{{json .State.Health}}' **1c64** \
| jq .
{
  "Status": "healthy",
  "FailingStreak": 0,
  "Log": [
    {
      "Start": "2018-05-30T00:00:00.000000000-07:00",
      "End": "2018-05-30T00:00:00.000000000-07:00",
      "ExitCode": 0,
      "Output": "[http://localhost:](http://localhost:8080/healthz\n)[8091/debug/vars](http://localhost:$HTTP_PORT/debug/vars)"
    },
    {
      "Start": "2018-05-30T00:00:00.000000000-07:00",
      "End": "2018-05-30T00:00:00.000000000-07:00",
      "ExitCode": 0,
      "Output": "[http://localhost:](http://localhost:8080/healthz\n)[8091/debug/vars](http://localhost:$HTTP_PORT/debug/vars)"
    }
  ]
}
```

有了 Trillian，这个映像可以部署到 Kubernetes。您可以获取它的 Pod(名称)，将其转发到端口并测试 healthcheck 端点:

```
kubectl get podsNAME                                             READY     STATUS
trillian-etcd-cluster-9hfhkxxcsf                 1/1       Running
trillian-etcd-cluster-gbz4mfqgvr                 1/1       Running
trillian-etcd-cluster-z268b5z9h2                 1/1       Running
trillian-etcd-operator-5bfd8fc6db-s9r2b          1/1       Running
trillian-logserver-deployment-546d8bd546-7l77l   2/2       Running
trillian-logserver-deployment-546d8bd546-g8gdw   2/2       Running
trillian-logserver-deployment-546d8bd546-pnlfr   2/2       Running
trillian-logserver-deployment-546d8bd546-z6n74   2/2       Running
trillian-logsigner-deployment-5548878bbd-knlvr   2/2       Running
trillian-logsigner-deployment-5548878bbd-xfph8   2/2       Running
```

任意抓取`trillian-logserver-deployment-546d8bd546-pnlfr`:

```
kubectl port-forward \
trillian-logserver-deployment-546d8bd546-pnlfr \
8091:8091Forwarding from 127.0.0.1:8091 -> 8091
Forwarding from [::1]:8091 -> 8091
```

然后:

```
curl localhost:8091/debug/varsok
```

## 超级网络

这让我们想到了第二个建议，当使用 Docker 和 Kubernetes 时，这个建议变得更加一致。

> **题外话** : Kubernetes 提供[活跃度和就绪性探测](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/)。这些适用于容器级别。豆荚聚合容器。因此，pod 可以针对每个容器进行特定的活性和就绪性探测。活性对应于健康检查(容器是否在运行),而就绪性——顾名思义——是确定容器是否准备好接受流量；容器中的服务器可以启动并暴露端点(活跃度)，但是在它准备好接受流量(准备就绪)之前可能需要额外的时间。

Kubernetes 活跃度探测器经常模仿 Docker 健康检查并使用`curl`，例如:

```
livenessProbe:
  exec:
    command:
    - **curl**
    - --fail
    - http://localhost:8091/debug/vars
  failureThreshold: 3
  periodSeconds: 30
```

但是，我们已经转储了`curl`，所以我们也不能在这里使用它。

我们可以用我们的 Golang `/healthcheck`代替`curl`，但是我们不需要这样做。Kubernetes 的开发人员意识到大多数健康检查都是简单的 HTTP GETs，所以…我们可以这样做:

```
livenessProbe:
  **httpGet**:
    path: /debug/vars
    port: 8091
  failureThreshold: 3
  periodSeconds: 30
  timeoutSeconds: 5
```

## 结论

最小化容器图像范围(以及大小)是一件好事。做好一件事是一个好原则，因为它的推论是，如果你做太多事情，你会做得最差。在容器映像中，太多的东西意味着更多的失败机会，更多的软件污染，更多的能源浪费，*而且*特别重要的是:更多的安全问题。

即使我们的容器映像中唯一无关的工具是`curl`，每次`curl`被修补，我们都需要重新构建使用它的每个容器映像。

从您的容器映像中排除可能有一天会用到的装满二进制文件的车库有很多好处，但是，有时也需要一个有用的工具。这篇文章提供了一种简单的方法来取代作为健康检查工具的`curl`。

随时欢迎反馈。

仅此而已！