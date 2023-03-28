# 启用 GKE 工作量标识

> 原文：<https://medium.com/google-cloud/enabling-gke-workload-identity-ad5dd586a811?source=collection_archive---------1----------------------->

*原载于*[*【https://pbhadani.com】*](https://pbhadani.com/posts/gke-workload-identity/)

在这篇博客中，我将讨论 GKE 工作负载身份特性以及为什么要使用这个特性。

![](img/44d367ab9ab393c5511dea24b72735bc.png)

# 有什么问题？

在 GKE 上运行的应用程序必须通过身份验证才能使用谷歌服务，如谷歌云存储(GCS)、云 SQL、BigQuery 等。可以通过使用 Kubernetes Secret space 或不同的方法(如 Vault)向应用程序提供服务帐户密钥 JSON 文件来进行身份验证。但是在这些方法中，服务帐户密钥 JSON(有 10 年的寿命)必须以明文形式存储在 pod 中，或者以 base64 编码存储在 Kubernetes 的秘密空间中。还有，钥匙轮换的过程一定是在一个不是好玩的过程的地方。

我们可以通过将服务帐户附加到 Kubernetes 节点来避免使用服务帐户密钥，但是这样一来，在该节点上运行的所有 pods 都获得了相同的权限，这不是一件理想的事情。

# 目标？

我们希望为一个 Pod 分配一个服务帐户，这样我们就可以隔离不同 Pod 的权限。

万岁，我们在测试版中提供了工作负载标识功能，解决了 GKE 的这个问题。

# 那么，什么是工作负载身份呢？

根据 Google 文档，“工作负载身份是从 GKE 访问 Google 云服务的推荐方式，因为它提高了安全性和可管理性。”

GKE 工作负载身份允许我们将服务帐户附加到 Kubernetes pod，并消除了在 pod 或集群中管理服务帐户凭证 JSON 文件的麻烦。

# 让我们在 GKE 集群中使用工作负载标识

# 先决条件

1.  如果你还没有在你的工作站上安装`gcloud`，那么参考我以前的博客[来快速安装和运行它。
    或者，您可以使用](https://pbhadani.com/posts/getting-started-wth-google-cloud-sdk/) [Google Cloud Shell](https://cloud.google.com/shell/) 来运行这些命令。
2.  请确保您是项目编辑者或项目所有者，或者有足够的权限运行以下命令。

# 设置 GKE 集群

按照以下步骤创建新的 GKE 集群并启用工作负载标识。

1.  启用[云 IAM API](https://console.cloud.google.com/apis/api/iamcredentials.googleapis.com/) 。
2.  安装并配置`gke-gcloud-auth-plugin`。

`gke-gcloud-auth-plugin`是 GKE 新的 Kubectl 认证插件。请阅读[文档](https://cloud.google.com/blog/products/containers-kubernetes/kubectl-auth-changes-in-gke)了解更多详情。

*   安装插件

```
gcloud components install gke-gcloud-auth-plugin
```

**注意:**如果`gcloud CLI component manager`被禁用，使用`yum`或`apt`包安装该插件。

对于 Debian:

```
sudo apt-get install google-cloud-sdk-gke-gcloud-auth-plugin
```

*   配置插件

```
echo "export USE_GKE_GCLOUD_AUTH_PLUGIN=True" >> ~/.bashrcsource ~/.bashrc
```

3.设置 GCP 默认值。

*   设置 GCP 项目

```
**export** GCP_PROJECT_ID=<YOUR_GCP_PROJECT_ID>gcloud config **set** project $GCP_PROJECT_ID
```

*   设置默认地区和区域

```
gcloud config **set** compute/region europe-west1gcloud config **set** compute/zone europe-west1-b
```

4.确保你已经安装了`kubectl`命令。

```
sudo apt-get install kubectl
```

运行以下命令进行验证。

```
kubectl **help**
```

**注意:**请参考文档:[外部包管理器](https://cloud.google.com/sdk/docs/components#external_package_managers)

5.创建新的 Google 服务帐户(GSA)。

```
gcloud iam service-accounts create workload-identity-test
```

**注意:**您可以使用现有的服务帐户。
所需许可:`iam.serviceAccounts.create`GCP 项目。

6.为应用程序所需的 Google 服务帐户添加权限。例如，`roles/storage.objectViewer`

```
gcloud projects add-iam-policy-binding $GCP_PROJECT_ID \ 
--member serviceAccount:workload-identity-test@${GCP_PROJECT_ID}.iam.gserviceaccount.com \ 
--role roles/storage.objectViewer
```

7.设置一个启用了*工作负载标识*的 GKE 集群。

```
**export** GKE_CLUSTER_NAME=gke-wigcloud container clusters create $GKE_CLUSTER_NAME \ 
--cluster-version=1.24 \ 
--workload-pool=$GCP_PROJECT_ID.svc.id.goog
```

**注意:** GKE 集群可能需要 5-10 分钟才能完全正常工作。需要的许可:`container.clusters.create`GCP 项目。

8.在您的终端上配置`kubectl`命令。

```
gcloud container clusters get-credentials $GKE_CLUSTER_NAME
```

**注意:**这将填充`~/.kube/config`文件。需要的许可:`container.clusters.get`GCP 项目。

9.(可选)如果不想使用`default`名称空间，可以创建一个 Kubernetes 名称空间。

```
kubectl create namespace newspace
```

10.创建 Kubernetes 服务帐户(KSA)。

```
kubectl create serviceaccount \ 
--namespace newspace \ 
workload-identity-test-ksa
```

11.绑定 Google 服务帐户(GSA)和 Kubernetes 服务帐户(KSA)，以便 KSA 可以使用授予 GSA 的权限。

```
gcloud iam service-accounts add-iam-policy-binding \ 
--role roles/iam.workloadIdentityUser \ 
--member "serviceAccount:${GCP_PROJECT_ID}.svc.id.goog[newspace/workload-identity-test-ksa]" \
workload-identity-test@${GCP_PROJECT_ID}.iam.gserviceaccount.com
```

12.添加注释

```
kubectl annotate serviceaccount \ 
--namespace newspace \ 
workload-identity-test-ksa \ 
iam.gke.io/gcp-service-account=workload-identity-test@${GCP_PROJECT_ID}.iam.gserviceaccount.com
```

13.使用创建的 KSA 创建一个 Pod 以进行验证。

```
kubectl run --rm -it test-pod \
--image google/cloud-sdk:slim \
--namespace newspace \
--overrides='{ "spec": { "serviceAccount": "workload-identity-test-ksa" }  }' sh
```

运行上面的命令将登录到 Pod 并提供它的 bash shell。现在运行以下命令，查看此 pod 配置了哪个服务帐户。

```
gcloud auth list
```

这应该打印 GSA 名称。

```
Credentialed Accounts 
ACTIVE   ACCOUNT 
*        workload-identity-test@workshop-demo-namwcb.iam.gserviceaccount.com
```

# 清除

不要忘记清理资源，一旦你不再需要它。
运行以下命令:

1.  删除 GKE 群集。

```
gcloud container clusters delete $GKE_CLUSTER_NAME
```

2.删除 Google 服务帐户(GSA)。

```
gcloud iam service-accounts delete workload-identity-test@${GCP_PROJECT_ID}.iam.gserviceaccount.com
```

希望这篇博客能帮助你熟悉工作负载身份，并在 GKE 上安全地部署应用。

如果您有反馈或问题，请通过 [LinkedIn](https://linkedin.com/in/pradeepbhadani) 或 [Twitter](https://twitter.com/bhadanipradeep) 联系我。