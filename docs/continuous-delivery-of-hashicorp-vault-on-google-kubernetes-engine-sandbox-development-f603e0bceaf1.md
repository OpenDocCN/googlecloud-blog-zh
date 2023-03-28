# 在 Google Kubernetes 引擎上持续交付 HashiCorp Vault:沙盒开发

> 原文：<https://medium.com/google-cloud/continuous-delivery-of-hashicorp-vault-on-google-kubernetes-engine-sandbox-development-f603e0bceaf1?source=collection_archive---------0----------------------->

**这是一个系列的第 5 部分:** [**指数**](/@blzysh/continuous-delivery-of-hashicorp-vault-on-google-kubernetes-engine-bcbf4e75f0f6)

# 概述:

[连续交付](https://continuousdelivery.com)的基础之一是[连续集成](https://continuousdelivery.com/foundations/continuous-integration)。持续集成的实践之一是频繁的签入。频繁的签入**不会**发生在对提交代码没有信心的团队中。

想象一下应用程序开发人员过渡到运营开发，或者想象一下一个系统管理员团队刚刚开始学习 IaC。这是一个相当大的变化，大多数人不喜欢大声学习。这个环境解决了这个问题，为开发人员提供了一个可以进行编码和测试的全功能环境。当我说“测试”时，我指的是测试基础设施的变化以及应用程序的变化，甚至弄清楚应用程序是如何工作的。

这些都记录在 GitHub 上的 my [README.md](https://github.com/lzysh/ops-gke-vault/blob/master/README.md) 中，并将继续维护和更新。这个设置是为了在 Linux 机器上进行 IaC 开发。

# 安装 Google Cloud SDK:

```
curl https://sdk.cloud.google.com | bash
exec -l $SHELL
gcloud init
gcloud components install kubectl beta
```

# 项目设置:

您将需要自己的沙盒项目，如初始设置中所述。就像上面提到的 ops-tools-prod 项目一样，这个项目是相同的，但是是为您自己的 IaC 个人沙盒开发的。

```
export project=ops-bcurtis-sb
gcloud config set project ${ptoject}
```

**创建一个** [**Google Cloud 托管的 DNS 区域**](https://cloud.google.com/dns/zones) **:**

```
gcloud dns managed-zones create obs-lzy-sh \
--description="My Sandbox Domain" --dns-name="obs.lzy.sh"
```

**在 ops-tools-prod:** 中向托管 dns 区域注册该区域

```
gcloud dns record-sets list --zone obs-lzy-shNAME               TYPE  TTL    DATA
obs.lzy.sh.        NS    21600  ns-cloud-a1.googledomains.com.,ns-cloud-a2.googledomains.com.,ns-cloud-a3.googledomains.com.,ns-cloud-a4.googledomains.com.
obs.lzy.sh.        SOA   21600  ns-cloud-a1.googledomains.com. cloud-dns-hostmaster.google.com. 1 21600 3600 259200 300
```

这将显示您的 NS 记录，获取数据并在 ops-tools-prod project lzy-sh managed cloud DNS zone 中创建 NS 记录。

```
gcloud --project ops-tools-prod dns record-sets transaction \
start -z=lzy-shgcloud --project ops-tools-prod dns record-sets transaction \
add -z=lzy-sh --name="obs.lzy.sh." --type=NS --ttl=300 \
"ns-cloud-a1.googledomains.com." "ns-cloud-a2.googledomains.com." \
"ns-cloud-a3.googledomains.com." "ns-cloud-a4.googledomains.com."gcloud --project ops-tools-prod dns record-sets transaction \
execute -z=lzy-sh
```

**创建一个** [**Google 云存储桶**](https://cloud.google.com/storage/docs/creating-buckets) **为** [**Terraform 远程状态**](https://www.terraform.io/docs/state/remote.html) **:**

```
gsutil mb -p ${project} -c multi_regional -l US \
gs://${project}_tf_state
```

同时启用[对象版本](https://cloud.google.com/storage/docs/object-versioning):

```
gsutil versioning set on gs://${project}_tf_state
```

# 安装 Terraform:

```
curl -O https://releases.hashicorp.com/terraform/0.11.8/terraform_0.11.8_linux_amd64.zipsudo unzip terraform_0.11.8_linux_amd64.zip -d /usr/local/bin
```

# 设置 Google 应用程序默认凭据:

```
gcloud auth application-default login
```

# 克隆项目:

```
git clone git@github.com:lzysh/ops-gke-vault.git
```

# 初始化地形:

```
cd ops-gke-vault/terraformterraform init -backend-config="bucket=${project}_tf_state" \
-backend-config="project=${project}"
```

> *注意:此时你被设置为在 Terraform 中使用* [*远程状态*](https://www.terraform.io/docs/state/remote.html) *。*

# 设置变量:

创建一个 local.tfvars 文件，并根据需要进行编辑:

```
cp local.tfvars.EXAMPLE local.tfvars
```

> *注意:folder_id 变量将是您设置了适当 IAM 角色的 Sanbox 文件夹的 id。*

# 地形规划和应用:

```
random=$RANDOMterraform plan -out="plan.out" -var-file="local.tfvars" \
-var="project=ops-vault-${random}-sb" \
-var="host=vault-${random}"terraform apply "plan.out"
```

terraform 应用完成后，大约需要 5-10 分钟才能访问 Vault 实例。入口正在做它的事情，DNS 正在传播，SSL 证书正在颁发。

解密根令牌的 URL 和命令在 Terraform 输出中。

# 在本地安装 Vault

```
curl -O https://releases.hashicorp.com/vault/0.11.1/vault_0.11.1_linux_amd64.zipsudo unzip vault_0.11.1_linux_amd64.zip -d /usr/local/bin
```

# 保险库测试示例

```
export VAULT_ADDR="$(terraform output url)"export VAULT_SKIP_VERIFY=true (Use for testing only)export VAULT_TOKEN="$(decrypted token)"
```

**启用**[**KV2**](https://www.vaultproject.io/docs/secrets/kv/kv-v2.html)**:**

```
vault kv enable-versioning secret
```

发布/获取秘密:

```
vault kv put secret/my_team/pre-prod/api_key \ key=QWsDEr876d6s4wLKcjfLPxxuyRTEvault kv get secret/my_team/pre-prod/api_key
====== Metadata ======
Key              Value
---              -----
created_time     2018-09-16T04:04:50.14260161Z
deletion_time    n/a
destroyed        false
version          1=== Data ===
Key    Value
---    -----
key    QWsDEr876d6s4wLKcjfLPxxuyRTE
```

**放置/获取多值机密:**

```
vault kv put secret/my_team/pre-prod/db_info url=foo.example.com:35533 db_name=users username=admin password=passw0rdvault kv get secret/my_team/pre-prod/db_info
====== Metadata ======
Key              Value
---              -----
created_time     2018-09-16T04:09:55.452868097Z
deletion_time    n/a
destroyed        false
version          1====== Data ======
Key         Value
---         -----
db_name     users
password    passw0rd
url         foo.example.com:35533
username    admin
```

# 地形破坏

```
terraform destroy -var-file="local.tfvars" \
-var="project=ops-vault-${random}-sb" \
-var="host=vault-${random}"
```

作为一个开发团队，我们现在可以在本地工作，尝试清理一些像这样的事情[null _ resource](https://www.terraform.io/docs/provisioners/null_resource.html)[local-exec](https://www.terraform.io/docs/provisioners/local-exec.html)[code](https://github.com/lzysh/ops-gke-vault/blob/master/terraform/main.tf#L267)。你不需要知道任何关于地形、拱顶、库伯内特等的知识。跟着自述文件走吧，**这是你学习**的地方。