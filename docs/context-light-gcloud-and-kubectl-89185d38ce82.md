# 更安全的“gcloud”和“kubectl”

> 原文：<https://medium.com/google-cloud/context-light-gcloud-and-kubectl-89185d38ce82?source=collection_archive---------1----------------------->

## 如何保护自己不被自己伤害

好吧，这有它自己的故事！

我一直在寻找一个与`kubectl`的安全网，我一直在与`gcloud`一起使用。今天，我意识到它一直在那里，只是我没有意识到。

现在我可以:

```
kubectl get nodes
The connection to the server localhost:8080 was refused - did you specify the right host or port?kubectl get pods
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```

…这让我很开心！

让我支持一下…因为我认为这是一个不错的做法，我已经和`gcloud`一起使用很久了。

## gcloud

像`kubectl`这样的 Google 云平台的 CLI(非正式的叫`gcloud`)想通过持久化|假设上下文来节省时间。你会经常看到:

```
gcloud config set project ${PROJECT}
gcloud [[ create marvels]]
gcloud [[ delete marvels]]
gcloud project delete
```

一切都好。

问题是——如果像我一样——你会犯错误，有时你喜欢发号施令，而`gcloud [[ delete ]]`并没有指向你想要的东西。

那么你不快乐。

所以，我用了一个最小的`gcloud config`。这是斯堪的纳维亚式的配置文件设计:

```
gcloud config list
[core]
account = dazwilkin@google.com
```

不过这需要我非常明确地使用我的`gcloud`命令:

```
gcloud container clusters create ... --project=${PROJECT} ...
```

或许更重要的是，对坏人非常明确:

```
gcloud container clusters delete ... --project=${PROJECT} ...
```

是的，我仍然可以通过`--project=${PROJECT}`获得点击快感，但是，这种方法进一步支持的一件事是，我维护了一个本地会话状态，因此，只要我不被多个 shells 冲昏头脑，我通常能够知道我正在操作哪些资源:

```
PROJECT=[[TODAY's PROJECT]]
REGION=[[ALWAYS-US-WEST1-COS-I'M-WEIRD-LIKE-THAT]]
ZONE=${REGION}-c
```

感谢 Preston 第一次向我展示了这个技巧，你可以在例如`env.sh`中插入这种东西，然后，当你进入项目目录时，你可以不受惩罚地`source ./env.sh`。

所以，是的，更多的打字，但更多的控制和信心。

如果你有一个模糊的`gcloud config list`，你可以`gcloud config unset [section]/[property]`或者，如果你有信心，直接编辑文件。很可能`~/.config/gcloud/configurations/config_default`所以也许:

```
CONFIG="${HOME}/.config/gcloud/configurations/config_default"
cp ${CONFIG} ${CONFIG}.$(date '+%y%m%d') && \
nano ${CONFIG}
```

## 库贝特尔

直到你可以一般(！)使用`kubectl`获得类似的结果。

需要注意的是，当您创建和删除集群时，`gcloud container clusters`会改变`kubectl`配置。这将重置您的默认集群。您可以通过以下方式取消设置默认集群:

```
kubectl config unset current-context
```

然后一切将打破良好；-)

```
kubectl get nodes
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```

这在功能上等同于取消设置项目和区域等。在`gcloud`中。现在，我们可以在每次使用`kubectl`时显式引用一个上下文(以及一个集群)。例如:

```
kubectl get nodes --context=[[YOUR-CONTEXT]]
```

但是如何知道哪些上下文可用呢？

```
kubectl config get-contextskubectl config get-contexts
CURRENT   NAME          CLUSTER        AUTHINFO    NAMESPACE
          black         black-180612   dazwilkin   
          white         white-180612   dazwilkin
```

> **NB** 您的上下文和集群名称——特别是如果它们是由 Kubernetes 引擎唯一生成的——将**不会**这么漂亮。

不要害怕，因为我们可以:

```
kubectl config rename-context white dev
kubectl config get-contextkubectl config get-contexts
CURRENT   NAME          CLUSTER        AUTHINFO    NAMESPACE
          black         black-180612   dazwilkin   
          dev           white-180612   dazwilkin
```

然后，当然，你可以:

```
kubectl delete namespace/next-game --context=dev
```

那更让人放心，不是吗？

是的，这需要更多的输入，但是—再一次—我们可以使用我们的 shell 环境来帮助我们:

```
CONTEXT=dev
kubectl get pods --context=${CONTEXT}
```

如果我能拯救一个人一次那种删除错误信息的可怕、沮丧的感觉，我会很开心！

## 更新 180614:重命名|删除

我一直迂腐地(通常只是完全删除，但通常只是)编辑`~/.kube/config`以保持整洁。有一种更简单的方法(使用 kubectl 自动完成更好，见下文)。

假设您有一堆 Kubernetes 引擎生成的独特但冗长的上下文:

```
kubectl config get-contexts
CURRENT   NAME                                     CLUSTER ...
          gke_[[PROJECT]]_[[REGION]]_[[NAME]]      ...
          gke_[[PROJECT]]_[[REGION]]_[[NAME]]      ...
          gke_[[PROJECT]]_[[REGION]]_[[NAME]]      ...
```

您可以简单地:

```
kubectl config rename-context gke_... [[ALIAS]]
```

而且，如果你已经启用了自动补全，你可以简单地在`ku`之后开始按 TAB 键获取`kubectl`，在`conf`之后获取`kubectl config`，在`gke_`之后开始枚举以`gke`为前缀的上下文。一旦选择了所需的上下文，就可以将其重命名为更有用的别名。

> **NB** Kubernetes 引擎对上下文、集群和 authinfo(通常是重复的)部分使用相同的名称。

类似地，如果枚举你的上下文，列出孤儿，你可以用`kubectl delete-context`整理它们。当然，注意这样做，不要删除重要的上下文。在许多情况下，您可以通过重新运行`gcloud container clusters get-credentials ...`来恢复 Kubernetes 引擎上下文。

## 启用自动完成

这太好了:

[https://kubernetes . io/docs/tasks/tools/install-kubectl/# enabling-shell-autocompletion](https://kubernetes.io/docs/tasks/tools/install-kubectl/#enabling-shell-autocompletion)