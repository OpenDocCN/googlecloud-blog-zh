# 使用 Google Kubernetes 引擎在可抢占节点上提供有状态服务

> 原文：<https://medium.com/google-cloud/stateful-services-on-preemptible-nodes-with-google-kubernetes-engine-c73ac10b8253?source=collection_archive---------0----------------------->

在谷歌 Kubernetes 引擎(GKE)上运行 Kubernetes 工作负载有很多[的好处。由于 Google 对 Kubernetes 开源项目的重大贡献，以及在 Google 内部(通过 Borg)大量使用 Google 版本的 Kubernetes，Google 总体上处于运行和编排容器的前沿，尤其是在公共云中。](https://sadasystems.com/blog/google-microservices-gcp)

通过 GKE 运行 Kubernetes 工作负载的一些优势包括在 GKE 上运行的微服务和通过[谷歌云平台](https://sadasystems.com/cloud-solutions/google-cloud/google-cloud-platform) (GCP)提供的其他服务之间的集成。通过 GKE 利用谷歌的数据仓库技术(如 BigQuery)相对简单，无需担心数据吞吐量或身份验证的细节，因为其中大部分完全由谷歌管理。只需点击几下鼠标，即可轻松使用 GPU 和 TPUs 构建集群。

由于允许 Kubernetes，更具体地说是 GKE，管理容器及其底层节点的编排的好处，使用谷歌的可抢占节点的吸引力变得更加明显。尽管可抢占的节点可能在任何时候死亡(一旦谷歌通过`SIGTERM`通知)，但它们可以比传统的长期运行的节点节省大约 70%的成本。由于 Google 管理容器*和*节点的扩展，节点的死亡不会对我们的微服务或应用程序整体产生影响。

这对大多数微服务来说是很棒的，但是在某些情况下，微服务需要维护状态。即使有状态是暂时的，并且状态最终被卸载，这对于具有可抢占实例的工作负载来说也是麻烦的。

由于 GKE 和可抢占节点的性质，一旦一个节点被安排删除，一个节点接收一个`SIGTERM`，但是底层的 pod 永远不知道它将要死亡，直到它实际上被终止。同样，对于无状态服务，这不会引起任何问题，因为 GKE 只是旋转新的节点，pods 可以在这些节点上进行调度。

在有状态服务的情况下，我们需要考虑节点的终止，并以优雅的方式管理它们的关闭。

为了解决这个问题，我们需要考虑一些需求。

*   一个节点必须通知在其上运行的 pods，它将有足够的时间来保存或卸载状态
*   我们希望遵循最小特权的原则，以确保一个节点只有能力去做，而没有任何其他不希望发生的事情，如果我们授予太多的权限给 T1
*   必须在启动时将`kubectl`代理与 GKE 主节点通信所需的权限授予每个 GKE 节点(以便它可以运行`kubectl drain `hostname``
*   这些节点配有安全的 CoreOS 操作系统。因此，`gcloud`没有被安装，也不可能被安装。事实上，出于安全原因，几乎没有任何东西可以安装在节点上。
*   我们希望尽可能保持集中管理和简单

我们认为最好的方法是开发一个解决方案，通过 Google Compute Engine (GCE)启动脚本管理所需配置文件向每个 GKE 节点的交付。我们还将通过 GCE 关闭脚本在每个 GKE 节点上包含`kubectl drain `hostname``来确保 pod 正常关闭。

要使这个解决方案起作用，需要做一些准备工作。我们首先必须创建一个 Kubernetes 角色，该角色拥有清空节点所需的权限，没有其他权限。

```
vim drain-role.yamlapiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
 name: **system:node-drainer**
rules:
- apiGroups:
 - ""
 resources:
 - pods/eviction
 verbs:
 - create- apiGroups:
 - apps
 resources:
 - statefulsets
 verbs:
 - get- apiGroups:
 - extensions
 resources:
 - daemonsets
 - replicasets
 verbs:
 - get- apiGroups:
 - batch
 resources:
 - jobs
 verbs:
 - get- apiGroups:
 - ""
 resources:
 - nodes
 verbs:
 - get
 - patch- apiGroups:
 - ""
 resources:
 - pods
 verbs:
 - list
```

接下来，我们需要在`drain-user.yaml`中建立一个 Kubernetes 服务帐户用户

```
apiVersion: v1
kind: ServiceAccount
metadata:
 name: **drain-user**
 namespace: default
```

然后我们需要确保通过`drain-binding.yaml`将这个角色绑定到这个用户

```
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
 name: **drain-user**
roleRef:
 apiGroup: rbac.authorization.k8s.io
 kind: ClusterRole
 name: **system:node-drainer**
subjects:
- kind: ServiceAccount
 name: **drain-user**
 namespace: default
```

然后，我们需要创建我们的 GKE 集群，并获得对它的访问凭据。

```
gcloud container clusters create [CLUSTER_NAME]
gcloud container clusters get-credentials [CLUSTER_NAME]
```

这将在`~/.kube/config`创建一个配置文件，我们可以用它来使用`kubectl`。我们应该注意，这个配置应该受到保护，因为它包含作为管理员与 GKE 通信的凭证。

既然我们已经获得了对 GKE 主服务器执行任何操作的权限，我们将应用之前创建的 YAMLs，并建立和配置新的服务帐户用户。

```
kubectl create -f drain-role.yaml
kubectl create -f drain-user.yaml
kubectl create -f drain-binding.yaml
```

现在我们需要为我们的新服务帐户“drain-user”获取令牌。

`kubectl describe serviceAccounts **drain-user**`

这产生了:

```
Name:         **drain-user**
Namespace:    default
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"v1","kind":"ServiceAccount","metadata":{"annotations":{},"name":"drain-user","namespace":"default"}}Image pull secrets:  <none>Mountable secrets:   drain-user-token-z9kr6Tokens:              **drain-user-token-z9kr6**Events:  <none>
```

在标记名中添加粗体以示强调。我们需要它用于下一个命令，该命令实际上获取令牌本身。同样，必须注意的是，这个令牌是向您的服务帐户授予权限的。你要确保它的安全。

`kubectl describe secrets **drain-user-token-z9kr6**`

我们现在需要看看由`gcloud`在`~/.kube/config`下创建的文件。我们应该从制作一个副本开始(因为我们不想损坏这个文件并冒失去访问 GKE 母版的风险)。然后，我们希望删除 users 部分，并用新获得的服务帐户用户令牌替换它。

我们最终的`config`文件应该如下所示(秘密编辑):

```
apiVersion: v1
clusters:
- cluster:
   certificate-authority-data: **REDACTED** # from the original config file
   server: https://**REDACTED** # from the original config file
 name: **CLUSTER_NAME** # from the original config file
contexts:
- context:
   cluster: **CLUSTER_NAME** # from the original config file
   user: **CLUSTER_NAME** # from the original config file
 name: **CLUSTER_NAME** # from the original config file
current-context: **CLUSTER_NAME** # from the original config file
kind: Config
preferences: {}
users:
- name: **CLUSTER_NAME** # from the original config file
 user:
   token: **REDACTED** # from our previous command
```

确保这个文件放在`~/.kube/config`中，这里的主目录是运行`kubectl`的用户的目录。

我们现在需要基于 GKE 创建的初始集群创建一个新的 GCE 实例组，并确保修改启动脚本，以便在启动时也将这个配置文件放置到位。

首先，我们需要获得 GKE 集群正在使用的实例模板。

`gcloud container clusters describe [CLUSTER_NAME] -z [CLUSTER_ZONE]`

我们需要从这个输出中提取实例模板名称(为了强调，添加了粗体)，并在下面的命令中使用它来获取启动和关闭脚本的内容。复制这些脚本至关重要，因为它们为节点与其 GKE 主机对话奠定了基础。

```
...
  instanceGroupUrls:
    - https://www.googleapis.com/compute/v1/projects/my-gcp-project/zones/us-central1-a/instanceGroupManagers/**my-gke-cluster-group**
...
```

`gcloud compute instance-groups managed describe **my-gke-cluster-group**`

现在，我们得到了用于该组的实例模板。

```
...
instanceGroup: https://www.googleapis.com/compute/v1/projects/my-gcp-project/zones/us-central1-a/instanceGroups/**my-gke-cluster-group**
instanceTemplate: https://www.googleapis.com/compute/v1/projects/my-gcp-project/global/instanceTemplates/**my-gke-cluster-instance-template**
kind: compute#instanceGroupManager
...
```

现在我们终于有了实例模板，我们可以获取元数据，包括启动和关闭脚本，以及其他重要的数据。同样，值得注意的是，这些脚本中有许多非常有价值的秘密，因此，它们必须作为密码或任何其他敏感数据来处理。

```
gcloud compute instance-templates describe **my-gke-cluster-instance-template**
```

这给了我们:

```
...
  metadata:
    fingerprint: **REDACTED**
    items:
    - key: cluster-location
      value: us-central1-a
    - key: kube-env
      value: **REDACTED**
    - key: google-compute-enable-pcid
      value: 'true'
    - key: user-data
      value: **REDACTED**
    - key: gci-update-strategy
      value: update_disabled
    - key: gci-ensure-gke-docker
      value: 'true'
    - key: configure-sh
      value: **REDACTED**
    - key: cluster-name
      value: **REDACTED**
    kind: compute#metadata
...
```

现在我们已经有了所有需要的信息，我们可以创建新的 GCE 实例组，它将包括可抢占的实例、与 GKE 主机通信所需的所有配置信息、所有必需的伸缩逻辑(下面举例说明)，以及在`SIGTERM`时从可抢占节点中排出 pod 的能力。

请注意，您也可以将元数据指定为文件，这对于非常大或特殊格式的值(如启动脚本)可能有意义。

我们需要从`configure-sh`键获取输出，并创建一个包含内容的新文件`startup.sh`。然后，我们需要将以下内容添加到该文件的末尾。

```
mkdir ~/.kube
touch ~/.kube/config
echo “**CONTENTS_OF_KUBE_CONFIG**” > ~/.kube/config # this should be the actual config file
```

我们还需要创建我们的关机脚本，我们将保存为`shutdown.sh`。

`kubectl drain `hostname` --force --ignore-daemonsets`

现在我们可以将它们放在一个实例模板中。

```
gcloud compute instance-templates create **new-gke-node-template** \
--machine-type=n1-standard-1 \
--**preemptible** \
--image-project=coreos-cloud \
--image-family=coreos-stable \
--metadata cluster-location=us-central1-a,kube-env=**REDACTED**,google-compute-enable-pcid=’true’,user-data=**REDACTED**,gci-update-strategy=update_disabled,gci-ensure-gke-docker=’true’,cluster-name=**REDACTED**
--metadata-from-file=configure-sh=**~/path/to/startup.sh**,shutdown-script=**~/path/to/shutdown.sh**
```

现在，我们有了一个实例模板，它将创建能够与 GKE 主机通信的实例，并在被谷歌抢占时自动清空。我们的最后一步是基于这个实例模板创建托管实例组，这样我们就可以获得我们想要的伸缩和自我修复。

```
gcloud compute instance-groups managed create **my-new-gke-instance-group** \
--size 5 \
--zone us-central1-a \
--template **my-gke-cluster-instance-template**gcloud compute instance-groups managed set-autoscaling **my-new-gke-instance-group** \
--max-num-replicas 40 \
--zone us-central1-a
```

我们现在已经满足了最初的要求。我们有一个可抢占节点的自动扩展实例组，与我们的 Google 管理的 GKE 主机通信。节点在`SIGTERM`(手动或自动)时耗尽自己，每个节点只有耗尽的权限，而没有执行任何其他 Kubernetes 操作的权限。我们的秘密是安全的，只存储在实例模板中(它受到保护 GCP 数据其余部分的相同安全措施的保护)，一切都由 GCP 结构集中管理(尽管通过 Terraform 之类的东西实现自动化会相对简单)。

我们现在能够利用 GKE 提供的所有优势，同时仍然允许在可抢占的节点下实现最佳成本优化，而不会牺牲状态性！