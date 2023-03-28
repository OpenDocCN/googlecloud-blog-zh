# 零宕机 GKE 集群和节点版本升级和规格更新

> 原文：<https://medium.com/google-cloud/zero-downtime-gke-cluster-node-version-upgrade-and-spec-update-dad917e25b53?source=collection_archive---------1----------------------->

集群版本升级不会影响底层服务，但是 Kubernetes api 组件可能会在几分钟内不可用。此时，我们不能创建新的部署，也不能对资源进行任何其他更新。

集群升级后，向集群添加新版本或规范的新节点:

```
gcloud container node-pools create **new-node** --cluster **mycluster** --machine-type n1-standard-2 --disk-size 25 --num-nodes 5
```

现在，耗尽旧节点上运行的资源，将它们调度到新创建的节点上。这里，节点标签是`cloud.google.com/gke-nodepool=default-pool`。通过描述池的任意节点得到标签:`kubectl describe node **nodename**` **。**

```
for node in $(kubectl get nodes -l **cloud.google.com/gke-nodepool=default-pool** -o=name); do
  kubectl drain --force --ignore-daemonsets --delete-local-data --grace-period=300 "$node";
done
```

封锁节点，以便旧节点池上不会安排新资源:

```
for node in $(kubectl get nodes -l **cloud.google.com/gke-nodepool=default-pool** -o=name); do kubectl cordon "$node"; done
```

现在，删除旧节点:

```
gcloud container node-pools delete [POOL_NAME] --cluster [CLUSTER_NAME]
```

通过类似的方式，可以实现节点磁盘大小调整、镜像更新( `— image-type`)的零停机升级。

如果您愿意进行自动升级，可以根据有利的时机指定维护窗口:

```
gcloud container clusters update gcloud [CLUSTER_NAME] --maintenance-window [HH:MM]
```