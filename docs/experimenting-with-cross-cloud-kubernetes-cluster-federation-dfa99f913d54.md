# 尝试跨云 Kubernetes 集群联盟

> 原文：<https://medium.com/google-cloud/experimenting-with-cross-cloud-kubernetes-cluster-federation-dfa99f913d54?source=collection_archive---------1----------------------->

# 什么/为什么？

在之前的帖子中，我介绍了一种在现有 AWS 环境中部署集群的方法，作为我对可集成性问题的回答。

那么我接下来最常被问到的问题是什么呢？你看了标题就知道了:

> **我们能否将 Kubernetes 跨越多个集群/云？**

随着越来越多的人将 k8s 视为下一个大平台，这使他们能够抽象出大量的云/裸机基础设施，这个问题定期出现是公平的。

这是前所未有的公平，因为有巨大的社区努力来实现它，并且在过去的 6 个月里取得了巨大的进展。

所以理论上…是的。现在，它在现实世界中有效吗？我们去看看吧！

让我把这篇文章的其余部分说清楚:不幸的是，我们将*而不是*实现多云 Kubernetes 的完全自动化。然而，我们将到达这样一个点，您可以以适中的成本运行一个多云联盟，并操纵最基本的原语。对于不工作的，我们会看到解决办法。至少，希望你已经对 k8s 联盟的现状有所了解。

# 关于联邦的预备词

什么是 Kubernetes 联盟？要回答这个问题，这么说吧，跨越一个全世界独一无二的集群，退一步说，是绝对不可能的。这就是为什么云有区域、az、单元、机架…您必须分割您的基础架构，以便特定的地理区域是构建更大解决方案的构建单位。

然而，当你在 AWS、Azure 或 GCP 上连接时，你可以跨所有地区创建和管理资源。更好的是，有些服务是全局的(DNS、对象存储…)，在整个环境中共享。

如果有很多人认为 Kubernetes 是(您的)可伸缩性问题的解决方案，那么您已经考虑或将考虑在几个区域和 az 中旋转几个集群。
控制所有星团的“库伯内特”位面是联邦。它提供了集中控制，并将命令分布到所有联合云。

TL；DR:如果你使用 Kubernetes，如果你想要一个跨多个地区和云的全球应用程序，那么联合就是你要做的。

现在 Federation 也是 Kubernetes 生态系统中非常年轻的子项目，距离足够成熟还有很长的路要走。让我们来看看对于那些开始使用 k8s 或考虑使用 k8s 的人来说，我们现在处于什么位置。

# 这个计划

在这篇博客中，我们将做以下事情:

1.  在亚马逊、Azure 和 GKE 部署 Kubernetes 集群
2.  用我们控制的域名创建一个谷歌云域名系统区域
3.  在 GKE 安装一个联邦控制面板
4.  测试开箱后哪些可行，哪些不可行

# 要求

接下来，重要的是:

*   你了解 Kubernetes 101
*   你了解 k8s 联盟的概念
*   您拥有 AWS、GCP 和 Azure 的管理员凭据
*   您熟悉可用于部署 Kubernetes 的工具
*   你有 GKE 和谷歌云域名系统(或 route53 或 Azure 域名系统)的概念

# 前戏

*   确保你已经安装了 Juju。

在 Ubuntu 上，

```
sudo apt-add-repository ppa:juju/stable
sudo apt update && apt upgrade -yqq
sudo apt install -yqq juju 
```

对于其他操作系统，查找[正式文档](https://jujucharms.com/docs/2.0/getting-started-general)

然后用你的凭证连接到 AWS 云，阅读[这一页](https://jujucharms.com/docs/2.0/help-aws)

最后，通过[这个页面](https://jujucharms.com/docs/2.0/help-azure)连接到 Azure

*   确保你已经安装了[谷歌云 SDK 套件](https://cloud.google.com/sdk/docs/quickstart-debian-ubuntu)
*   最后，复制回购协议，以访问所有的来源

```
git clone [https://github.com/madeden/blogposts](https://github.com/madeden/blogposts) ./
cd blogposts/k8s-federation
```

好吧！让我们现在就结盟吧！

# 部署

在本节中，我们将

*   在 Azure、AWS 和 GKE 上部署 k8s
*   为联盟安装工具
*   部署联盟

## 微软 Azure

让我们首先在 Azure 中生成一个 k8s 集群:

```
# Bootstrap the Juju Controller
juju bootstrap azure/westeurope azure \
  --bootstrap-constraints “root-disk=64G mem=8G” \
  --bootstrap-series xenial
# Deploy Canonical Distribution of Kubernetes
juju deploy src/bundle/k8s-azure.yaml
```

Azure 相对来说旋转起来比较慢，所以你可以让它过去，我们稍后再回来。

## 亚马逊 AWS

现在，在 AWS 上采取同样的行动

```
juju bootstrap aws/us-west-2 aws \
  --bootstrap-constraints "root-disk=64G mem=8G" \
  --bootstrap-series xenial
# Deploy Canonical Distribution of Kubernetes
juju deploy src/bundle/k8s-aws.yaml
```

这大约需要 10 分钟，所以让它运行，我们稍后再回来

## GKE

在这里，我们部署了一个 DNS 区域和一个小型 GKE 群集

```
# Spin up the DNS Zone
gcloud dns managed-zones create federation \
  --description "Kubernetes federation testing" \
  --dns-name demo.madeden.com
# Spin up a GKE cluster
gcloud container clusters create gke \
  --zone=us-east1-b \
  --scopes "cloud-platform,storage-ro,service-control,service-management,[https://www.googleapis.com/auth/ndev.clouddns.readwrite](https://www.googleapis.com/auth/ndev.clouddns.readwrite)" \
  --num-nodes=2
```

您将需要一个全球可用的 DNS，您可以通过编程来驱动，以操作联盟，这就是为什么我们使用这个谷歌云 DNS。它也适用于 AWS Route53，但其他集成正在进行中。

我必须将它配置为从甘地委派子域，这相当容易，因为谷歌在他们的[帮助页面](https://cloud.google.com/dns/zones/)上给了你所有你需要的说明。

# 联盟

## 安装 kubefed

从 1.5 开始，Kubernetes 提供了一个名为 [KubeFed](https://kubernetes.io/docs/admin/federation/kubefed/) 的工具来管理联盟的生命周期。

安装时使用:

```
curl -O [https://storage.googleapis.com/kubernetes-release/release/v1.5.2/kubernetes-client-linux-amd64.tar.gz](https://storage.googleapis.com/kubernetes-release/release/v1.5.2/kubernetes-client-linux-amd64.tar.gz)
tar -xzvf kubernetes-client-linux-amd64.tar.gz
sudo cp kubernetes/client/bin/kubefed /usr/local/bin
sudo chmod +x /usr/local/bin/kubefed
sudo cp kubernetes/client/bin/kubectl /usr/local/bin
sudo chmod +x /usr/local/bin/kubectl
mkdir -p ~/.kube
```

## 配置 kubectl

在 Azure 上，检查您的群集现在是否启动并运行:

```
# Switch Juju to the Azure cluster
juju switch azure
# Get status
juju status
# Which gets you (if finished)
Model    Controller        Cloud/Region      Version
default  azure             azure/westeurope  2.1-beta5App                Version  Status  Scale  Charm              Store       Rev  OS      Notes
easyrsa            3.0.1    active      1  easyrsa            jujucharms    6  ubuntu  
etcd               2.2.5    active      3  etcd               jujucharms   23  ubuntu  
flannel            0.7.0    active      4  flannel            jujucharms   10  ubuntu  
kubernetes-master  1.5.2    active      1  kubernetes-master  jujucharms   11  ubuntu  exposed
kubernetes-worker  1.5.2    active      3  kubernetes-worker  jujucharms   13  ubuntu  exposedUnit                  Workload  Agent  Machine  Public address  Ports           Message
easyrsa/0*            active    idle   0        40.114.244.142                  Certificate Authority connected.
etcd/0                active    idle   1        40.114.247.142  2379/tcp        Healthy with 3 known peers.
etcd/1*               active    idle   2        104.47.167.187  2379/tcp        Healthy with 3 known peers.
etcd/2                active    idle   3        104.47.163.137  2379/tcp        Healthy with 3 known peers.
kubernetes-master/0*  active    idle   4        40.114.243.251  6443/tcp        Kubernetes master running.
  flannel/2           active    idle            40.114.243.251                  Flannel subnet 10.1.96.1/24
kubernetes-worker/0   active    idle   5        104.47.162.134  80/tcp,443/tcp  Kubernetes worker running.
  flannel/1           active    idle            104.47.162.134                  Flannel subnet 10.1.94.1/24
kubernetes-worker/1*  active    idle   6        104.47.162.82   80/tcp,443/tcp  Kubernetes worker running.
  flannel/0*          active    idle            104.47.162.82                   Flannel subnet 10.1.58.1/24
kubernetes-worker/2   active    idle   7        104.47.160.138  80/tcp,443/tcp  Kubernetes worker running.
  flannel/3           active    idle            104.47.160.138                  Flannel subnet 10.1.43.1/24Machine  State    DNS             Inst id    Series  AZ
0        started  40.114.244.142  machine-0  xenial  
1        started  40.114.247.142  machine-1  xenial  
2        started  104.47.167.187  machine-2  xenial  
3        started  104.47.163.137  machine-3  xenial  
4        started  40.114.243.251  machine-4  xenial  
5        started  104.47.162.134  machine-5  xenial  
6        started  104.47.162.82   machine-6  xenial  
7        started  104.47.160.138  machine-7  xenialRelation      Provides           Consumes           Type
certificates  easyrsa            etcd               regular
certificates  easyrsa            kubernetes-master  regular
certificates  easyrsa            kubernetes-worker  regular
cluster       etcd               etcd               peer
etcd          etcd               flannel            regular
etcd          etcd               kubernetes-master  regular
cni           flannel            kubernetes-master  regular
cni           flannel            kubernetes-worker  regular
cni           kubernetes-master  flannel            subordinate
kube-dns      kubernetes-master  kubernetes-worker  regular
cni           kubernetes-worker  flannel            subordinate
```

现在下载配置文件

```
juju scp kubernetes-master/0:/home/ubuntu/config ./config-azure
```

在 AWS 上重复该操作

```
juju switch aws
Model    Controller     Cloud/Region   Version
default  aws            aws/us-west-2  2.1-beta5App                Version  Status  Scale  Charm              Store       Rev  OS      Notes
easyrsa            3.0.1    active      1  easyrsa            jujucharms    6  ubuntu  
etcd               2.2.5    active      3  etcd               jujucharms   23  ubuntu  
flannel            0.7.0    active      4  flannel            jujucharms   10  ubuntu  
kubernetes-master  1.5.2    active      1  kubernetes-master  jujucharms   11  ubuntu  exposed
kubernetes-worker  1.5.2    active      3  kubernetes-worker  jujucharms   13  ubuntu  exposedUnit                  Workload  Agent  Machine  Public address  Ports           Message
easyrsa/0*            active    idle   2        10.0.251.198                    Certificate Authority connected.
etcd/0*               active    idle   1        10.0.252.237    2379/tcp        Healthy with 3 known peers.
etcd/1                active    idle   6        10.0.251.143    2379/tcp        Healthy with 3 known peers.
etcd/2                active    idle   7        10.0.251.31     2379/tcp        Healthy with 3 known peers.
kubernetes-master/0*  active    idle   0        35.164.145.16   6443/tcp        Kubernetes master running.
  flannel/0*          active    idle            35.164.145.16                   Flannel subnet 10.1.37.1/24
kubernetes-worker/0*  active    idle   3        52.27.16.150    80/tcp,443/tcp  Kubernetes worker running.
  flannel/3           active    idle            52.27.16.150                    Flannel subnet 10.1.11.1/24
kubernetes-worker/1   active    idle   4        52.10.62.234    80/tcp,443/tcp  Kubernetes worker running.
  flannel/1           active    idle            52.10.62.234                    Flannel subnet 10.1.43.1/24
kubernetes-worker/2   active    idle   5        52.27.1.171     80/tcp,443/tcp  Kubernetes worker running.
  flannel/2           active    idle            52.27.1.171                     Flannel subnet 10.1.68.1/24Machine  State    DNS            Inst id              Series  AZ
0        started  35.164.145.16  i-0a3fdb3ce9590cb7e  xenial  us-west-2a
1        started  10.0.252.237   i-0dcbd977bee04563b  xenial  us-west-2b
2        started  10.0.251.198   i-04cedb17e22064212  xenial  us-west-2a
3        started  52.27.16.150   i-0f44e7e27f776aebf  xenial  us-west-2b
4        started  52.10.62.234   i-02ff8041a61550802  xenial  us-west-2a
5        started  52.27.1.171    i-0a4505185421bbdaf  xenial  us-west-2a
6        started  10.0.251.143   i-05a855d5c0c6f847d  xenial  us-west-2a
7        started  10.0.251.31    i-03f1aafe15d163a34  xenial  us-west-2aRelation      Provides           Consumes           Type
certificates  easyrsa            etcd               regular
certificates  easyrsa            kubernetes-master  regular
certificates  easyrsa            kubernetes-worker  regular
cluster       etcd               etcd               peer
etcd          etcd               flannel            regular
etcd          etcd               kubernetes-master  regular
cni           flannel            kubernetes-master  regular
cni           flannel            kubernetes-worker  regular
cni           kubernetes-master  flannel            subordinate
kube-dns      kubernetes-master  kubernetes-worker  regular
cni           kubernetes-worker  flannel            subordinate
```

和

```
juju scp kubernetes-master/0:/home/ubuntu/config ./config-aws
```

现在是 GKE

```
gcloud container clusters get-credentials gce --zone=us-east1-b
```

这最后一个操作将实际创建或修改您的 **~/。GKE 的 kube/config** 文件，所以您可以直接从 kubectl 查询上下文。GKE 有创建非常长的名字的倾向，我们现在只想要一个短的来减轻我们的命令行负担。

```
# Identify the cluster name
LONG_NAME=$(kubectl config view -o jsonpath='{.contexts[*].name}')
# Replace it in kubeconfig
sed -i "s/$LONG_NAME/gke/g" ~/.kube/config
```

现在修改 Juju 下载的文件，将它们集成到这个配置文件中。

一些聪明的人构建了一个工具来合并 kubeconfig 文件: [load-kubeconfig](https://github.com/Collaborne/load-kubeconfig)

```
# Install tool 
sudo npm install -g load-kubeconfig
# Replace the username and context name with our cloud names in both files and combine
for cloud in aws azure
do
  sed -i -e "s/juju-cluster/${cloud}/g" \
    -e "s/juju-context/${cloud}/g" \
    -e "s/ubuntu/${cloud}/g" \
    ./config-${cloud}
  load-kubeconfig ./config-${cloud}
done
```

很好，现在你可以很容易地在 3 个集群之间切换，只需使用— **context={gke | aws | azure }。**

## 标记节点

部署 Kubernetes 联盟的目标之一是确保应用程序的多区域 HA。在区域内，您也希望在 az 之间有 HA。因此，您应该考虑为每个 AZ 部署一个集群。

对于一个联盟来说，没有什么比另一个 k8s 更像一个 k8s 集群了。如果不给它一些提示，你就不能指望它有区域意识。这就是我们将在这里做的事情。

默认情况下，Juju 会给你以下标签:

```
# AWS
kubectl --context=aws get nodes --show-labels
NAME           STATUS    AGE       LABELS
ip-10-0-1-54   Ready     1d        beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=ip-10-0-1-54
ip-10-0-1-95   Ready     1d        beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=ip-10-0-1-95
ip-10-0-2-43   Ready     1d        beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=ip-10-0-2-43
# Azure
kubectl --context=azure get nodes --show-labels
NAME        STATUS    AGE       LABELS
machine-5   Ready     2h        beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=machine-5
machine-6   Ready     2h        beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=machine-6
machine-7   Ready     2h        beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=machine-7
```

你只有 3 个集群和不同的地区(AWS 是美国西部 2，Azure 是欧盟西部 2，GKE 是美国东部 1)，所以我们会稍微假装一下

```
# Labelling the nodes of AWS for US West 2a (random pick)
for node in $(kubectl - aws get nodes -o json | jq --raw-output '.items[].metadata.name')
do
  kubectl --context=aws label nodes \
    ${node} \
    failure-domain.beta.kubernetes.io/region=us-west-2
  kubectl --context=aws label nodes \
    ${node} \
    failure-domain.beta.kubernetes.io/zone=us-west-2a
done
# Labelling the nodes of Azure for EU West 2a (random pick)
for node in $(kubectl - azure get nodes -o json | jq --raw-output '.items[].metadata.name')
do
  kubectl --context=azure label nodes \
    ${node} \
    failure-domain.beta.kubernetes.io/region=eu-west-2
  kubectl --context=azure label nodes \
    ${node} \
    failure-domain.beta.kubernetes.io/zone=eu-west-2a
done
```

**注意**:在真实环境中，您将有更多的节点需要配置，并且有更好的策略需要采用。这只是为了解释当您联合集群以管理集群和云的故障时，在 k8s 中设置这个参数是很重要的。

## 确保集群共享相同的 GLBC

[本页](https://kubernetes.io/docs/user-guide/federation/federated-ingress/)明确指出，我们的 GLBC 在所有集群中必须完全相同。因为您用 Juju 部署了两个集群，用 GKE 部署了一个集群，所以现在情况不是这样。虽然 Canonical 的 CDK 100%在上游，但情况几乎是这样。只是标签和一些标记不一样。

因此，以下步骤是可选的。如果不这样做，您将不得不手动向所有集群发布 DNS 配置和入口。这是唯一的区别。

让我们删除 Azure & AWS 集群上的 L7 LB，用类似于 GKE 的其他集群来替换它们

```
for cluster in aws azure
do
  # Delete old ones
  kubectl --context ${cloud} delete \
    rc/default-http-backend \
    rc/nginx-ingress-controller \
    svc/default-http-backend
  # Replace by new ones taken from GKE
  kubectl --context ${cloud} create -f \
    src/manifests/l7-svc.yaml
  kubectl --context ${cloud} create -f \
    src/manifests/l7-deployment.yamldone
```

## 正在初始化联盟

现在我们准备联合我们的集群，这实质上意味着我们添加了一个跨集群控制平面，它本身托管在第三个 Kubernetes 集群中。

联合集群带来了一些好处，比如

*   能够定义多群集服务、副本集/部署、入口，
*   区域之间的服务故障转移
*   单点服务定义

您之前安装的 Kubefed 是联邦生命周期(从初始化到销毁)的官方工具。我们将使用 GKE 集群来管理我们在 Azure 和 AWS 中的两个集群。

用以下命令初始化我们的“魔环”的控制平面

```
kubefed init magicring \
  --host-cluster-context=gke \
  --dns-zone-name="demo.madeden.net."
Federation API server is running at: 130.211.62.225
```

命令很简单，负责安装

*   主机集群(GKE)中一个名为 federation-system 的新命名空间
*   联邦的新 API 服务器
*   联邦的新控制器管理器
*   kubeconfig 文件中的一个新上下文，专门用于与 Kubernetes 的这个超级层进行交互，它以联邦名称命名

为了控制联邦，我们现在可以回到 kubectl 并切换到它的上下文中

```
kubectl config use-context magicring
Switched to context “magicring”.
```

现在将我们的两个集群添加到魔戒中

```
# add AWS
kubefed join aws \
  --host-cluster-context=gke
cluster "aws" created
# Now Azure
kubefed join azure \
  --host-cluster-context=gke
cluster "azure" created
```

这些命令将根据您的 kubeconfig 在联邦控制平面中创建一对秘密，以便它可以与各个 Kube API 服务器进行交互。如果您计划构建跨 VPN 或复杂网络的联盟，这意味着您必须确保控制平面可以与您部署的各种 API 端点对话。

最后，我们可以用 kubectl 查询一个新的构造“集群”:

```
kubectl get clusters
NAME STATUS AGE
aws Ready 1m
azure Ready 1m
```

恭喜您，您在不到 30 分钟的时间内联合了一对 Kubernetes 集群！！！

现在让我们一起玩吧

# 部署多云应用程序

既然联邦正在运行，让我们尝试部署联邦应该管理的所有原语:

*   配额/资源管理的名称空间
*   共享数据的配置映射和机密
*   应用程序的部署/复制集/复制控制器
*   展会服务/入口

## 名称空间

让我们部署一个测试名称空间:

```
# Creation
kubectl --context=magicring create -f src/manifests/test-ns.yaml 
namespace "test-ns" created# Check AWS
kubectl --context=aws get ns
NAME             STATUS    AGE
ns/default       Active    3d
ns/kube-system   Active    3d
ns/test-ns       Active    50s# Check Azure
kubectl --context=azure get ns
NAME             STATUS    AGE
ns/default       Active    2d
ns/kube-system   Active    2d
ns/test-ns       Active    1m
```

好的，配额和资源管理的基础在全球范围内都可用。这是一个胜利。

## 配置映射/机密

将测试配置图推送到群集，以确认其工作正常:

```
# Publish
kubectl --context magicring create -f src/manifests/test-configmap.yaml 
configmap "test-configmap" created# Check AWS
kubectl --context aws get cm 
NAME       DATA      AGE
test-configmap   1         55s# Check Azure
kubectl --context azure get cm 
NAME       DATA      AGE
test-configmap   1         1m
```

好了，我们的配置图已经在所有的云中共享了。我们可以像预期的那样对 CMs 的配置进行单点控制。

这也适用于秘密。

## 部署/副本集/守护集

首先，部署 10 个微型机器人的副本(CDK 的演示应用程序):

```
# Note we are still in the Magic Ring context…
kubectl create -f src/manifests/microbots-deployment.yaml 
deployment “microbot” created
```

让我们看看他们去了哪里:

```
# Querying the Federation control planed does not work
kubectl get pods -o wide
the server doesn't have a resource type "pods"# Querying AWS cluster directly
kubectl --context=aws get pods 
NAME                             READY     STATUS    RESTARTS   AGE
default-http-backend-wqrmm       1/1       Running   0          1d
microbot-1855935831-6n08n        1/1       Running   0          1m
microbot-1855935831-fvd7q        1/1       Running   0          1m
microbot-1855935831-gg5ql        1/1       Running   0          1m
microbot-1855935831-kltf0        1/1       Running   0          1m
microbot-1855935831-z7zp1        1/1       Running   0          1m# Now querying Azure directly
kubectl --context=azure get pods 
NAME                             READY     STATUS    RESTARTS   AGE
default-http-backend-04njk       1/1       Running   0          1h
microbot-1855935831-19m1p        1/1       Running   0          1m
microbot-1855935831-2gwjt        1/1       Running   0          1m
microbot-1855935831-8k3hc        1/1       Running   0          1m
microbot-1855935831-fgrn0        1/1       Running   0          1m
microbot-1855935831-ggvvf        1/1       Running   0          1m
```

联邦在云之间均匀地共享我们的微型机器人。这是预期的行为。如果我们有更多的集群，每一个都将得到公平份额的豆荚。

然而，值得注意的是，并不是所有的事情都奏效了，尽管在现阶段我还不清楚是哪里出了问题以及后果。日志显示:

```
E0210 10:35:53.691358 1 deploymentcontroller.go:516] Failed to ensure delete object from underlying clusters finalizer in deployment microbot: failed to add finalizer orphan to deployment : Operation cannot be fulfilled on deployments.extensions “microbot”: the object has been modified; please apply your changes to the latest version and try again
E0210 10:35:53.691566 1 deploymentcontroller.go:396] Error syncing cluster controller: failed to add finalizer orphan to deployment : Operation cannot be fulfilled on deployments.extensions “microbot”: the object has been modified; please apply your changes to the latest version and try again
```

您可以使用以下工具测试 DaemonSets:

```
kubectl --context=magicring create -f src/manifests/microbots-ds.yaml 
daemonset "microbot-ds" createdkubectl --context aws get po -n test-ns
NAME                READY     STATUS    RESTARTS   AGE
microbot-ds-5c25n   1/1       Running   0          48s
microbot-ds-cmvtj   1/1       Running   0          48s
microbot-ds-lp0j0   1/1       Running   0          48skubectl --context azure get po -n test-ns
NAME                READY     STATUS    RESTARTS   AGE
microbot-ds-bkj34   1/1       Running   0          53s
microbot-ds-r85z4   1/1       Running   0          53s
microbot-ds-w8kxg   1/1       Running   0          53s
```

因此，我们擅长应用程序分发。只有 2 更！

## 服务

现在让我们创建服务:

```
# Service creation...
kubectl --context=magicring create -f src/manifests/microbots-svc.yaml 
service "microbot" created
```

不要着急，因为这可能需要几分钟时间…

```
# On AWS
$ kubectl --context=aws get svc
NAME                   CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
default-http-backend   10.152.183.100   <none>        80/TCP         3d
kubernetes             10.152.183.1     <none>        443/TCP        3d
microbot               10.152.183.173   <none>        80/TCP         1m# On Azure
$ kubectl --context=azure get svc
NAME                   CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
default-http-backend   10.152.183.103   <none>        80/TCP    2d
kubernetes             10.152.183.1     <none>        443/TCP   2d
microbot               10.152.183.153   <none>        80/TCP    1m
```

很好！我们可以再次看到，服务已经如预期的那样在两个集群上复制。联邦正在各地共享服务。

更好的是，它还同步了我们在 Google Cloud DNS 中的 DNS 区域:

```
$ gcloud dns record-sets list  --zone demo-madeden
NAME TYPE TTL DATA
demo.madeden.net. NS 21600 ns-cloud-a1.googledomains.com.,ns-cloud-a2.googledomains.com.,ns-cloud-a3.googledomains.com.,ns-cloud-a4.googledomains.com.
demo.madeden.net. SOA 21600 ns-cloud-a1.googledomains.com. cloud-dns-hostmaster.google.com. 2 21600 3600 259200 300
microbot.default.magicring.svc.eu-west-2a.eu-west-2.demo.madeden.net. CNAME 180 microbot.default.magicring.svc.eu-west-2.demo.madeden.net.
microbot.default.magicring.svc.eu-west-2.demo.madeden.net. CNAME 180 microbot.default.magicring.svc.demo.madeden.net.
microbot.default.magicring.svc.us-west-2.demo.madeden.net. CNAME 180 microbot.default.magicring.svc.demo.madeden.net.
microbot.default.magicring.svc.us-west-2a.us-west-2.demo.madeden.net. CNAME 180 microbot.default.magicring.svc.us-west-2.demo.madeden.net.
```

因此，我们可以看到我们的区域配置正用于创建记录。如果我们之前没有配置这些，日志会丢弃一些错误，这些错误就不会被创建。

从<service>开始，DNS 结构的变化也不值一提。 <namespace>.svc.cluster.local 到<service>。<namespace>。<federation>. SVC<failure-zone></failure-zone></federation></namespace></service></namespace></service>

现在，microbot.default.magicring.svc.demo.madeden.net 透明地连接到两个集群，这意味着我们在集群之间拥有**全球 DNS 解析服务**。相当牛逼！

## 入口

不幸的是，这并不顺利…

```
# deploying Ingress...
kubectl --context=magicring create -f 
  src/manifests/microbots-ing.yaml 
ingress "microbot-ingress" created
```

检查结果:

```
# Querying ing on AWS
kubectl --context=aws get ing
NAME               HOSTS                        ADDRESS            PORTS     AGE
microbot-ingress   microbots.demo.madeden.net   10.0.1.95,10....   80        1d
# On AWS
kubectl --context=azure get ing
No resources found.
# Oups!! 
kubectl --context=magicring get ing
NAME               HOSTS                        ADDRESS            PORTS     AGE
microbot-ingress   microbots.demo.madeden.net   10.0.1.95,10....   80        1d
kubectl --context=magicring describe ing microbot-ingress
Name:     microbot-ingress
Namespace:    default
Address:    10.0.1.95,10.0.2.43,10.0.2.43
Default backend:  default-http-backend:80 (<none>)
Rules:
  Host        Path  Backends
  ----        ----  --------
  microbots.demo.madeden.net  
            /   microbot:80 (<none>)
Annotations:
  first-cluster:  aws
Events:
  FirstSeen LastSeen  Count From        SubObjectPath Type    Reason    Message
  --------- --------  ----- ----        ------------- --------  ------    -------
  1d    3m    2 {federated-ingress-controller }     Normal    CreateInCluster Creating ingress in cluster azure
  1d    1m    1 {federated-ingress-controller }     Normal    UpdateInCluster Updating ingress in cluster azure
  1d    1m    6 {federated-ingress-controller }     Normal    CreateInCluster Creating ingress in cluster aws
```

显然这里有一个问题。入口已经 ***而不是*** 被推送到所有集群。

当集群没有部署在 GCE/GKE 时，这是一个已知的错误(这是迄今为止唯一测试联合的环境)

你可以结账

*   [https://github.com/kubernetes/kubernetes/issues/33943](https://github.com/kubernetes/kubernetes/issues/33943)
*   https://github.com/kubernetes/kubernetes/issues/34291

了解更多细节。

如果您想从日志中截取这个错误，

```
# log from the Federation Controller Manager 
## And specific for the ingress creation
E0210 08:54:08.464928 1 ingress_controller.go:725] Failed to ensure delete object from underlying clusters finalizer in ingress microbot-ingress: failed to add finalizer orphan to ingress : Operation cannot be fulfilled on ingresses.extensions “microbot-ingress”: the object has been modified; please apply your changes to the latest version and try again
E0210 08:54:08.472338 1 ingress_controller.go:672] Failed to update annotation ingress.federation.kubernetes.io/first-cluster:aws on federated ingress “default/microbot-ingress”, will try again later: Operation cannot be fulfilled on ingresses.extensions “microbot-ingress”: the object has been modified; please apply your changes to the latest version and try again
```

如果您尝试删除入口，情况最终会变得更糟，此时它根本不会消失，您必须在每个集群上删除。

您仍然想要访问入口端点！下面是一个手动解决方法

```
for cloud in aws azure
do
  kubectl --context=${cloud} create -f src/manifests/microbots-ing.yaml 
done
```

然后，您可以直接在托管区域中公开该服务

```
# Identify the public addresses of the workers
juju switch aws
AWS_INSTANCES="$(juju show-status kubernetes-worker --format json | jq --raw-output '.applications.kubernetes-worker".units[]."public-address"' | tr '\n' ' ')"juju switch azure
AZURE_INSTANCES="$(juju show-status kubernetes-worker --format json | jq --raw-output '.applications."kubernetes-worker".units[]."public-address"' | tr '\n' ' ')"# Create the Zone File
touch /tmp/zone.list
for instance in ${AWS_INSTANCES} ${AZURE_INSTANCES}; 
do 
  echo "microbots.demo.madeden.net. IN A ${instance}" | tee -a /tmp/zone.list
done# Add a A record to the zone
gcloud dns record-sets import -z demo-madeden \
      --zone-file-format \
      /tmp/zone.list
```

几分钟后，当你将浏览器指向这个端点时，你会看到 Microbot 网页，均匀地显示来自 AWS 和 Azure 实例的 podnames。

因此，您可以创建世界范围的应用程序！！只是它没有像我们希望的那样自动化。

无论如何，这是 90%的工作，这对于一个如此年轻的技术来说是相当惊人的。向开发者致敬！

# 拆毁

在得出结论之前，我们先来分析一下集群。

拆除联邦既快又简单:您只需销毁名称空间！高效，即使正在努力使这个过程变得更好。

```
kubectl --context=magicring delete clusters \
  aws azure
kubectl --context=gke delete namespace \
  federation-system
```

摧毁 GKE 集群和 DNS 区域

```
gcloud dns managed-zones delete demo-madeden
gcloud container clusters delete gce --zone=us-east1-b
```

摧毁 2 个 Juju 控制器

```
# AWS
juju destroy-controller aws --destroy-all-models
WARNING! This command will destroy the "k8s-us-west-2" controller.
This includes all machines, applications, data and other resources.Continue? (y/N):y
Destroying controller
Waiting for hosted model resources to be reclaimed
Waiting on 1 model, 8 machines, 5 applications
Waiting on 1 model, 8 machines, 5 applications
Waiting on 1 model, 8 machines, 5 applications
Waiting on 1 model, 8 machines, 5 applications
Waiting on 1 model, 8 machines, 5 applications
Waiting on 1 model, 8 machines, 5 applications
Waiting on 1 model, 8 machines
Waiting on 1 model, 8 machines
Waiting on 1 model, 1 machine
Waiting on 1 model, 1 machine
Waiting on 1 model
Waiting on 1 model
All hosted models reclaimed, cleaning up controller machines # Azure
juju destroy-controller azure --destroy-all-models
WARNING! This command will destroy the "k8s-us-west-2" controller.
This includes all machines, applications, data and other resources.Continue? (y/N):y
Destroying controller
Waiting for hosted model resources to be reclaimed
Waiting on 1 model, 8 machines, 5 applications
Waiting on 1 model, 8 machines, 5 applications
Waiting on 1 model, 8 machines, 5 applications
Waiting on 1 model, 8 machines, 5 applications
Waiting on 1 model, 8 machines, 5 applications
Waiting on 1 model, 8 machines, 5 applications
Waiting on 1 model, 8 machines
Waiting on 1 model, 8 machines
Waiting on 1 model, 1 machine
Waiting on 1 model, 1 machine
Waiting on 1 model
Waiting on 1 model
All hosted models reclaimed, cleaning up controller machines
```

瞧吧！你已经完成了今天的库伯内特联盟实验。

# 结论

现在就在多云/全球范围的解决方案上与联盟合作是否值得？
肯定是的。Kubernetes 的这一部分是任何企业现在都在关注的*部分，了解其行为和架构绝对是一个优势。*

*作为一家公司，你应该已经为此做好生产准备了吗？自担风险。*

*Kubefed 部署的联邦控制平面(从今天起)可以工作，但是依赖于一个由 PV 支持的 etcd pod。不足以保证生产的稳定性和可靠性。因此，你需要工程来使它更坚固。由于 [Juju charms](https://jujucharms.com/etcd/23) 可以大规模部署，etcd 相对容易管理。但这仍然是一种努力。*

*此外，AWS 和 GKE/GCE 的一些关键功能(如入口分发)还没有准备好，这意味着除非你认真努力开发自己的解决方案，否则很有可能会受到影响。*

*最后，脱离 Google Cloud DNS 或 Route53，目前还没有可用的解决方案。快到了，但还没到。敬请期待*！*

*但是，您可以并且应该做的是通过使用 [CDK](https://jujucharms.com/canonical-kubernetes/) 在每一个云上部署来真正增加 k8s 的深度，并了解解决方案如何适应这些环境，每种云的优缺点以及机器类型。*

*作为 DevOps 社区中的个人，您应该与 K8s 和 Federation 合作吗？是啊！！！
首先，对于任何有发布应用程序经验的人来说，这真的是一个游戏改变者，而且部署起来并不困难，即使是中等规模。*

*那么下一步是什么？到目前为止，我们已经涵盖了*

*   *[为 k8s 构建 DYI GPU 集群](https://hackernoon.com/installing-a-diy-bare-metal-gpu-cluster-for-kubernetes-364200254187)*
*   *[将 k8s 与现有基础设施集成](/@samnco/automate-the-deployment-of-kubernetes-in-existing-aws-infrastructure-aa369df2f651)*
*   *联邦的现状(本文)*

*现在我需要更多的想法。另一个反复出现的问题是关于 k8s 中的身份管理，这可能是一个主题。*

*但是如果你有任何我应该做的酷的东西的想法…无论如何，在 GitHub (SaMnCo)或# kubernetes-users Slack(Samuel-me)上评论并找到我*