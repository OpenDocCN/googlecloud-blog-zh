# 交叉平面(IaC)，新成员

> 原文：<https://medium.com/google-cloud/crossplane-iac-new-kid-on-the-block-c9057c859c6a?source=collection_archive---------1----------------------->

在过去，管理 IT 基础架构是一项艰难的工作。系统管理员必须手动管理和配置应用程序运行所需的所有硬件和软件。

然而，近年来，事情发生了巨大的变化。云、敏捷和开发运维实践的进步彻底改变了 IT 基础设施的设计、开发和维护。基础设施即代码(IaC)是这些实践的关键组成部分之一。IaC 使用配置文件管理您的 IT 基础设施，并提供诸如提高速度、效率和一致性等好处。IaC 促进了更快的基础设施部署，因为团队不必维护各个部署环境的设置。

市场上有几种 IaC 解决方案。Terraform 是 IaC 当前的行业领导者，它是一个了不起的工具；但是，您必须精通 Terraform 领域特定语言，如 HashiCorp 配置语言(HCL)。Pulumi 是另一个更面向开发人员的开源 IaC 工具，因为它支持 Python、JavaScript、Java、TypeScript、Go 和. NET 等语言。您可以使用这些语言开发可重用的函数、包、类和 Pulumi 组件。但是在 IaC 市场上有一个新的参与者，cross plane([https://crossplane.io/](https://crossplane.io/))，它遵循 GitOps 的 IaC 方法，我们将在这篇博客中介绍这个新的参与者。

你可能想知道，在混合使用 [Terraform](http://terraform.io/) 、 [Pulumi](http://pulumi.com/) ，甚至 [Ansible](https://www.ansible.com/) 的情况下，我为什么要考虑 Crossplane？

关于 Crossplane，有几件事真正引起了我的兴趣:

*   **坚持基础设施即数据的原则。**这是 Kubernetes-native，因此您使用 Kubernetes 自定义资源(YAML，即文本)以声明方式*供应云基础架构。*
*   **无状态文件。**与 [Terraform](http://terraform.io/) 和 [Pulumi](http://pulumi.com/) 不同，Crossplane 不使用状态文件。作为一个 Kubernetes 运营商，它有一个控制器来协调期望状态和当前状态，利用 etcd 来做到这一点。
*   真理之源。与 [Terraform](http://terraform.io/) 和 [Pulumi](http://pulumi.com/) 不同，如果有人试图做一些偷偷摸摸的事情，比如使用 CLI 或管理控制台更新云资源，Crossplane 不会让你得逞。

# **什么是交叉平面？**

Crossplane 是一个开源的 Kubernetes 插件，可以将您的集群转换成一个通用控制平面。Crossplane 使平台团队能够组装来自多个供应商的基础设施，并为应用程序团队提供更高级别的自助服务 API，而无需编写任何代码。

Crossplane 扩展了您的 Kubernetes 集群，以支持编排任何基础设施或托管服务。

Crossplane 是一个[云本地计算基础](https://www.cncf.io/)项目。

# 概念

Crossplane 引入了多个构建块，使您能够使用 Kubernetes API 供应、组合和消费基础设施。这些单独的概念一起工作，允许在组织中不同角色之间的关注点的强大分离，这意味着团队的每个成员在适当的抽象级别上与 Crossplane 交互。

# 包装

[封装](https://crossplane.io/docs/v1.10/concepts/packages.html)允许扩展交叉板以包含新功能。这通常看起来像捆绑一组 Kubernetes[CRD](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)和[控制器](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#custom-controllers)，它们代表并管理外部基础设施(即提供商)，然后将它们安装到运行 Crossplane 的集群中。交叉板处理确保任何新的 CRD 不与现有的 CRD 冲突，以及管理新包的 RBAC 和安全性。不严格要求包是提供者，但这是目前包最常见的用例。

# 提供者

提供者是使 Crossplane 能够在外部服务上提供基础设施的包。他们带来了与外部基础设施资源一一对应的 CRD(即托管资源)，以及管理这些资源生命周期的控制器。您可以在[提供者文档](https://crossplane.io/docs/v1.10/concepts/providers.html)中阅读更多关于提供者的信息，包括如何安装和配置它们。

# 托管资源

托管资源是 Kubernetes 自定义资源，代表基础设施原语。API 版本为`v1beta1`或更高版本的托管资源支持云提供商为给定资源所做的每一项工作。您可以在[上行市场](https://marketplace.upbound.io/)上找到每个提供商的托管资源及其 API 规范，并在[托管资源文档](https://crossplane.io/docs/v1.10/concepts/managed-resources.html)中了解更多信息。

# 复合资源

复合资源(XR)是一种特殊的定制资源，由`CompositeResourceDefinition`定义。它将一个或多个被管理的资源组成一个更高级别的基础设施单元。复合资源是面向基础设施操作者的，但是也可以提供面向应用程序开发人员的复合资源声明，作为复合资源的代理。您可以在[组合文档](https://crossplane.io/docs/v1.10/concepts/composition.html)中了解更多关于所有这些概念的信息。

# 入门指南

# 选择交叉平面分布

第一次使用 Crossplane 的用户现在有两种选择。第一种方法是使用由社区维护和发布的 Crossplane 版本，该版本可在 [Crossplane GitHub](https://github.com/crossplane/crossplane) 上找到。

第二种选择是使用供应商支持的交叉板分布。这些发行版[已通过 CNCF](https://github.com/cncf/crossplane-conformance) 认证，符合 Crossplane，但可能会包含额外的功能或工具，使其更易于在生产环境中使用。

# 从上游交叉板开始

将 Crossplane 安装到现有的 Kubernetes 集群中需要更多的设置，但是可以为需要它的用户提供更多的灵活性。

# 获取 Kubernetes 集群

1.  **自制 MAC OS**

```
brew upgrade
brew install kind
brew install kubectl
brew install helm
kind create cluster --image kindest/node:v1.23.0 --wait 5m
```

2.**对于 macOS / Linux，使用以下内容:**

*   [Kubernetes 集群](https://kubernetes.io/docs/setup/)
*   [种类](https://kind.sigs.k8s.io/docs/user/quick-start/)
*   [Minikube](https://minikube.sigs.k8s.io/docs/start/) ，最低版本`v0.28+`
*   [舵](https://helm.sh/docs/intro/using_helm/)，最低版本`v3.0.0+`。

# 安装交叉面板

**舵 3 稳定**

使用 Helm 3 安装 Crossplane 的最新官方`stable`版本，适用于社区使用和测试:

```
kubectl create namespace crossplane-system
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update

helm install crossplane --namespace crossplane-system crossplane-stable/crossplane
```

# 检查交叉面板状态

```
helm list -n crossplane-system

kubectl get all -n crossplane-system
```

# 安装交叉面板 CLI

```
curl -sL https://raw.githubusercontent.com/crossplane/crossplane/master/install.sh | sh
```

# 选择入门配置

Crossplane 不仅仅是将基础设施原语建模为定制资源，它使您能够用自己选择的模式定义新的定制资源。我们称这些为“复合资源”(XRs)。复合资源由托管资源组成——Kubernetes 定制资源提供了基础设施原语的高保真表示，如 SQL 实例或防火墙规则。

我们使用两种特殊的交叉面板资源来定义和配置这些新的自定义资源:

*   一个`CompositeResourceDefinition` (XRD) *定义了*一种新的复合资源，包括它的模式。一个 XRD 可以选择*提供*一个声明(XRC)。
*   一个`Composition`指定了复合资源将由哪些资源组成，以及应该如何配置它们。您可以为每个复合资源创建多个`Composition`选项。

XRDs 和 Compositions 可以作为*配置*打包和安装。配置是一个组合配置的[包],可以通过创建一个声明性的`Configuration`资源或者使用`kubectl crossplane install configuration`轻松地安装到 Crossplane。

在下面的例子中，我们将安装一个配置，该配置定义一个新的`XPostgreSQLInstance` XR 和`PostgreSQLInstance` XRC，它们接受单个`storageGB`参数，并创建一个带有`username`、`password`和`endpoint`键的连接`Secret`。对于能够满足一个`PostgreSQLInstance`的每一个提供者都存在一个`Configuration`。我们开始吧！

# 安装配置包

**GCP**

```
kubectl crossplane install configuration registry.upbound.io/xp/getting-started-with-gcp:v1.10.1
```

等待，直到所有程序包都正常运行:

```
watch kubectl get pkg
```

# 获取 GCP 帐户密钥文件

```
# replace this with your own gcp project id and the name of the service account
# that will be created.
PROJECT_ID=my-project
NEW_SA_NAME=test-service-account-name

# create service account
SA="${NEW_SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"
gcloud iam service-accounts create $NEW_SA_NAME --project $PROJECT_ID

# enable cloud API
SERVICE="sqladmin.googleapis.com"
gcloud services enable $SERVICE --project $PROJECT_ID

# grant access to cloud API
ROLE="roles/cloudsql.admin"
gcloud projects add-iam-policy-binding --role="$ROLE" $PROJECT_ID --member "serviceAccount:$SA"

# create service account keyfile
gcloud iam service-accounts keys create creds.json --project $PROJECT_ID --iam-account $SA
```

# 创建提供者机密

```
kubectl create secret generic gcp-creds -n crossplane-system --from-file=creds=./creds.json
```

# 配置提供程序

我们将创建以下`ProviderConfig`对象来配置 GCP 提供者的凭证:

```
# replace this with your own gcp project id
PROJECT_ID=my-project
echo "apiVersion: gcp.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  projectID: ${PROJECT_ID}
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: gcp-creds
      key: creds" | kubectl apply -f -
```

# 供应基础设施

**GCP**

```
apiVersion: database.example.org/v1alpha1
kind: PostgreSQLInstance
metadata:
  name: my-db
  namespace: default
spec:
  parameters:
    storageGB: 20
  compositionSelector:
    matchLabels:
      provider: gcp
  writeConnectionSecretToRef:
    name: db-conn
```

```
kubectl apply -f https://raw.githubusercontent.com/crossplane/crossplane/release-1.10/docs/snippets/compose/claim-gcp.yaml
```

创建了`PostgreSQLInstance`之后，Crossplane 将开始在您选择的提供商上提供一个数据库实例。一旦配置完成，您应该在运行以下命令时在输出中看到`READY: True`:

```
kubectl get postgresqlinstance my-db
```

*   `kubectl get claim`:获取所有索赔种类的所有资源，如`PostgreSQLInstance`。
*   `kubectl get composite`:获取所有复合类的资源，比如`XPostgreSQLInstance`。
*   `kubectl get managed`:获取代表一个外部基础设施单元的所有资源。
*   `kubectl get <name-of-provider>`:获取与`<provider>`相关的所有资源。
*   `kubectl get crossplane`:获取所有与交叉平面相关的资源。

尝试以下命令，查看您调配的资源是否准备就绪:

```
kubectl get crossplane -l crossplane.io/claim-name=my-db
```

一旦您的`PostgreSQLInstance`准备好了，您应该在`default`名称空间中看到一个名为`db-conn`的`Secret`，它包含了我们在 XRD 中定义的键。如果它们是由组合填充的，那么它们应该出现:

```
$ kubectl describe secrets db-conn
Name:         db-conn
Namespace:    default
...

Type:  connection.crossplane.io/v1alpha1

Data
====
password:  27 bytes
port:      4 bytes
username:  25 bytes
endpoint:  45 bytes
```

# 消耗您的基础设施

因为连接秘密是作为 Kubernetes `Secret`编写的，所以它们很容易被 Kubernetes 原语使用。Kubernetes 中最基本的构建模块是`Pod`。让我们定义一个`Pod`,它将显示我们能够连接到新提供的数据库。

```
apiVersion: v1
kind: Pod
metadata:
  name: see-db
  namespace: default
spec:
  containers:
  - name: see-db
    image: postgres:12
    command: ['psql']
    args: ['-c', 'SELECT current_database();']
    env:
    - name: PGDATABASE
      value: postgres
    - name: PGHOST
      valueFrom:
        secretKeyRef:
          name: db-conn
          key: endpoint
    - name: PGUSER
      valueFrom:
        secretKeyRef:
          name: db-conn
          key: username
    - name: PGPASSWORD
      valueFrom:
        secretKeyRef:
          name: db-conn
          key: password
    - name: PGPORT
      valueFrom:
        secretKeyRef:
          name: db-conn
          key: port
```

```
kubectl apply -f https://raw.githubusercontent.com/crossplane/crossplane/release-1.10/docs/snippets/compose/pod.yaml
```

这个`Pod`只是连接到一个 PostgreSQL 数据库并打印它的名称，所以如果您运行`kubectl logs see-db`，您应该在创建它之后看到下面的输出(或类似的内容):

```
current_database
------------------
 postgres
(1 row)
```

# 打扫

要清理`Pod`，运行:

```
kubectl delete pod see-db
```

要清理已调配的基础架构，您可以删除`PostgreSQLInstance` XRC:

```
kubectl delete postgresqlinstance my-db
```

# 结论

与保存状态文件的 [Terraform](http://terraform.io/) 和 [Pulumi](http://pulumi.com/) 不同，Crossplane 允许你在工具之外改变云基础设施(可能会带来毁灭性的后果)，Crossplane 不会让你得逞。

像 [Ansible](https://www.ansible.com/integrations/cloud) 一样，它坚持基础设施即数据的原则，使用 YAML 来声明性地提供云资源。这绝对是我的好书。

Crossplane 出现的时间并不长，它显示了这一点。虽然有很多对 AWS 资源的支持，但相比之下，对 T2 谷歌云的支持却很少。也就是说，我相信随着它在未来的加速发展。