# 降低 Kubernetes 集群(GKE)的成本—应用的概念

> 原文：<https://medium.com/google-cloud/cutting-costs-on-kubernetes-cluster-gke-concepts-applied-cc340eba0bec?source=collection_archive---------0----------------------->

> 免责声明:-此博客补充了参考资料部分提到的博客，以进一步降低您的基础设施成本。将所有这些放在一起将**显著**降低您的云基础设施成本。

降低现有基础设施成本对任何组织来说都是一项成就。在这篇博客中，让我们在 K8s 集群上实现这一点，毕竟它是任何现代云 it 基础架构的主要部分。为了实现这一点，首先想到的是—可抢占的虚拟机。您可以用可抢占的虚拟机组成您的 Kubernetes 集群(GKE)或它的一部分。但是现在，您一定想知道什么是“可抢占的虚拟机”？

> **什么是可抢占虚拟机及其弊端？**

有大量博客向您解释可抢占的虚拟机。简而言之，可抢占的虚拟机是一个实例，您可以以比普通实例低得多的价格(高达 80%)创建和运行它。那么显而易见的问题是“为什么不在可抢占的虚拟机上运行一切”。这个问题的答案是，“可抢占的虚拟机也有它们的缺点”。我不会进入细节，但几个缺点包括-最大。生命周期为 24 小时，它们可以在任何时候被终止(您对此无能为力&这只是部分事实)，并且它们不在 SLA 的覆盖范围内。[点击此处](https://cloud.google.com/compute/docs/instances/preemptible)了解更多关于可抢占虚拟机的信息&其缺点。

**现在让我们进入正题——削减 K8s (GKE)集群的成本**

在这篇博客中，我将利用以下事情来降低成本:- **GCP** ， **GKE** (Kubernetes)，**可抢占的虚拟机**，**谷歌云功能**和 **K8s 特征验证准入 Webhook** (准入控制器)。

很明显，如果我想在上降低成本，我将会在(谷歌云平台)上工作。但是请记住，您也可以将我们在这里讨论的内容应用到其他云提供商及其托管版本的 K8s 上。

## 步骤 1 -构建您的 GKE 集群

在 GKE 上，您可以构建异构集群，这意味着您可以用普通的和**可抢占的虚拟机组成您的 GKE 集群。**一旦您选择了想要在 GKE 集群中运行的可抢占虚拟机的数量，请确保它们位于单独的节点池中。 [*点击此处*](https://cloud.google.com/kubernetes-engine/docs/how-to/creating-a-cluster) 了解如何构建 GKE 集群和节点池(使用特定类型的虚拟机，在我们的示例中，虚拟机类型是可抢占的虚拟机)。

当构建上面提到的 GKE 集群和节点池时，确保将 [***污点***](https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/) 添加到您的*可抢占虚拟机节点池*。让我们假设，您添加了如下内容:-

```
NO_SCHEDULE task=preemptive
```

在 GKE 上构建*可抢占的虚拟机节点池*时，需要记住三件事。
1)。选择**启用**选项— *可抢占节点*。
②。选择上的**选项— *自动缩放*。因为它对自动缩放是神圣的。
3)。点击**+添加污点**选项— *节点污点*。然后如上所述添加污点。**

***注意:-*** 我们现在已经克服了上面提到的可抢占虚拟机的前两个主要缺点😎。由于可抢占的虚拟机现在是 GKE 节点池的一部分，每次可抢占的虚拟机终止时，都会自动创建一个替代虚拟机。很酷不是吗？

## 步骤 2 -增加工作负荷的容忍度

在上一步中，您已经选择了 GKE 集群中的*可抢占虚拟机节点池*的大小，并将*污点*添加到该节点池中。确保添加对工作负载(部署、复制集、状态集、复制控制器等)的容差。，您认为它可以容忍在一个*可抢占的虚拟机节点池*上运行——该池仅由可抢占的虚拟机组成。例如，您可以决定像-Nginx 部署这样的工作负载在*可抢占的 VM 节点池*上运行，使用类似下面的工作负载的容差***spec . template . spec***(在本例中是- *部署*)。

```
tolerations: 
- effect: NoSchedule
  key: task
  operator: Equal        
  value: preemptive
```

将上述容忍度添加到您认为有资格在*可抢占虚拟机节点池*上运行的所有工作负载中。请记住，在*可抢占虚拟机节点池*上运行的工作负载越多，而不是在具有普通(标准)实例的节点池上运行的工作负载越多，您的成本削减就越好。说到这里，让我们进入下一步。

## 第 3 步-在您的 GKE 集群上强制削减成本

就像我说的，在*可抢占的虚拟机节点池*上运行的工作负载越多，成本削减就越好。现在，我们变得专横一点，强制削减成本怎么样😉。这个想法很简单——允许来自暂存命名空间的工作负载只在*可抢占的虚拟机节点池*上运行，并确保这些工作负载有*容忍度*在*可抢占的虚拟机节点池*上运行，方法是使用一个 web hook——在我们的例子中是一个*谷歌云功能*。

为此，让我们使用 K8s 特性- ValidatingAdmissionWebhook(准入控制器)。 [*点击此处*](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#validatingadmissionwebhook) 进入详情，了解验证准入网钩。但简而言之，验证准入 Webhook 是一个 K8s 资源，它在通过身份验证和授权之后，但在获准进入 K8s 集群之前接收资源请求。现在，创建如下所示的 ValidatingAdmissionWebhook 配置，根据一组规则拦截来自名为 *test-staging-namespace* 的临时命名空间的所有请求。当然，在您的 GKE(K8s)集群中会有许多临时名称空间，所以您可以在下面的配置中随意添加它们。

```
apiVersion: admissionregistration.k8s.io/v1beta1
kind: ValidatingWebhookConfiguration
metadata:
  name: deny-absence-of-tolerations
webhooks:
  - name: deny.absence.of.tolerations
    rules:
      - apiGroups: ["apps","extensions"]
        apiVersions: ["v1","v1beta1","v1beta2"]
        operations: [ "CREATE","UPDATE" ]
        #operations: [ "*" ]
        resources: ["deployments","statefulsets"]
    namespaceSelector:
      matchExpressions:
      - key: namespace
        operator: In
        values:
        - test-staging-namespace
    failurePolicy: Fail
    clientConfig:
      url: "https://url_of_the_google_cloud_function"
      caBundle: AddCAbundleValueHere
```

基本上，上面的`ValidatingWebhookConfiguration`拦截了来自`test-staging-namespace`的所有部署和状态集。中断后，webhook `"https://deny_absence_of_tolerations"`被调用。这个 webhook 只不过是我们将在下一步讨论的*Google cloud Function*——它将确保您的工作负载具有容忍度，从而隐式地强制工作负载从 staging namespaces 运行在*可抢占 VM 节点池*上。

## 第四步:-创建一个谷歌云功能作为你的网络钩子

现在，您已经创建了 ValidatingAdmissionWebhook 配置。我们现在需要创建一个 webhook，在我们的例子中，它不过是一个 *Google Cloud 函数*。创建一个 *Google Cloud Function* 非常简单，一旦你创建了它，记下它的 url 并在上面的 Validating Admission Webhook 配置中替换它。您的 webhook 应该如下所示，它检查您的工作负载是否存在容差，如果不存在则返回一个错误。

```
var admissionResponse = {
    allowed: false
  };

  var found = false;

  if (!object.spec.template.spec.tolerations) {

      console.log("Workload is not using tolerations");

      admissionResponse.status = {
        status: 'Failure',
          message: "On Staging/Testing please use tolerations",
        reason: " Workload ( ie.,deployment ) Requirement Failed",
        code: 402
      };

      found = true;

  };

  if (!found) {
    admissionResponse.allowed = true;
  }

  var admissionReview = {
    response: admissionResponse
  };

  res.setHeader('Content-Type', 'application/json');
  res.send(JSON.stringify(admissionReview));
  res.status(200).end();
```

你可以在这里 找到上面 [*的完整代码，以备不时之需。请注意，当您运行异构集群时，您不必担心 SLA，因为即使在*计算引擎*决定取走所有可抢占的虚拟机的情况下，您的工作负载仍将落在普通(标准)虚拟机节点池中。当然，您也应该为此启用自动缩放。*](https://github.com/ChikkannaSuhas/IngeniousTechnologiesAG/blob/master/checkForTolerations_googlecloudfunction.js)

## 结论

在这篇博客中，我提到了一些概念，为了有效地降低基础设施成本，您可以集体利用这些概念。这个博客是各种话题的一个很好的接触点，比如 GKE，可抢占的虚拟机，谷歌云功能和验证准入网络钩子。我建议不要满足于现状，深入挖掘所有这些概念，以便更好地理解和进一步优化基础架构成本以及其他方面。希望你喜欢读这篇文章，如果你喜欢，就为这篇文章鼓掌😊。任何建议都非常感谢！。

> ***参考文献***

[https://cloud . Google . com/blog/products/containers-kubernetes/cutting-costs-with-Google-kubernetes-engine-using-the-cluster-auto scaler-and-preemptable-VMs](https://cloud.google.com/blog/products/containers-kubernetes/cutting-costs-with-google-kubernetes-engine-using-the-cluster-autoscaler-and-preemptible-vms)

[https://it next . io/save-costs-in-your-kubernetes-cluster-with-5-open-source-projects-7f 53899 a 1429](https://itnext.io/save-costs-in-your-kubernetes-cluster-with-5-open-source-projects-7f53899a1429)

[https://www . replex . io/blog/7-things-you-can-do-today-to-reduce-AWS-kubernetes-costs](https://www.replex.io/blog/7-things-you-can-do-today-to-reduce-aws-kubernetes-costs)