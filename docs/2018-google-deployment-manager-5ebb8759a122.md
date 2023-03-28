# 使用部署管理器将 GCP 基础设施作为代码

> 原文：<https://medium.com/google-cloud/2018-google-deployment-manager-5ebb8759a122?source=collection_archive---------0----------------------->

![](img/92e647ca7cb64b4f9c61adf6e8b05d88.png)

基础设施作为代码是一种实践，它通过使用代码来描述基础设施，使基础设施的配置可复制、可伸缩且易于审查。基础设施即代码源于对基础设施也是“软件”的认识，对于公共云来说尤其如此。因为它是软件，所以也应该进行版本控制、测试和评审。更好的是，您可以开始将所需基础设施的描述与应用程序的代码放在一起，从而在一个地方获得所需的所有内容的完整定义。您可以开始一起测试和部署一切。

有许多工具可以帮助你实现这一点，例如 Ansible 和 Terraform，但它们并不总是能够提供与公共云托管平台(如 AWS、Azure 或 Google Cloud Platform)完全匹配的服务。考虑到新功能推出的惊人速度，这并不奇怪。这就是为什么一些云提供商也给你他们自己的基础设施作为代码工具，作为他们产品的一部分。AWS 有“亚马逊云形成”，Azure 有“Azure 资源管理器”，对于谷歌云有“部署管理器”。它们都提供了一种使用声明性语言来定义云基础设施资源的方法，例如，您可以将其放入 git 存储库中。此外，您可以使用模板，这允许您避免代码重复，并使得在不同的部署阶段测试相同的基础设施代码成为可能。

在本文中，我们将重点关注 Google 云平台，以及如何使用 Google Cloud Deployment Manager 来自动化所有 GCP 资源的配置。作为实际演示，我们将部署一个 Kubernetes 集群和一个在其上运行的简单应用程序，所有这些都是自动创建的。它包括以下资源:

*   一个 Kubernetes 集群(Google Kubernetes 引擎)
*   Kubernetes 部署
*   Kubernetes 服务
*   Kubernetes 入口定义

## 部署管理器基础

在我们开始使用部署管理器创建 GCP 资源之前，让我们先快速总结一下它是如何工作的。

部署管理器有一个*部署*的概念，它是 GCP 资源的集合，这些资源形成一个逻辑单元并一起部署。部署的资源可以是 GCP 上任何可用的资源:虚拟机、IP 地址、数据库服务器、Kubernetes 集群等。

为了创建部署，您需要一个*部署配置*，它是一个包含资源定义的 YAML 文件。为了让您有个概念，它可能如下所示:

```
resources:
- name: example-vm
  type: compute.v1.instance
  properties:
    zone: europe-west1-b
    machineType: zones/europe-west1-b/machineTypes/n1-standard-1
    disks:
       ...
```

每个列出的资源总是有一个*类型*，这是将要创建的资源的种类(VM、IP 地址等)。)，一个*名称*，和*属性*，描述应该用什么参数创建资源。

与 Ansible 相比，一个重要的概念是，当您创建一个部署时，它作为资源本身存在于 GCP 中。如果您稍后更改了部署的配置(例如，通过修改 YAML 文件)，并运行更新命令，部署管理器会将新配置与其之前部署的配置进行比较，并且只进行所需的更改。不仅需要做的工作更少，更重要的是，它还确保了从配置中删除的任何资源也将从 GCP 基础架构中删除。

您可以在 GitHub 上的[Google Deployment Manager samples](https://github.com/GoogleCloudPlatform/deploymentmanager-samples)项目中找到许多部署配置示例。它通常比[官方文档](https://cloud.google.com/deployment-manager/docs/)更有用，可以找出你应该定义什么参数。

## 准备工作(如果你想执行代码)

如果您想了解部署管理器是如何工作的，我建议您在阅读时也尝试部署这个示例应用程序。您将会看到，部署所有这些组件是多么的快速和简单。通常需要几天才能设置好的东西，现在只需要几分钟。如果您想这样做，请做如下准备:

首先，确保您有一个 GCP 帐户和一个启用了计费的 GCP 项目。此外，您需要安装[谷歌云 SDK](https://cloud.google.com/sdk/) ，其中包括我们将使用的“gcloud”命令行工具。

```
$ gcloud auth login
$ gcloud config set project $MYPROJECT_ID
```

接下来，克隆我用本文中的所有代码示例准备的 github repo:

```
$ git clone https://github.com/schweikert/gcp-infra-as-code
$ cd gcp-infra-as-code
```

## 创建 Kubernetes 集群

我们要做的第一件事是创建一个 Kubernetes 集群。在[支持的资源列表](https://cloud.google.com/deployment-manager/docs/configuration/supported-resource-types)中可以发现，部署管理器中资源的名称是“container.v1.cluster”。一个非常简单的定义如下:

```
resources:

- name: cluster
  type: container.v1.cluster
  properties:
    zone: europe-west1-b
    cluster:
      description: "My example cluster"
      initialNodeCount: 2
```

这将在具有两个节点的单个区域中创建一个 Kubernetes 集群。

github 存储库中的 [cluster.yaml](https://github.com/schweikert/gcp-infra-as-code/blob/master/cluster-1/cluster.yaml) 文件指定了更多的参数，比如跨两个区域分布节点，并为集群节点启用“自动升级”。

您可以使用“create”命令实例化此部署:

```
$ gcloud deployment-manager deployments create example-cluster --config cluster-1/cluster.yaml
```

这将使用 cluster-1/cluster.yaml 中的定义创建一个名为“example-cluster”的部署。您可以稍后更新 yaml 文件，并使用“update”命令更新已部署的部署:

```
$ gcloud deployment-manager deployments update example-cluster --config cluster-1/cluster.yaml
```

该命令将已部署的部署与新定义进行比较，并仅执行所需的更改(例如，创建您添加的新资源)。

现在，删除集群。我们稍后将再次重新创建它。

```
$ gcloud deployment-manager deployments delete example-cluster
```

## 模板

这些 yaml 文件实际上是您的基础设施“代码”，您可能希望在部署之前先测试这些代码。为了有效地实现这一点，您应该为测试和生产部署相同的代码，即使这些环境可能需要不同地指定一些参数。

这就是模板非常有用的地方。您可以将一个模板用于所有环境，只需参数化需要不同的内容。部署管理器支持 Python 脚本和 Jinja 2 作为模板语言。我将使用 Jinja 2 演示模板化，因为尽管 Python 是 Google 推荐的，但我发现 Jinja 2 更适合这个用例，而且更容易阅读。

“cluster-2”目录包含相同的示例，但分为两个文件。

[资源定义](https://github.com/schweikert/gcp-infra-as-code/blob/master/cluster-2/example-cluster.yaml):

```
imports:
- path: templates/cluster.jinja
  name: cluster.jinja

resources:
- name: example-cluster
  type: cluster.jinja
  properties:
    description: "Example Cluster"
    zones:
    - europe-west3-b  
    - europe-west3-c
    initialNodeCount: 1
```

这个文件的结构与非模板版本非常相似，但是它不是直接实例化 GCP 资源，而是实例化一个模板。该模板实例还有一个名称和属性，可以在模板定义中使用。

[模板](https://github.com/schweikert/gcp-infra-as-code/blob/master/cluster-2/templates/cluster.jinja):

```
resources:

- name: {{ env['name'] }}
  type: container.v1.cluster
  properties:
    zone: {{ properties['zones'][0] }}
    ...
```

使用 Jinja 2 语法，模板实例名和属性被用来参数化它。

创建部署的命令与之前相同:

```
$ gcloud deployment-manager deployments create example-cluster --config cluster-2/example-cluster.yaml
```

## 部署经理和第三方资源

Deployment Manager 的一个惊人的能力是，除了它已经知道的以外，您可以教它管理您想要的任何类型的资源。如果管理这些资源的 API 满足[的一些标准](https://cloud.google.com/deployment-manager/docs/configuration/type-providers/process-adding-api)，那么您可以添加该 API 并定义第三方资源作为您的部署管理器部署的一部分。

一个非常有趣的用例是 Kubernetes 资源的管理。部署管理器并不知道如何创建、修改或删除 Kubernetes 集群的 Kubernetes 资源，但是您可以添加代码来实现这一点。

Kubernetes 资源是通过与 Kubernetes 的交互来管理的，这种交互是通过一个定义良好的版本化 API 来实现的。为了使它同时具有可扩展性和向后兼容性，有多个 API 端点来反映托管资源的范围和成熟度。例如，“服务”是核心功能的一部分，被认为是成熟的，因为这种管理服务资源的方法是 API 端点“/api/v1”的一部分。其他最近添加的资源类型可以在其他端点找到:“部署”被认为是测试版，由 API 端点“/API/extensions/v1 beta 1”管理。

对于这些 API 端点中的每一个，我们将需要为部署管理器添加指令，以便它知道如何使用它。所需的代码通常作为集群部署的一部分进行部署，这样一旦创建了集群，就可以开始管理其中的 Kubernetes 资源。

“cluster-3”示例包含一个额外的部分，它将使部署 Kubernetes 资源成为可能。如果您查看“cluster-3/templates/cluster . jinja”文件，您会发现:

```
{% set K8S_ENDPOINTS = {
      '': 'api/v1',
      '-apps': 'apis/apps/v1',
      '-rbac': 'apis/rbac.authorization.k8s.io/v1',
      '-v1beta1-extensions': 'apis/extensions/v1beta1'
} %}

...

{% for typeSuffix, endpoint in K8S_ENDPOINTS.iteritems() %}
- name: {{ env['name'] }}-type{{ typeSuffix }}
  type: deploymentmanager.v2beta.typeProvider
  properties:
    ...
    descriptorUrl: https://$(ref.{{ env['name'] }}.endpoint)/swaggerapi/{{ endpoint }}
{% endfor %}
```

我们不会深入细节，但是您可以看到它使用 Jinja 循环为我们想要使用的每个 API 端点创建定义。它使用 Swagger 或 OpenAPI URL 来发现每个端点上有哪些可用的方法，以及如何调用它们。模板中循环的第一次迭代将产生这样的结果:

```
- name: example-cluster-type
  type: deploymentmanager.v2beta.typeProvider
  properties:
    descriptorUrl: https://$(ref.example-cluster.endpoint)/swaggerapi/api/v1
    ...
```

`$(ref.example-cluster.endpoint)`是内联引用，在部署时解析为已创建资源(“示例集群”)的属性(“端点”)。在这种情况下，它解析为 Kubernetes 主服务的 IP 地址。它还设置了这两个资源(`example-cluster-type`和`example-cluster`)之间的依赖关系，部署管理器将确保资源以正确的顺序创建。

`descriptorUrl`类似于:`https://35.198.131.165/swaggerapi/api/v1`，用于解析 Kubernetes 的 api/v1 "方法。该示例包含另外三个 API 端点，您可以根据需要对其进行扩展(例如，根据您的部署需要访问的 API)。

您可以在同一个部署中开始使用这些新资源类型，也可以在同一项目中创建的新部署中开始使用。

部署集群后，您可以在 GCP 控制台(注意，有一个专门用于部署管理器的部分)或使用 gcloud 工具看到新的类型:

```
$ gcloud deployment-manager types list | grep example
example-cluster-type
example-cluster-type-apps
example-cluster-type-rbac
example-cluster-type-v1beta1-extensions
```

## Kubernetes 应用程序

现在我们终于可以使用部署管理器创建 Kubernetes 应用程序了。我们将定义三个资源:一个 Kubernetes“部署”,带有一个 pod 规范、一个服务和一个入口，使应用程序可以从集群外部访问。

正如您在[模板文件](https://github.com/schweikert/gcp-infra-as-code/blob/master/hello-world-1/templates/hello-world.jinja)中看到的，每个 Kubernetes 资源都是通过引用上述基于 API 的类型之一创建的，我们将其定义为集群部署的一部分。例如:

```
- name: example-hello-world-svc
  type: env[‘project’]/example-cluster-type:/api/v1/namespaces/{namespace}/services
  properties:
    apiVersion: v1
    kind: Service
```

一些宏用于使编写资源(名称前缀、集群类型等)变得更容易。).

您可以使用 3 个 Kubernetes 资源部署模板，如下所示:

```
$ gcloud deployment-manager deployments update example-hello-world --config hello-world-1/example.yaml
```

GCP 将自动分配一个短暂的静态 IP 地址来访问您的应用程序。您可以使用 kubectl 命令找出它是什么，但是首先您需要获得它所需的凭证:

```
$ gcloud container clusters get-credentials example-cluster --zone europe-west1-b

$ kubectl get pod                              
NAME                                   READY     STATUS    RESTARTS   AGE                                                
hello-world-example-75d79ccdd5-pmjcg   1/1       Running   0          1m
hello-world-example-75d79ccdd5-vtb6g   1/1       Running   0          1m
```

现在，您可以查看入口资源，找出外部 IP 地址:

```
$ kubectl get ingress -o wide
NAME                  HOSTS     ADDRESS         PORTS     AGE
hello-world-example   *         35.201.65.191   80        10m
```

…并在此访问您的申请:【http://35.201.65.191 。

瞧，一个完全可复制的 Kubernetes 集群和应用程序，描述为代码:)

要继续使用部署管理器，您可以查看一下 Google KMS，它允许您使用存储在 GCP 的密钥来加密秘密(这样您就可以将加密的值放在您的配置文件中)，并使用该密钥生成 Kubernetes 秘密。我已经测试过并且运行良好的另一个方法是创建一个 PostgreSQL 数据库，并使用通过部署管理器生成的服务帐户部署云 SQL 代理。让我知道你是否有兴趣阅读这方面的文章。

完成实验后，不要忘记清理一切，这样你就不会浪费 GCP 学分。