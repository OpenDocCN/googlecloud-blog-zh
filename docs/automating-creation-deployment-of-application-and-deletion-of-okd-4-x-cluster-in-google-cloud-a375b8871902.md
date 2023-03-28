# 在谷歌云中自动创建、部署和删除 OKD 4.x 集群

> 原文：<https://medium.com/google-cloud/automating-creation-deployment-of-application-and-deletion-of-okd-4-x-cluster-in-google-cloud-a375b8871902?source=collection_archive---------1----------------------->

![](img/883474e3841b0623aaecb15008c92d30.png)

谷歌云中的 OKD 4.x

随着越来越多的人采用云，特别是采用多种云供应商，人们对在 Google Cloud 中自动安装和设置不同的 Kubernetes 风格越来越感兴趣。kubernetes 的一个这样的 redhat 版本通常被称为 OKD(Openshift)。

这篇文章讨论了一个这样的模块，它帮助自动创建 okd 集群，然后在这个 okd 集群上部署一个运行在 google cloud 上的应用程序。

让我们从设置这个集群的可用选项开始。用户可以遵循两种方法:

1.  **用户调配的基础架构** (UPI) —期望最终用户能够调配创建 OKD 集群所需的基础架构。
2.  **安装程序提供的基础设施**—okd 提供的安装程序将帮助我们提供 okd 集群以及所需的基础设施，前提是在您的 google cloud 项目中满足使用该安装程序的所有先决条件。

虽然第二种方法对于自动化 OKD 集群的安装和配置来说听起来很棒，但是仍然存在一些警告，比如执行一些手动步骤来满足先决条件。在设置之前和之后都需要执行这些手动步骤，以确保成功创建 OKD 集群，并在其中部署应用程序。

此外，用户可能需要在不同 OKD 版本的不同 GCP 项目中创建多个 okd 集群。所有这些不同的组合可能会使状态和日志文件的管理变得困难。

为了解决上述问题，我们创建了一个模块，使开发人员和最终用户能够快速启动 okd 集群，而无需担心先决条件。

此模块使用 terraform 代码处理诸如确保组织策略、创建服务帐户、创建 DNS 区域等先决条件。

以下是运行此代码的要求:

1.  运行代码的用户必须拥有`**owner**`权限。
2.  你必须安装`**oc cli , gcloud cli and terraform cli**`。
3.  这里写的脚本在`**linux machine**`中测试过，因此如果不使用 bash，可能需要做一些修改。

# 部署步骤

**创建 OKD 集群**

1.  Git 克隆 repo 或复制该位置的文件夹(4.x 文件夹)[https://github.com/google/shifter/tree/main/okd-cluster/4.x](https://github.com/google/shifter/tree/main/okd-cluster/4.x)
2.  您必须更新**强制变量**部分下的`install.sh`中的变量。下面是这些变量的一个片段，以及应该替换的示例值。
3.  一旦变量被更新，你可以执行`install.sh`***install . sh***将允许你在同一个项目中创建多个集群或者在不同的项目中创建多个集群。
4.  每当创建一个公共区域时，我们必须确保该公共区域的`registrar-setup`已经正确执行。如果我们计划为多个 okd 集群使用单个项目，这将是一次性工作。作为安装过程的一部分，这些名称服务器的值将显示为输出，您可以参考这些值在您的注册商处进行配置。

**链接**更新域名服务器:[https://cloud.google.com/dns/docs/update-name-servers](https://cloud.google.com/dns/docs/update-name-servers)

```
PROJECT_ID=""                    #e.g. : "pm-okd-11"
CLUSTER_NAME=""                  #e.g. : "okd-41"
OKD_VERSION=""                   #e.g. : "4.10"
BILLING_ACCOUNT_ID=""            #e.g. : "aaaaa-bbbbbb-ccccc"
PARENT=""                        #e.g. : "organizations/111222333444"
DOMAIN=""                        #e.g. : "your-domain.com."
SSH_KEY_PATH=""                  #e.g. : usr/local/google/home/username/.ssh/id_ed25519.pub# More details on redhat pull secret can be found here [https://console.redhat.com/openshift/install/pull-secret](https://console.redhat.com/openshift/install/pull-secret)REDHAT_PULL_SECRET='{"auths":{"fake":{"auth":"aWQ6cGFzcwo="}}}'
PROJECT_CREATE="false"           #make this as true if you want to create a new project under the PARENT
```

**输出**

成功执行`install.sh`后，集群详细信息如**集群端点**、**用户名**和**密码**将通过控制台日志共享，或者您也可以在日志和凭证的`okd-cluster/4.x/install-config/<PROJECT_ID>/<CLUSTER_NAME>`中找到这些详细信息，包括 okd 集群的 KUBECONFIG。

**部署应用程序**

集群自动化并没有就此结束，它还将一个示例微服务应用程序`bank-of-anthos`部署到您的集群上，以确保集群正常运行。部署应用程序后，它会显示已部署应用程序的端点。
您可以访问这个端点，进一步探索部署在集群中的应用程序。
安索斯银行**链接**:[https://github.com/GoogleCloudPlatform/bank-of-anthos](https://github.com/GoogleCloudPlatform/bank-of-anthos)

**删除 OKD 集群** 一旦您在 GCP 尝试并测试了您的应用程序和 OKD 集群的部署，并且您不再需要该集群，那么您可以按照下面提到的步骤删除该集群。

1.  为了删除集群，我们将类似地更新出现在`destroy.sh`文件的 MANDATORY_VARIABLES 部分中的变量。我们必须确保下面列出的变量被正确更新。
2.  执行`destroy.sh`将允许您删除集群。

```
########## Mandatory Variables ###########
PROJECT_ID=””   # e.g. “pm-okd-11”
CLUSTER_NAME=”” # e.g. ”okd-41"
OKD_VERSION=””  # e.g. ”4.10"
```

**注意:**`destroy.sh`脚本使用在`install.sh`文件执行过程中创建的同一个`service-account-key.json`。如果这个`service-account-key.json`被删除，那么你可以按照下面提到的步骤创建一个替代的`service-account-key.json`。

```
gcloud iam service-accounts keys create ${SA_JSON_FILENAME} --iam-account=okd-sa@${PROJECT_ID}.iam.gserviceaccount.com #gcloud iam service-accounts keys create service-account-key.json --iam-account=okd-sa@pm-okd-11.iam.gserviceaccount.com mkdir ${CWD_PATH}/01-projectsetup/sa-keys/${PROJECT_ID}/ mv ${CWD_PATH}/${SA_JSON_FILENAME} ${CWD_PATH}/01-projectsetup/sa-keys/${PROJECT_ID}/
```

本博客使用一个模块来帮助您自动创建 OKD 集群、在 OKD 集群中部署应用程序以及删除 OKD 集群(如果需要)。共享的模块目前支持 OKD 4.9 和 OKD 4.10，但是代码完全可扩展以支持任何 OKD 4.x 版本。

**链接**:[https://github.com/google/shifter/tree/main/okd-cluster/4.x](https://github.com/google/shifter/tree/main/okd-cluster/4.x)

该模块是 Shifter 的一部分，使最终用户能够测试并将工作负载从 Openshift 迁移到 Anthos。关于这个模块的更多细节可以在[https://github.com/google/shifter](https://github.com/google/shifter/tree/main/okd-cluster/4.x)找到

如果您对本文中提供的自动化有任何疑问或建议，请随时联系我们。