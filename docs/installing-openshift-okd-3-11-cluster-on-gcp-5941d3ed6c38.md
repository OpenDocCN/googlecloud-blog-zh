# 在 GCP 上安装 Openshift OKD 3.11 集群

> 原文：<https://medium.com/google-cloud/installing-openshift-okd-3-11-cluster-on-gcp-5941d3ed6c38?source=collection_archive---------2----------------------->

这篇博客将带你了解在谷歌计算引擎上配置 Openshift OKD 3.11 集群所需的步骤。
我正在寻找一种简单的方法来启动 100 个 Openshift 集群进行测试和验证。

自动化使用 Terraform 在 GCP 上配置基础架构，然后部署 Ansible playooks 来配置 OKD 集群。Automation 还在 Openshift 上部署了一个示例应用程序( [Bank of Anthos](https://github.com/avinashkumar1289/bank-of-anthos) ),并将管理员角色绑定到默认用户。它还生成一个可用于登录集群的不记名令牌。

让我们开始吧。

我将把整个自动化分解成不同的任务，并详细讨论每个任务。

让我们在开始自动化之前先讨论一下先决条件。

资源库:[https://github . com/Google/shifter/tree/v 0 . 3 . 0/okd-cluster/3.11](https://github.com/google/shifter/tree/main/okd-cluster)

**先决条件**

1.  拥有所有者权限的谷歌云项目
2.  如果从 GCP 以外的地方运行自动化，则允许创建服务帐户密钥
3.  在部署脚本之前，需要有效的计费帐户详细信息。需要在 terraform tfvars 文件中更新计费详细信息。
4.  组织 ID 是必需的，需要在 terraform.tfvars 文件中更新。
5.  部署 OKD 群集需要有效的域，并且需要更新 DNS 条目

如果您已经具备了所需的所有先决条件，那么让我们直接进入每项任务。请参考个人任务的 *okd-create.sh* 脚本

**生成 SSH**

此任务生成 SSH 密钥，该密钥将用于建立开发人员节点和控制/工作人员节点的无密码访问。ssh 密钥在目录“＄{ HOME }/GCP _ keys/id _ RSA”中生成

**提供基础设施**

此任务使用 Terraform 文件并在 GCP 上配置基础架构。

**terra form 的先决条件** 1)将 terraform.tfvars.sample 文件更新为 terraform.tfvars 文件，并更新变量。
样本 terraform.tfvars 文件

```
prefix                    = "dev"                            
region                   = "europe-west1"                             master_count             = 1                             
node_count               = 2                                                           project_name             = "okd-tf"                                                        
org_id                   = "34545645634345"                          folder_id                = ""  
environment              = "dev"                             billing_code             = "1234"         
billing_account          = "0090FE-ED3D81-ER565"                            application_name         = "" 
primary_contact          = "avinash@example.com"                             activate_apis = [                               "compute.googleapis.com",                               "cloudbilling.googleapis.com",                               "dns.googleapis.com",                               "servicenetworking.googleapis.com"]                             master_subdomain        = "okd.example.com"                             public_subdomain        = "console.okd.example.com"                             dns_master_subdomain    = "okd.example.com."                             ssh_user                = ""
```

2)使用后端 GCS bucket
更新 backend.tf 文件 3)通过 gcloud 命令进行身份验证，或者如果您从 GCP 虚拟机运行脚本，则将正确的服务帐户附加到虚拟机

Terraform 提供了一个主节点和三个工作节点，以及一个用于运行 ansible 剧本的开发人员虚拟机。
检查 *okd-template* 目录下的*ansi ble-hosts . template . txt*文件。该主机文件用于运行 ansible 剧本来创建 OKD 集群。

**复制清单文件**

该步骤将所需的 ansible 主机模板文件和 ssh 密钥复制到 developer 节点，这是运行 ansible 剧本和无密码访问 control 和 worker 节点所需的

**供应集群**

这些步骤克隆了 Openshift [github](https://github.com/openshift/openshift-ansible.git) 仓库
中可用的剧本，并签出相关的 OKD 版本(在我们的例子中是 3.11)。
这一步从 bastion 主机运行所需的 ansible 剧本。这包括 prerequisites.yml 行动手册和 deploy_cluster.yml 行动手册。
成功运行此任务后，您应该已经创建了 OKD 集群，并且可以从 VPC 中访问该集群。
要访问域上的集群，请检查生成的名称服务器的日志，并更新域提供商上的名称服务器。

**部署清单**

这一步在 OKD 集群上部署一个样本测试应用程序库 Anthos([https://github.com/avinashkumar1289/bank-of-anthos](https://github.com/avinashkumar1289/bank-of-anthos))。

**OKD 配置**

该步骤通过下面的命令将集群管理员角色绑定到默认用户。

```
oc adm policy add-cluster-role-to-user cluster-admin shifter
```

它还设置了一个环境变量$TOKEN，该变量存储用于访问 OKD 集群的承载令牌。您可以使用以下命令向群集进行身份验证。

```
oc login --token=$TOKEN --server=console.okd.example.com
```

这标志着 OKD 集群创建自动化脚本的结束。
脚本文件夹中有一个 variable.sh 文件。默认变量文件如下所示

```
#!/bin/bash                                                           SSH_PATH=${HOME}/gcp_keys/id_rsa                             SSH_PUB_FILE=${SSH_PATH}.pub                             SSH_USER=avinash                             
REGION=europe-west1                             
ZONE=europe-west1-b                             OKD_VERSION=release-3.11                             LOG_PATH="${HOME}/okd/logs"                             LOG_FILE="${LOG_PATH}/create-okd-logs-$(date +'%Y-%m-%d-%H:%M:%S')"                             DELETE_LOG_FILE="${LOG_PATH}/delete-okd-logs-$(date +'%Y-%m-%d-%H:%M:%S')"
```

**删除基础设施**

运行 *okd-delete.sh* 删除 anthos 银行的默认部署，并销毁通过 Terraform 提供的基础设施。

OKD 集群自动化是 Google Shifter 工具的一部分，用于将 Kubernetes 的工作负载从 Openshift 迁移到 GKE。有兴趣了解更多关于移位器的信息，请查看这个 [github](https://github.com/google/shifter) 库。

至此，我们已经到了博客的结尾。我希望它是有用的。
如果你有任何问题或反馈，你可以通过 Linkedin 联系我。