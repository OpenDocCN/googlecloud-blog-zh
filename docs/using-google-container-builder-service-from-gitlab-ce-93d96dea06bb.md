# 使用 Gitlab CE 的 Google 容器构建器服务

> 原文：<https://medium.com/google-cloud/using-google-container-builder-service-from-gitlab-ce-93d96dea06bb?source=collection_archive---------1----------------------->

我是 Gitlab 的忠实粉丝，但说到容器注册，Google Cloud Container Builder 更加灵活、快速、经济，而且开销非常少。无论是大量的构建还是大量的拉取(采用容器编排平台，Kubernetes)，或者管理具有拉取权限的 docker 映像，使用 Google Cloud Platform Container Builder 都很容易。漏洞扫描是现成的。

## 为 Gitlab Runner 创建 Google 云服务帐户

从 IAM & admin 部分创建一个具有合适名称的新服务帐户，并下载密钥。然后将容器生成器的[角色](https://cloud.google.com/container-builder/docs/securing-builds/configure-access-control#roles)分配给服务帐户。现在，我们给云容器构建器编辑、
存储管理员、项目查看者角色。登录 gitlab 服务器并[激活服务账户](https://cloud.google.com/sdk/gcloud/reference/auth/activate-service-account) : `gcloud auth activate-service-account [ACCOUNT] --key-file=[KEY_FILE]`

## 创造。gitlab-ci.yml

```
stages:- buildbuild:stage: buildscript:- gcloud container builds submit . --config=cloudbuild.yaml --substitutions BRANCH_NAME=$CI_COMMIT_REF_NAME,_IMAGE_NAME=$IMAGE_NAMEonly:- branches
```

创建`$IMAGE_NAME`环境变量，这将是我们在提取 docker 图像时使用的名称。例如，`mycoolimage`是`gcr.io/myproject/mycoolimage`的`IMAGE_NAME`。

## 创建 cloudbuild.yaml

```
steps:- name: gcr.io/cloud-builders/dockerargs: ['build', '-t', 'gcr.io/$PROJECT_ID/${_IMAGE_NAME}:${BRANCH_NAME}', '.']images: ['gcr.io/$PROJECT_ID/${_IMAGE_NAME}']
```

## 注册跑步者

为了简单起见，我们在 gitlab 服务器上安装了 gcloud sdk，并使用项目的令牌创建了一个 shell runner。

现在，在任何分支上的每次推送时，它都会触发标记名为 branch 的容器构建。

## 额外收获:删除 GCR 上未标记的或过去的码头图片

当使用与先前相同的标签创建新的 docker 图像时，最后一个图像将被取消标签，而较新的图像将被分配标签。因此，当您有大量构建时，不可用的映像会堆积在云存储中。以下是移除它们的方法:

```
for reg in $(gcloud container images list --repository=gcr.io/[PROJECT_NAME]);do for digest in $(gcloud container images list-tags "${reg}" --filter='-tags:*'  --format='get(digest)' --limit=50); do gcloud container images --quiet delete "${reg}"@"${digest}";
```

Kubernetes cronjob 在一天结束时删除未标记的 google 容器图像:

```
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: gcloud-cron
  namespace: default
spec:
  concurrencyPolicy: Allow
  failedJobsHistoryLimit: 1
  jobTemplate:
    metadata:
      creationTimestamp: null
    spec:
      template:
        metadata:
          creationTimestamp: null
        spec:
          containers:
          - args:
            - /bin/bash
            - -c
            - for reg in $(gcloud container images list --repository=gcr.io/[PROJECT_NAME]);
              do for digest in $(gcloud container images list-tags "${reg}" --filter='-tags:*'  --format='get(digest)'
              --limit=50); do gcloud container images --quiet delete "${reg}"@"${digest}";
              done; done
            image: google/cloud-sdk
            imagePullPolicy: Always
            name: gcloud-cron
            resources: {}
            terminationMessagePath: /dev/termination-log
            terminationMessagePolicy: File
          dnsPolicy: ClusterFirst
          restartPolicy: OnFailure
          schedulerName: default-scheduler
          securityContext: {}
          terminationGracePeriodSeconds: 30
  schedule: 0 0 0 * *
  successfulJobsHistoryLimit: 3
  suspend: false
```