# 使用 GKE 自动缩放器将您的 kubernetes 集群缩放至(几乎)零

> 原文：<https://medium.com/google-cloud/scale-your-kubernetes-cluster-to-almost-zero-with-gke-autoscaler-9c78051cbf40?source=collection_archive---------0----------------------->

![](img/73e888011f59280d89126ae3297ab5ac.png)

# 介绍

虽然 kubernetes 最出名的是编排适应需求的永久工作负载(部署)(水平 pod 自动缩放),但它正越来越多地用于短暂的批处理过程。在某些情况下，这些过程需要昂贵或稀有的专用硬件。机器学习训练工作就是这种情况，通常需要使用 GPU 的大型实例类型。在这种情况下，您需要确保作业完成后，您的集群立即释放所有这些资源。

考虑一个具有以下要求的机器学习培训工作(真实故事):

*   该作业必须在 4 个节点上并行运行。
*   每个节点都是 n1-highmem-96 机器，8 个 NVIDIA Tesla V100 GPUs(共 32 个)。
*   完成培训工作大约需要 12 个小时。

按照目前的价格，在没有特别折扣的情况下，这种设置每小时要花费 100 多美元。我们想在工作结束后尽快释放这些节点，以节省一些资金。

# 集群自动缩放功能助您一臂之力

使用 GKE 时，最有趣的特性之一是集群自动缩放功能。如 [GKE 文档](https://cloud.google.com/kubernetes-engine/docs/concepts/cluster-autoscaler)中所述，集群自动缩放允许:

> 根据工作负载的需求，自动调整 GKE 集群节点池的大小。当需求较高时，cluster autoscaler 会向节点池中添加节点。当需求较低时，集群自动缩放会缩小到您指定的最小大小。这可以在您需要时提高工作负载的可用性，同时控制成本。

但是，集群自动缩放无法将整个集群完全缩小到零。群集中必须至少有一个节点始终可用，才能运行系统窗格。所以你至少需要保留一个节点。但这并不意味着你需要保持一个*昂贵的*节点空闲运行。

GKE 的另一个非常有趣的特性是节点池。节点池是群集中具有相同配置的一组节点。每个集群至少有一个*默认*节点池，但是您可以根据需要添加其他节点池。

因此，为了满足我们的 ML 培训需求，我们将创建一个包含两个节点池的集群:

1.  一个固定大小的缺省节点池，其中一个节点具有较小的实例大小(例如`g1-small`)。
2.  第二个节点池(我们称之为突发池)包含我们的 ML 训练作业所需的实例类型(`n1-highmem-96`机器，带有 8 个*NVIDIA Tesla V100*GPU)。我们将在该节点池上设置群集自动扩展，以允许最多 4 个节点，最少 0 个节点。

# 确保您的 pod 在突发节点池中运行

现在我们已经配置了一个 GKE 集群来满足我们的自动扩展需求，我们需要确保我们的 ML 训练工作负载在突发池上运行。我们需要确保以下几点:

1.  我们希望我们的 ML 训练作业在突发节点池上运行，因为这里将创建带有 GPU 的 *highmem* 实例。
2.  我们的 ML 训练作业被设计成在一个节点中获取所有可用的资源，并且期望在每个节点中运行一个单独的训练 pod。
3.  突发节点池必须专用于 ML 训练作业。我们不希望让任何其他工作负载在这些节点上运行，因为我们希望在工作完成后立即释放它们。

我们不需要任何特殊的 GKE 功能来满足这些要求。以下标准 kubernetes 特性将帮助我们实现工作负载分配要求:

1.  我们将在 pod 中使用一个[节点选择器](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#nodeselector)来确保它们在 bust 节点池中运行。为此，我们将向该池中的节点添加一个标签。
2.  我们将使用一个[反关联性](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#affinity-and-anti-affinity)规则来确保您的两个培训单元不能被安排在同一个节点上。
3.  我们将向我们的猝发池节点添加一个[污点](https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/)，以防止其他工作负载在猝发节点池中运行。我们需要给我们的 ML 训练舱增加适当的容错能力，让它们在这些节点上运行。

# 把所有的放在一起

在实践中，我们将从创建一个具有默认节点池的群集开始，该节点池仅包含一个小节点:

```
PROJECT_ID="apszaz-kube-playground"
GCP_ZONE="europe-west1-b"
GKE_CLUSTER_NAME="burstable-cluster"
GKE_BURST_POOL="burst-zone"

gcloud container clusters create **${**GKE_CLUSTER_NAME**}** **\**
       --machine-type=g1-small **\**
       --num-nodes=1 **\**
       --zone=**${**GCP_ZONE**}** **\**
       --project=**${**PROJECT_ID**}**
```

现在，我们将使用以下参数添加拆分节点池:

*   这是我们希望用于 ML 培训工作的实例类型。与默认节点相反，默认节点只包含一个类型为`g1-small`的实例。
*   `--accelerator=nvidia-tesla-v100,8`:我们希望每个节点有 8 个*NVIDIA TESLA V100*GPU。这些 GPU 并非在所有地区和区域都可用，因此我们需要找到一个具有足够容量的区域。
*   `--node-labels=gpu=tesla-v100`:我们向突发池中的节点添加一个标签，以允许使用[节点选择器](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#nodeselector)在我们的 ML 训练工作负载中选择它们。
*   `--node-taints=reserved-pool=true:NoSchedule`:我们向节点添加了一个[污点](https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/)，以防止任何其他工作负载意外地被调度到这个节点池中。

其余选项指的是自动缩放，不言自明。完整的命令如下所示:

```
gcloud container node-pools create **${**GKE_BURST_POOL**}** **\**
       --cluster=**${**GKE_CLUSTER_NAME**}** **\**
       --machine-type=n1-highmem-96 **\**
       --accelerator=nvidia-tesla-v100,8 **\**
       --node-labels=gpu=tesla-v100 **\**
       --node-taints=reserved-pool=true:NoSchedule  **\**
       --enable-autoscaling **\**
       --min-nodes=0 **\**
       --max-nodes=4 **\**
       --zone=**${**GCP_ZONE**}** **\**
       --project=**${**PROJECT_ID**}**
```

为了测试配置，我们将创建一个作业，该作业将在 10 分钟内创建 4 个并行运行的 pod。我们的工作负载中的单元需要具有以下元素:

*   与我们添加到突发节点池的标签相匹配的一个`nodeSelector`:`gpu=tesla-v100`。
*   一个`podAntiAffinity`规则表明我们不希望两个带有相同标签`app=greedy-job`的 pod 在同一个节点上运行。为此，我们将为我们的 pod 添加适当的标签，并在`topologyKey`中指出它适用于`hostname`级别(同一节点中没有两个这样的 pod)。
*   最后，我们需要为附加到节点上的污点添加一个容差，这样就允许在这些节点上调度这些 podes。

完整的作业 YAML 文件(姑且称之为`greedy_job.yaml`)如下所示:

```
apiVersion: batch/v1
kind: Job
metadata:
  name: greedy-job
spec:
  parallelism: 4
  template:
    metadata:
      name: greedy-job
      **labels:** app: greedy-app
    spec:
      containers:
      - name: busybox
        image: busybox
        args:
        - sleep
        - "300"
      **nodeSelector:** gpu: tesla-v100
      **affinity:**
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - greedy-app
            topologyKey: "kubernetes.io/hostname"
      **tolerations:**
      - key: reserved-pool
        operator: Equal
        value: "true"
        effect: NoSchedule
      restartPolicy: OnFailure
```

# 验证它是否有效

首先，您需要获得集群凭证，以便能够在这个集群上运行 kubectl 命令:

```
gcloud container clusters get-credentials ${GKE_CLUSTER_NAME} \
       --zone=${GCP_ZONE} \
       --project=${PROJECT_ID}
```

我们看到，最初，GKE 启动群集时，突发池中有 3 个节点，默认池中有 1 个节点:

```
~ $ kubectl get nodes
NAME                                               STATUS   ROLES    AGE     VERSION
gke-burstable-cluster-**burst-zone**-183c2d4b-2vw9     Ready    <none>   9m7s    v1.13.11-gke.14
gke-burstable-cluster-**burst-zone**-183c2d4b-jkzt     Ready    <none>   9m10s   v1.13.11-gke.14
gke-burstable-cluster-**burst-zone**-183c2d4b-p2w8     Ready    <none>   9m7s    v1.13.11-gke.14
gke-burstable-cluster-**default-pool**-794fe9e9-jdk3   Ready    <none>   12m     v1.13.11-gke.14
```

让我们等待集群冷却下来，并删除突发池中的节点。几分钟后，我们看到所有突发池节点都已删除:

```
NAME                                               STATUS   ROLES    AGE   VERSION
gke-burstable-cluster-**default-pool**-794fe9e9-jdk3   Ready    <none>   24m   v1.13.11-gke.14
```

既然我们的集群已经处于待机模式(突发池中没有节点)，我们就可以开始运行测试了。我们将使用我们在上一节中定义的作业(我们称之为`greedy_job.yaml`)。该作业将运行四个并行运行的进程，并将在 10 分钟后完成。

最初，我们没有在默认名称空间中运行的 pods:

```
~ $ kubectl get pods
No resources found.
```

正如我们前面看到的，只有默认节点池中的虚拟节点:

```
~ $ kubectl get nodes
NAME                                               STATUS   ROLES    AGE   VERSION
gke-burstable-cluster-**default-pool**-794fe9e9-jdk3   Ready    <none>   26m   v1.13.11-gke.14
```

如果我们申请我们的工作:

```
~ $ kubectl apply -f greedy_job.yaml
job.batch/greedy-job created
```

我们看到 pod 已创建，但还需要等待一段时间:

```
~ $ kubectl get pod
NAME               READY   STATUS    RESTARTS   AGE
greedy-job-9wlb8   0/1     Pending   0          8s
greedy-job-hr2tc   0/1     Pending   0          8s
greedy-job-lqshk   0/1     Pending   0          8s
greedy-job-mcbmm   0/1     Pending   0          8s
```

如果查看其中一个窗格中的事件，您会看到它触发了一个集群纵向扩展事件:

```
~ $ kubectl describe pod greedy-job-9wlb8 
Name:           greedy-job-9wlb8
Namespace:      default
...
Events:
  Type     Reason            Age   From                Message
  ----     ------            ----  ----                -------
  Warning  FailedScheduling  26s   default-scheduler   0/1 nodes are available: 1 node(s) didn't match node selector.
  Normal   TriggeredScaleUp  20s   **cluster-autoscaler**  pod triggered **scale-up**: [{[https://content.googleapis.com/](https://content.googleapis.com/)... 0->1 (max: 4)}]
```

我们看到豆荚逐渐开始运行:

```
~ $ kubectl get pod -o wide
NAME               READY   STATUS              RESTARTS   AGE     IP          NODE                                             NOMINATED NODE   READINESS GATES
greedy-job-9wlb8   1/1     **Running**             0          2m47s   10.16.1.2   gke-burstable-cluster-burst-zone-183c2d4b-n1f3   <none>           <none>
greedy-job-hr2tc   1/1     **Running**             0          2m47s   10.16.2.2   gke-burstable-cluster-burst-zone-183c2d4b-sf5r   <none>           <none>
greedy-job-lqshk   0/1     Pending             0          2m47s   <none>      <none>                                           <none>           <none>
greedy-job-mcbmm   0/1     ContainerCreating   0          2m47s   <none>      gke-burstable-cluster-burst-zone-183c2d4b-jm49   <none>           <none>
```

几秒钟后再次检查:

```
~ $ kubectl get pod -o wide
NAME               READY   STATUS    RESTARTS   AGE     IP          NODE                                             NOMINATED NODE   READINESS GATES
greedy-job-9wlb8   1/1     **Running**   0          4m27s   10.16.1.2   gke-burstable-cluster-burst-zone-183c2d4b-n1f3   <none>           <none>
greedy-job-hr2tc   1/1     **Running**   0          4m27s   10.16.2.2   gke-burstable-cluster-burst-zone-183c2d4b-sf5r   <none>           <none>
greedy-job-lqshk   1/1     **Running**   0          4m27s   10.16.4.2   gke-burstable-cluster-burst-zone-183c2d4b-kbw2   <none>           <none>
greedy-job-mcbmm   1/1     **Running**   0          4m27s   10.16.3.2   gke-burstable-cluster-burst-zone-183c2d4b-jm49   <none>           <none>
```

一旦 10 分钟过去，吊舱就会终止:

```
~ $ kubectl get pod -o wide
NAME               READY   STATUS      RESTARTS   AGE     IP          NODE                                             NOMINATED NODE   READINESS GATES
greedy-job-9wlb8   0/1     Completed   0          7m58s   10.16.1.2   gke-burstable-cluster-burst-zone-183c2d4b-n1f3   <none>           <none>
greedy-job-hr2tc   0/1     Completed   0          7m58s   10.16.2.2   gke-burstable-cluster-burst-zone-183c2d4b-sf5r   <none>           <none>
greedy-job-lqshk   1/1     Running     0          7m58s   10.16.4.2   gke-burstable-cluster-burst-zone-183c2d4b-kbw2   <none>           <none>
greedy-job-mcbmm   0/1     Completed   0          7m58s   10.16.3.2   gke-burstable-cluster-burst-zone-183c2d4b-jm49   <none>           <none>
```

如果我们继续观察我们的节点，我们会看到，在第一批作业完成大约 10 分钟后，它们的状态变为`NotReady`(它们正在被耗尽)，最后消失。您可以使用此命令每 60 秒查看一次节点列表:

```
while true; do kubectl get nodes ; sleep 60; done
```

输出如下所示:

```
NAME                                               STATUS   ROLES    AGE   VERSION
gke-burstable-cluster-burst-zone-183c2d4b-jm49     Ready    <none>   14m   v1.13.11-gke.14
gke-burstable-cluster-burst-zone-183c2d4b-kbw2     Ready    <none>   13m   v1.13.11-gke.14
gke-burstable-cluster-burst-zone-183c2d4b-n1f3     Ready    <none>   16m   v1.13.11-gke.14
gke-burstable-cluster-burst-zone-183c2d4b-sf5r     Ready    <none>   15m   v1.13.11-gke.14
gke-burstable-cluster-default-pool-794fe9e9-jdk3   Ready    <none>   45m   v1.13.11-gke.14
NAME                                               STATUS     ROLES    AGE   VERSION
gke-burstable-cluster-burst-zone-183c2d4b-jm49     Ready      <none>   15m   v1.13.11-gke.14
gke-burstable-cluster-burst-zone-183c2d4b-kbw2     Ready      <none>   14m   v1.13.11-gke.14
gke-burstable-cluster-burst-zone-183c2d4b-n1f3     Ready      <none>   17m   v1.13.11-gke.14
gke-burstable-cluster-**burst-zone**-183c2d4b-sf5r     **NotReady**   <none>   16m   v1.13.11-gke.14
gke-burstable-cluster-default-pool-794fe9e9-jdk3   Ready      <none>   46m   v1.13.11-gke.14
NAME                                               STATUS     ROLES    AGE   VERSION
gke-burstable-cluster-**burst-zone**-183c2d4b-jm49     **NotReady**   <none>   16m   v1.13.11-gke.14
gke-burstable-cluster-burst-zone-183c2d4b-kbw2     Ready      <none>   15m   v1.13.11-gke.14
gke-burstable-cluster-burst-zone-183c2d4b-n1f3     Ready      <none>   18m   v1.13.11-gke.14
gke-burstable-cluster-**burst-zone**-183c2d4b-sf5r     **NotReady**   <none>   17m   v1.13.11-gke.14
gke-burstable-cluster-default-pool-794fe9e9-jdk3   Ready      <none>   47m   v1.13.11-gke.14
NAME                                               STATUS     ROLES    AGE   VERSION
gke-burstable-cluster-**burst-zone**-183c2d4b-kbw2     **NotReady**   <none>   16m   v1.13.11-gke.14
gke-burstable-cluster-burst-zone-183c2d4b-n1f3     Ready      <none>   19m   v1.13.11-gke.14
gke-burstable-cluster-default-pool-794fe9e9-jdk3   Ready      <none>   48m   v1.13.11-gke.14
NAME                                               STATUS     ROLES    AGE   VERSION
gke-burstable-cluster-**burst-zone**-183c2d4b-kbw2     **NotReady**   <none>   17m   v1.13.11-gke.14
gke-burstable-cluster-burst-zone-183c2d4b-n1f3     Ready      <none>   20m   v1.13.11-gke.14
gke-burstable-cluster-default-pool-794fe9e9-jdk3   Ready      <none>   49m   v1.13.11-gke.14
NAME                                               STATUS     ROLES    AGE   VERSION
gke-burstable-cluster-**burst-zone**-183c2d4b-n1f3     **NotReady**   <none>   21m   v1.13.11-gke.14
gke-burstable-cluster-default-pool-794fe9e9-jdk3   Ready      <none>   50m   v1.13.11-gke.14
NAME                                               STATUS     ROLES    AGE   VERSION
gke-burstable-cluster-**burst-zone**-183c2d4b-n1f3     **NotReady**   <none>   22m   v1.13.11-gke.14
gke-burstable-cluster-default-pool-794fe9e9-jdk3   Ready      <none>   51m   v1.13.11-gke.14
NAME                                               STATUS   ROLES    AGE   VERSION
gke-burstable-cluster-default-pool-794fe9e9-jdk3   Ready    <none>   52m   v1.13.11-gke.14
```

几分钟后，所有拆分节点都已删除，只剩下默认池中的`g1-small`节点。

# 结论

我们成功地创建了一个集群，它可以扩展到一个非常小(而且便宜)的节点，只需使用:

*   GKE 的[集群自动缩放](https://cloud.google.com/kubernetes-engine/docs/concepts/cluster-autoscaler)。
*   kubernetes 的特点是[节点选择器](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#nodeselector)、[反亲和规则](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#affinity-and-anti-affinity)和[污染和容忍](https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/)。

如果您不太关心实际的节点规格，也可以使用[节点自动配置](https://cloud.google.com/kubernetes-engine/docs/how-to/node-auto-provisioning)。通过节点自动配置，可以根据不可调度的 pod 的[规范](https://cloud.google.com/kubernetes-engine/docs/concepts/pod#pod-templates)自动创建和删除新的节点池。