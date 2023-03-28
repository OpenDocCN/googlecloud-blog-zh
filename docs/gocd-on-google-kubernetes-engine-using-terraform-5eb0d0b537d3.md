# 使用 Terraform 在 Google Kubernetes 引擎上运行 GoCD

> 原文：<https://medium.com/google-cloud/gocd-on-google-kubernetes-engine-using-terraform-5eb0d0b537d3?source=collection_archive---------0----------------------->

![](img/858c51e03a49479e8d2394a73887ae56.png)

GoCD 是由 [ThoughtWorks](https://www.thoughtworks.com) 开发的一个非常酷的 CD 工具。我最近一直在研究[连续交付](https://www.thoughtworks.com/continuous-delivery)，我想如果我想要一个工具与之匹配，我会尝试这个，因为你知道，它们与[书](https://www.amazon.com/dp/0321601912)有关。

# 初始设置:

在您的[云平台资源层级](https://cloud.google.com/resource-manager/docs/cloud-platform-resource-hierarchy)中创建一个文件夹，并在该文件夹中创建一个项目。例如，一个名为“工具”的文件夹和一个名为“我的工具产品”的项目。

该项目将运行 [Google Cloud DNS](https://cloud.google.com/dns) ，我们将使用[外部 dns](https://github.com/kubernetes-incubator/external-dns) 同步 Kubernetes 入口资源。它还将为 [Terraform 远程状态](https://www.terraform.io/docs/state/remote.html)运行[谷歌云存储](https://cloud.google.com/storage)。

```
export project=my-tools-prod
gcloud config set project ${project}
```

**创建一个** [**Google Cloud 托管 DNS 区域**](https://cloud.google.com/dns/zones) **:**

```
gcloud dns managed-zones create domain-com \
--description="My Default Domain" --dns-name="domain.com"
```

**向您的域名注册机构注册该区域:**

```
gcloud dns record-sets list --zone domain-comNAME          TYPE  TTL    DATA
domain.com.                         NS     21600  ns-cloud-c1.googledomains.com.,ns-cloud-c2.googledomains.com.,ns-cloud-c3.googledomains.com.,ns-cloud-c4.googledomains.com.
domain.com.                         SOA    21600  ns-cloud-c1.googledomains.com. cloud-dns-hostmaster.google.com.
```

这将显示您的 NS 记录，获取数据并在您的注册服务商上创建 NS 记录。从这一点开始，Google Cloud DNS 将管理“domain.com”。

**创建一个** [**Google 云存储桶**](https://cloud.google.com/storage/docs/creating-buckets) **为** [**Terraform 远程状态**](https://www.terraform.io/docs/state/remote.html) **:**

```
gsutil mb -p ${project} -c multi_regional -l US \
gs://${project}_tf_state
```

同时启用[对象版本](https://cloud.google.com/storage/docs/object-versioning):

```
gsutil versioning set on gs://${project}_tf_state
```

现在我们有了一个包含项目和资源的资源层次结构，它将支持我们部署 GoCD 的依赖项。

# 部署 GoCD:

这将为您提供一个公开的 GoCD 实例，并使用来自 Let's Encrypt 的 DNS 和 SSL 运行。

**安装谷歌云 SDK:**

```
curl https://sdk.cloud.google.com | bash
exec -l $SHELL
gcloud init
gcloud components install kubectl
```

**安装地形:**

```
curl -O https://releases.hashicorp.com/terraform/0.11.8/terraform_0.11.8_linux_amd64.zip
sudo unzip terraform_0.11.8_linux_amd64.zip -d /usr/local/bin
```

**设置 Google 应用默认凭证:**

```
gcloud auth application-default login
```

**克隆项目:**

```
git clone git@github.com:lzysh/ops-gke-gocd.git
```

**初始化地形:**

```
cd ops-gke-gocd/terraform
terraform init -backend-config="bucket=${project}_tf_state" -backend-config="project=${project}"
```

> *注意:此时您被设置为在 Terraform 中使用* [*远程状态*](https://www.terraform.io/docs/state/remote.html) *。*

**设置变量:**

创建一个`local.tfvars`文件，并根据需要进行编辑:

```
cp local.tfvars.EXAMPLE local.tfvars
```

> *注意:folder_id 变量将是你在上面创建的“工具”文件夹的 id。*

**地形图&适用:**

```
random=$RANDOM
terraform plan -out="plan.out" -var-file="local.tfvars" -var="project=<gocd-project>-${random}-sb" -var="host=gocd-${random}"
terraform apply "plan.out"
```

terraform 应用成功后，大约需要 5-10 分钟才能访问 GoCD 实例。入口正在做它的事情，DNS 正在传播，SSL 证书正在颁发。

**地形摧毁:**

```
terraform destroy -var-file="local.tfvars" -var="project=<gocd-project>-${random}-sb -var="host=gocd-${random}"
```