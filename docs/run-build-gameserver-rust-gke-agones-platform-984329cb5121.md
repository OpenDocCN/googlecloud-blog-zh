# 运行并构建游戏服务器(Rust)- GKE+Agones 平台

> 原文：<https://medium.com/google-cloud/run-build-gameserver-rust-gke-agones-platform-984329cb5121?source=collection_archive---------3----------------------->

Agones 是一个在 [Kubernetes](https://kubernetes.io/) 上托管、运行和扩展[专用游戏服务器](https://en.wikipedia.org/wiki/Game_server#Dedicated_server)的库。它是一个开源平台，用于部署、托管、扩展和协调大型多人游戏的专用游戏服务器，构建在行业标准的分布式系统平台 [Kubernetes](https://kubernetes.io/) 之上。它用可以利用和共同开发的开源解决方案取代了定制或专有的集群管理和游戏服务器扩展解决方案，因此您可以专注于构建多人游戏的重要方面，而不是开发支持它的基础架构。

Agones 在构建时考虑到了云和内部基础架构，可以根据需要调整其策略以进行设备管理、自动扩展等，从而确保用于托管专用游戏服务器的资源对于其所处的环境来说是成本最优的。

这篇博文将讲述如何在一个简单的 Rust 游戏服务器中使用 Agones Rust SDK。我们将利用谷歌云平台(谷歌 kubernetes 引擎)上安装的 Agones 管理 Kubernetes 产品。

下面是在您开始使用 Rust SDK for Agones 构建和运行一个简单的游戏服务器之前的一些先决条件。

1.  [码头工人](https://www.docker.com/get-started/)
2.  Agones 安装在 GKE
3.  正确配置 kubectl
4.  Agones 存储库的本地副本
5.  Docker 映像的存储库，如 [Docker Hub](https://hub.docker.com/) 或 [GC 容器注册表](https://cloud.google.com/container-registry/)

按照以下步骤为您的 Agones 安装创建一个 [Google Kubernetes 引擎(GKE)](https://cloud.google.com/kubernetes-engine/) 集群。

# 开始之前

采取以下步骤来启用 Kubernetes 引擎 API:

1.  访问谷歌云平台控制台中的 [Kubernetes 引擎](https://console.cloud.google.com/kubernetes/list)页面。
2.  创建或选择一个项目。
3.  等待 API 和相关服务被启用。这可能需要几分钟时间。
4.  [为您的项目启用计费](https://support.google.com/cloud/answer/6293499#enable-billing)。

# 选择外壳

我们可以使用[谷歌云外壳](https://cloud.google.com/shell/)或者本地外壳。

Google Cloud Shell 是一个 Shell 环境，用于管理托管在 Google 云平台(GCP)上的资源。Cloud Shell 预装了`[gcloud](https://cloud.google.com/sdk/gcloud/)`和`[kubectl](https://kubernetes.io/docs/user-guide/kubectl-overview/)`命令行工具。`gcloud`为 GCP 提供主要的命令行接口，而`kubectl`为针对 Kubernetes 集群运行命令提供命令行接口。

如果您喜欢使用本地 shell，那么您必须在您的环境中安装`gcloud`和`kubectl`命令行工具。

## 云壳

要启动云壳，请执行以下步骤:

1.  进入[谷歌云平台控制台](https://console.cloud.google.com/home/dashboard)。
2.  从控制台的右上角，单击**激活谷歌云外壳**按钮。

![](img/de6606455ffe5a33f2e60532396112da.png)

3.云 Shell 会话在控制台底部的框架中打开。使用这个 shell 来运行`gcloud`和`kubectl`命令。

4.使用以下命令在您的地理区域设置一个计算区域。一个示例计算区域是`us-west1-a`。

```
gcloud config set compute/zone [COMPUTE_ZONE]
```

# 创建防火墙

我们需要一个防火墙来允许 UDP 流量通过端口 7000-8000 到达标记为`game-server`的节点。这些防火墙规则适用于您将在下一节中创建的集群节点。

```
gcloud compute firewall-rules create game-server-firewall \
  --allow udp:7000-8000 \
  --target-tags game-server \
  --description "Firewall to allow game server udp traffic"
```

# 创建集群

一个[集群](https://cloud.google.com/kubernetes-engine/docs/concepts/cluster-architecture)由至少一个*控制平面*机和多个称为*节点*的工作机组成。在 Google Kubernetes 引擎中，节点是运行 Kubernetes 进程的[计算引擎虚拟机](https://cloud.google.com/compute/docs/instances/)实例，这些进程使节点成为集群的一部分。

```
gcloud container clusters create [CLUSTER_NAME] --cluster-version=1.23 \
  --tags=game-server \
  --scopes=gke-default \
  --num-nodes=4 \
  --no-enable-autoupgrade \ 
  --enable-image-streaming \
  --machine-type=e2-standard-4
```

# (可选)创建专用节点池

为要安装的 Agones 资源创建一个[专用节点池](https://cloud.google.com/kubernetes-engine/docs/concepts/node-pools)。如果您跳过这一步，Agones 控制器将与您的游戏服务器共享默认节点池，这对于实验来说很好，但不建议用于生产部署。

```
gcloud container node-pools create agones-system \
  --cluster=[CLUSTER_NAME] \
  --no-enable-autoupgrade \
  --node-taints agones.dev/agones-system=true:NoExecute \
  --node-labels agones.dev/agones-system=true \
  --num-nodes=1
```

# (可选)创建度量节点池

如果您想使用 Prometheus with Grafana 或 Cloud Logging and Monitoring 来监控 Agones 系统，请为[指标创建一个节点池。](https://agones.dev/site/docs/guides/metrics/)

```
gcloud container node-pools create agones-metrics \
  --cluster=[CLUSTER_NAME] \
  --no-enable-autoupgrade \
  --node-taints agones.dev/agones-metrics=true:NoExecute \
  --node-labels agones.dev/agones-metrics=true \
  --num-nodes=1
```

# 设置集群凭据

最后，让我们告诉`gcloud`我们正在与这个集群对话，并获取 auth 凭证供`kubectl`使用。

```
gcloud config set container/cluster [CLUSTER_NAME]
gcloud container clusters get-credentials [CLUSTER_NAME]
```

# 使用舵安装 Agones

使用 [Helm](https://helm.sh/) 软件包管理器在 [Kubernetes](http://kubernetes.io/) 集群上安装 Agones。

# 先决条件

*   [掌舵](https://helm.sh/)包管理器 3.2.3+
*   [支持 Kubernetes 集群](https://agones.dev/site/docs/installation/#usage-requirements)

# 舵 3

使用我们的 stable helm 库安装发布名为`my-release`的图表:

```
helm repo add agones https://agones.dev/chart/stable
helm repo update
helm install my-release --namespace agones-system --create-namespace agones/agones
```

在生产环境中运行时，Agones 应该安排在专用的节点池中，与游戏服务器的安排位置不同，以获得更好的隔离和弹性。默认情况下，Agones 更喜欢被安排在标有`agones.dev/agones-system=true`的节点上，并且容忍节点污染`agones.dev/agones-system=true:NoExecute`。如果没有专用节点可用，Agones 将在常规节点上运行，但不建议生产使用。

# 名称空间

默认情况下，Agones 被配置为与部署在`default`名称空间中的游戏服务器协同工作。如果您计划使用另一个名称空间，您可以通过参数`gameservers.namespaces`配置 Agones。

例如使用`default` **和** `xbox`命名空间:

```
kubectl create namespace xbox
helm install my-release agones/agones --set "gameservers.namespaces={default,xbox}" --namespace agones-system
```

如果您想在升级您的版本后添加一个新的名称空间:

```
kubectl create namespace ps4
helm upgrade my-release agones/agones --reuse-values --set "gameservers.namespaces={default,xbox,ps4}" --namespace agones-system
```

# 卸载图表

要卸载/删除`my-release`部署:

```
helm uninstall my-release --namespace=agones-systemRBAC
```

默认情况下，`agones.rbacEnabled`设置为真。这将在 Agones 中启用 RBAC 支持，并且如果在您的集群中启用了 RBAC，则必须为真。

该图表将负责为 Agones 创建所需的服务帐户和角色。

```
helm install my-release --namespace agones-system \
  --set gameservers.minPort=1000,gameservers.maxPort=5000 agones
```

上述命令将把 Agones 控制器部署到`agones-system`名称空间。此外，Agones 将使用 1000-5000 的动态游戏服务器端口分配范围。或者，可以在安装图表时提供指定参数值的 YAML 文件。举个例子，

```
helm install my-release --namespace agones-system -f values.yaml agones/agones
```

# 舵试验

通过运行以下命令检查 Agones 安装:

```
helm test my-release --cleanup
```

```
RUNNING: agones-test
PASSED: agones-test
```

这个测试将创建一个`GameServer`资源，然后删除它。

# 控制器 TLS 证书

默认情况下，agones chart 生成准入控制器使用的 tls 证书，虽然这很方便，但它需要 agones 控制器在每个`helm upgrade`命令后重新启动。

# 指南

对于大多数用例，控制器无论如何都需要重启(例如:控制器映像更新)。但是，如果您确实需要避免重启，我们建议您关闭 tls 自动生成功能(`agones.controller.generateTLS`到`false`，并提供您自己的证书(`certs/server.crt`、`certs/server.key`)。

# 证书管理器

另一种方法是使用 [cert-manager.io](https://cert-manager.io/) 解决方案进行集群级证书管理。

为了使用证书管理器解决方案，首先[在集群上安装证书管理器](https://cert-manager.io/docs/installation/kubernetes/)。然后，[配置](https://cert-manager.io/docs/configuration/)一个`Issuer` / `ClusterIssuer`资源，最后[配置](https://cert-manager.io/docs/usage/certificate/)一个`Certificate`资源来管理控制器`Secret`。确保根据您的系统要求配置`Certificate`，包括有效性`duration`。

下面是一个使用自签名`ClusterIssuer`配置控制器`Secret`的例子，其中秘密名称为`my-release-cert`或`{{ template "agones.fullname" . }}-cert`:

```
#!/bin/bash
# Create a self-signed ClusterIssuer
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned
spec:
  selfSigned: {}
EOF
```

```
# Create a Certificate with IP for the my-release-cert )
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: my-release-cert
  namespace: agones-system
spec:
  dnsNames:
    - agones-controller-service.agones-system.svc
  secretName: my-release-cert
  issuerRef:
    name: selfsigned
    kind: ClusterIssuer
EOF
```

生成证书后，我们将希望[将 caBundle](https://cert-manager.io/docs/concepts/ca-injector/) 注入控制器 webhook，并通过设置以下内容禁用控制器密码创建:

```
helm install my-release \
  --set agones.controller.disableSecret=true \
  --set agones.controller.customCertSecretPath[0].key='ca.crt',customCertSecretPath[0].path='ca.crt'
  --set agones.controller.customCertSecretPath[1].key='tls.crt',customCertSecretPath[1].path='server.crt'
  --set agones.controller.customCertSecretPath[2].key='tls.key',customCertSecretPath[2].path='server.key'
  --set agones.controller.allocationApiService.annotations={'cert-manager.io/inject-ca-from': 'agones-system/my-release-cert'} \
  --set agones.controller.allocationApiService.disableCaBundle=true \
  --set agones.controller.validatingWebhook.annotations={'cert-manager.io/inject-ca-from': 'agones-system/my-release-cert'} \
  --set agones.controller.validatingWebhook.disableCaBundle=true \
  --set agones.controller.mutatingWebhook.annotations={'cert-manager.io/inject-ca-from': 'agones-system/my-release-cert'} \
  --set agones.controller.mutatingWebhook.disableCaBundle=true \
  --namespace agones-system --create-namespace  \
  agones/agones
```

# 运行简单的游戏服务器

首先，运行预构建版本的简单游戏服务器，并记下创建的名称:

```
kubectl create -f https://raw.githubusercontent.com/googleforgames/agones/release-1.27.0/examples/rust-simple/gameserver.yaml
GAMESERVER_NAME=$(kubectl get gs -o go-template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
```

游戏服务器设置 Agones SDK，调用`sdk.ready()`通知 Agones 准备好提供流量，每 10 秒打印一条消息，然后在一分钟后调用`sdk.shutdown()`表示游戏服务器将要退出。

您可以通过运行以下命令来跟踪游戏服务器的生命周期

```
kubectl logs ${GAMESERVER_NAME} rust-simple -f
```

这将产生类似于

```
Rust Game Server has started!
Creating SDK instance
Setting a label
Starting to watch GameServer updates...
Health ping sent
Setting an annotation
Marking server as ready...
...marked Ready
Getting GameServer details...
GameServer name: rust-simple-txsc6
Running for 0 seconds
GameServer Update, name: rust-simple-txsc6
GameServer Update, state: Scheduled
GameServer Update, name: rust-simple-txsc6
GameServer Update, state: Scheduled
GameServer Update, name: rust-simple-txsc6
GameServer Update, state: RequestReady
GameServer Update, name: rust-simple-txsc6
GameServer Update, state: Ready
Health ping sent
Health ping sent
Health ping sent
Health ping sent
Health ping sent
Running for 10 seconds
GameServer Update, name: rust-simple-txsc6
GameServer Update, state: Ready
...
Shutting down after 60 seconds...
...marked for Shutdown
Running for 60 seconds
Health ping sent
GameServer Update, name: rust-simple-txsc6
GameServer Update, state: Shutdown
GameServer Update, name: rust-simple-txsc6
GameServer Update, state: Shutdown
...
```

如果一切按预期进行，游戏服务器将在大约一分钟后自动退出。

在某些情况下，游戏服务器会进入一种不健康的状态，在这种情况下，它会无限期地重新启动。如果发生这种情况，您可以通过运行

```
kubectl delete gs ${GAMESERVER_NAME}
```

# 构建一个简单的游戏服务器

将目录切换到本地 agones/examples/rust-simple 目录。为了试验 SDK，在你最喜欢的编辑器中打开`main.rs`,通过将分配给`let _health`的线程中的行修改为

```
thread::sleep(Duration::from_secs(20));
```

接下来，通过运行以下命令构建一个新的 docker 映像

```
cd examples/rust-simple
REPOSITORY=<your-repository> # e.g. gcr.io/agones-images
make build-image REPOSITORY=${REPOSITORY}
```

多阶段 Dockerfile 将下拉构建映像所需的所有依赖项。请注意，这通常需要几分钟才能完成。

一旦构建了容器，就将其推送到您的存储库

```
docker push ${REPOSITORY}/rust-simple-server:0.4
```

# 运行定制的游戏服务器

现在是时候将新创建的 gameserver 容器部署到 Agones 集群中了。

首先，您需要编辑`examples/rust-simple/gameserver.yaml`以指向您的新图像:

```
containers:
- name: rust-simple
  image: $(REPOSITORY)/rust-simple-server:0.4
  imagePullPolicy: Always
```

然后，部署您的游戏服务器

```
kubectl create -f gameserver.yaml
GAMESERVER_NAME=$(kubectl get gs -o go-template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
```

同样，遵循游戏服务器的生命周期，运行

```
kubectl logs ${GAMESERVER_NAME} rust-simple -f
```

这将产生类似于

```
Rust Game Server has started!
Creating SDK instance
Setting a label
Starting to watch GameServer updates...
Health ping sent
Setting an annotation
Marking server as ready...
...marked Ready
Getting GameServer details...
GameServer name: rust-simple-z6lz8
Running for 0 seconds
GameServer Update, name: rust-simple-z6lz8
GameServer Update, state: Scheduled
GameServer Update, name: rust-simple-z6lz8
GameServer Update, state: RequestReady
GameServer Update, name: rust-simple-z6lz8
GameServer Update, state: RequestReady
GameServer Update, name: rust-simple-z6lz8
GameServer Update, state: Ready
Running for 10 seconds
GameServer Update, name: rust-simple-z6lz8
GameServer Update, state: Ready
GameServer Update, name: rust-simple-z6lz8
GameServer Update, state: Unhealthy
Health ping sent
Running for 20 seconds
Running for 30 seconds
Health ping sent
Running for 40 seconds
GameServer Update, name: rust-simple-z6lz8
GameServer Update, state: Unhealthy
Running for 50 seconds
Health ping sent
Shutting down after 60 seconds...
...marked for Shutdown
Running for 60 seconds
Running for 70 seconds
GameServer Update, name: rust-simple-z6lz8
GameServer Update, state: Unhealthy
Health ping sent
Running for 80 seconds
Running for 90 seconds
Health ping sent
Rust Game Server finished.
```

随着健康检查时间间隔的延长，游戏服务器会自动被 Agones 标记为`Unhealthy`。

最后，通过手动移除来清理游戏服务器

```
kubectl delete gs ${GAMESERVER_NAME}
```