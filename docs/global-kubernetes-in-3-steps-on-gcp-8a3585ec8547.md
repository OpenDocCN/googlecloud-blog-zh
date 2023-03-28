# GCP 三步走全球 Kubernetes

> 原文：<https://medium.com/google-cloud/global-kubernetes-in-3-steps-on-gcp-8a3585ec8547?source=collection_archive---------0----------------------->

创建一个全球联合的 Kubernetes 集群可能听起来令人生畏，但实际上只需要几个小步骤。

*   创建项目和集群
*   安装并加入 kubefed
*   全球部署

***kubefed*** 实用程序在这个过程中花费了大部分精力。如果您想深入了解幕后发生的事情，请查看 Kelsey Hightower 的 [Kubernetes 集群联盟。我在这里写的很多东西都是模仿凯尔西和](https://github.com/kelseyhightower/kubernetes-cluster-federation/blob/master/README.md)[桑迪普的原始文章](/google-cloud/planet-scale-microservices-with-cluster-federation-and-global-load-balancing-on-kubernetes-and-a8e7ef5efa5e)的，只是根据我的需要进行了简化和调整。

**TL；DR**

# 先决条件

**库贝菲德**

*   本教程使用一个名为 Kubefed 的实用程序来自动化联邦。[点击这里获取 Kubefed】](https://kubernetes.io/docs/tasks/federation/set-up-cluster-federation-kubefed/#getting-kubefed)

域名服务器(Domain Name Server)

*   kube-dns 是在这个过程中安装的，用于设置和管理服务的 dns 条目。为了有效地使用它，您需要有一个 dns 区域可供使用。记下区域名称(带有尾随点),并将其设置在下面的变量中

首先，让我们设置一些变量，以便稍后在命令中使用

```
export PROJECT=your-projectname
export DNS_ZONE=your.domain.com.
```

# 步骤 1 —创建集群

在东部和西部创建集群

```
gcloud container clusters create west-cluster \
--zone us-west1-a --scopes “cloud-platform,storage-ro,logging-write,monitoring-write,service-control,service-management,[https://www.googleapis.com/auth/ndev.clouddns.readwrite](https://www.googleapis.com/auth/ndev.clouddns.readwrite)"gcloud container clusters create east-cluster \
--zone us-east1-b --scopes “cloud-platform,storage-ro,logging-write,monitoring-write,service-control,service-management,[https://www.googleapis.com/auth/ndev.clouddns.readwrite](https://www.googleapis.com/auth/ndev.clouddns.readwrite)"
```

获取凭据

```
# Workaround for RBAC error
# [https://github.com/kubernetes/kubernetes/issues/42559](https://github.com/kubernetes/kubernetes/issues/42559)
gcloud config set container/use_client_certificate True
export CLOUDSDK_CONTAINER_USE_CLIENT_CERTIFICATE=True# Get credentials
gcloud container clusters get-credentials west-cluster --zone=us-west1-a
gcloud container clusters get-credentials east-cluster --zone=us-east1-b
```

**创建别名**

```
kubectl config set-context east --cluster=gke_${PROJECT}_us-east1-b_east-cluster --user=gke_${PROJECT}_us-east1-b_east-clusterkubectl config set-context west --cluster=gke_${PROJECT}_us-west1-a_west-cluster --user=gke_${PROJECT}_us-west1-a_west-cluster
```

# 步骤 2 —安装并加入 kubefed

这里我们使用 kubefed 来初始化使用“east”上下文的联邦

```
kubefed init kfed \
 --host-cluster-context=east \
 --dns-zone-name=${DNS_ZONE} \
 --dns-provider=google-clouddnskubectl config use-context kfed
```

“kfed”上下文应该用于所有联合部署。您可以独立地检查各个集群，但是所有的操作都应该在联邦上下文中执行。

**将集群加入联盟**
向联盟服务器提供加入集群的上下文

```
kubefed --context=kfed join east-cluster \
--cluster-context=east \
--host-cluster-context=eastkubefed --context=kfed join west-cluster \
--cluster-context=west \
--host-cluster-context=east
```

> 注意:在联盟中添加来自欧洲的集群时，由于某种原因，我无法使用别名。我将原来的上下文名称替换回参数`-cluster-context`,它完美地连接起来

**创建默认名称空间**

```
kubectl --context=kfed create ns default
```

**查看集群是否已加入**

```
kubectl --context=kfed get clusters
```

# 步骤 3 —部署您的应用程序

让我们部署一个简单的应用程序来看看这一点

```
kubectl --context=kfed create deployment nginx --image=nginx && \
 kubectl --context=kfed scale deployment nginx --replicas=4
```

看到它在两个集群中自动部署

让我们查看一下**东**和**西**星团的豆荚

```
**kubectl** --**context=east get pods**
NAME READY STATUS RESTARTS AGE
nginx-167110513–83gdg 1/1 Running 0 32s
nginx-167110513-s30vs 1/1 Running 0 32s**kubectl** --**context=west get pods**
NAME READY STATUS RESTARTS AGE
nginx-167110513–1wrjz 1/1 Running 0 38s
nginx-167110513-v71h2 1/1 Running 0 38s
```

# 结论

很好很容易！设置一个全球分布的应用程序并不需要太多的努力。通过 Kubenetes 集群联合，您可以将多个集群视为一个实体。只需几个简单的步骤，设置就完成了，可以使用了。

如果这还不够简单的话，下面是一个脚本中的全部内容

```
## Requires these variables to be set
# export PROJECT=your-projectname
# export DNS_ZONE=your.domain.com.[[ -z “$PROJECT” ]] && { echo “Env var PROJECT is missing, please export PROJECT=your-project” ; exit 1; }
[[ -z “$DNS_ZONE” ]] && { echo “Env var DNS_ZONE is missing, please export DNS_ZONE=your.domain.com.” ; exit 1; }# Create Clusters
gcloud container clusters create west-cluster --zone us-west1-a --scopes “cloud-platform,storage-ro,logging-write,monitoring-write,service-control,service-management,[https://www.googleapis.com/auth/ndev.clouddns.readwrite](https://www.googleapis.com/auth/ndev.clouddns.readwrite)"
gcloud container clusters create east-cluster --zone us-east1-b --scopes “cloud-platform,storage-ro,logging-write,monitoring-write,service-control,service-management,[https://www.googleapis.com/auth/ndev.clouddns.readwrite](https://www.googleapis.com/auth/ndev.clouddns.readwrite)"gcloud config set container/use_client_certificate True
export CLOUDSDK_CONTAINER_USE_CLIENT_CERTIFICATE=True# Get credentials
gcloud container clusters get-credentials west-cluster --zone=us-west1-a
gcloud container clusters get-credentials east-cluster --zone=us-east1-b# Make Aliases
kubectl config set-context east --cluster=gke_${PROJECT}_us-east1-b_east-cluster --user=gke_${PROJECT}_us-east1-b_east-cluster
kubectl config set-context west --cluster=gke_${PROJECT}_us-west1-a_west-cluster --user=gke_${PROJECT}_us-west1-a_west-cluster# Install and join the federation
kubefed init kfed --host-cluster-context=east --dns-zone-name=${DNS_ZONE} --dns-provider=google-clouddns
kubefed --context=kfed join east-cluster --cluster-context=east --host-cluster-context=east
kubefed --context=kfed join west-cluster --cluster-context=west --host-cluster-context=east
kubectl --context=kfed create ns default# Use kfed context to create resources and deployments
kubectl config use-context kfed# Show me the money
echo “kubectl --context=kfed get clusters “
kubectl --context=kfed get clusters
```